# Ammalgam - Findings Report

## Table of Contents
- High Risk Findings
    - [H-01. Naturally occurring DoS of _repay() due to rounding in calcTrancheAtStartOfLiquidation()](#H-01)
- Informational Findings
    - [I-01. Repayment of debt blocked incl. liquidations](#I-01)

---

## Contest Summary

**Sponsor:** Ammalgam

**Dates:** TBD

---

## Results Summary

| Severity      | Count |
|---------------|-------|
| High          | 1     |
| Medium        | 0     |
| Low           | 0     |
| Informational | 1     |

---

# High Risk Findings

## <a id='H-01'></a>H-01. Naturally occurring DoS of _repay() due to rounding in calcTrancheAtStartOfLiquidation()

### Summary

`calcTranchAtStartOfLiquidation` can produce `inputQ64 = 0` during repayments due to a rounding issue, preventing users to repay their debt, or being liquidated, if `netDebtXorYAssets` becomes way smaller than `activeLiquidityAssets`. This will happen during rapid market movements (consecutive buy or sell periods), or can happen when the pool constellation drastically changes.

### Finding Description

Consider the following function in `./library/Saturation.sol`:

```solidity
function calcTrancheAtStartOfLiquidation(
    uint256 netDebtXorYAssets,
    uint256 activeLiquidityAssets,
    uint256 trancheSpanInTicks,
    uint256 desiredThresholdMag2,
    bool netDebtX
) internal pure returns (int256 trancheStartOfLiquidationMag2) {
    int16 direction;
    uint256 inputQ64;

    ///// ***** SNIP ***** /////

        unchecked {
            inputQ64 =
                // top is uint128 * MAG2 * Q6.72 which fits in 211, we add 32 before dividing, then
                // add another 32 to make it a Q64 number. bottom is same. round down.
@>              Convert.mulDiv(
@>                  netDebtXorYAssets * SATURATION_TIME_BUFFER_IN_MAG2,
@>                  baseToPowerOfCount * Q32,
@>                  (baseToPowerOfCount - Q72) *
@>                      activeLiquidityAssets *
@>                      desiredThresholdMag2,
@>                  false
                ) *
                Q32;
        }

        direction = int16(netDebtX ? INT_ONE : INT_NEGATIVE_ONE);
    }

@>  int256 tick = TickMath.getTickAtPrice(inputQ64 ** 2);


	///// ***** SNIP ***** /////
```

During the call of `_repay()` within `AmmalgamPair.sol` the call chain will pass `calcTrancheAtStartOfLiquidation` as seen above. The described issue here arises for small remaining balances of `netDebtXorYAsset` which would result in `inputQ64` being assigned to 0, which will then be passed into `TickMath.getTickAtPrice()`, however this call will then ultimately revert with `PriceOutOfBounds()`, DoSing the operation of the `_repay()` function within `AmmalgamPair.sol`, which is being called when a user wants to repay his positions, or during liquidation. Either, locking the user collateral (if he would like to repay and sell it) or make him unliquidatable, which evidently then causes bad debt.

### Impact Explanation

DoS of the `AmmalgamPair._repay()` function can prevent users from repaying their debt, making them lose their collateral, it can freeze the collateral due to inability to repay, and affect liquidations also dependent on the `_repay()` logic. A high impact seems accurate.

### Likelihood Explanation

The pair configuration, seen in the PoC below, is nothing special and is basically using the protocol as it is intended. Therefore no pre-conditions, and not even malicious intend, is met. Likelihood is therefore High.

### Proof of Concept

Copy and paste the following Code into a new file `./test/PoC.t.sol` and execute `forge test --mt test_PoC_roundingWithinSaturationPreventsRepay`, `forge test --mt test_PoC_roundingPreventsRepaySelloffExtendedEdition` or/and `forge test --mt test_PoC_roundingWithinSaturationPreventsRepayTradingRefreshLimitedFeeWasteEdition`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {IERC20} from "@openzeppelin/contracts/interfaces/IERC20.sol";
import {StubERC20} from "./shared/StubErc20.sol";

import {AmmalgamPair} from "contracts/AmmalgamPair.sol";
import {AmmalgamFactory, IPairFactory, PairFactory} from "contracts/factories/AmmalgamFactory.sol";
import {PluginRegistry} from "contracts/tokens/PluginRegistry.sol";
import {IAmmalgamPair} from "contracts/interfaces/IAmmalgamPair.sol";
import {AmmalgamPair} from "contracts/AmmalgamPair.sol";
import {IAmmalgamERC20} from "contracts/interfaces/tokens/IAmmalgamERC20.sol";
import {DEPOSIT_X, DEPOSIT_Y, DEPOSIT_L, BORROW_X, BORROW_Y, BORROW_L} from "contracts/interfaces/tokens/ITokenController.sol";
import {IAmmalgamFactory} from "contracts/interfaces/factories/IAmmalgamFactory.sol";
import {ITokenFactory} from "contracts/interfaces/factories/ITokenFactory.sol";
import {INewTokensFactory} from "contracts/interfaces/factories/INewTokensFactory.sol";
import {deployFactory, deployTokenFactory} from "contracts/utils/deployHelper.sol";
import {SaturationAndGeometricTWAPState} from "contracts/SaturationAndGeometricTWAPState.sol";
import {FactoryPairTestFixture, MAX_TOKEN} from "test/shared/FactoryPairTestFixture.sol";
import {Validation} from "contracts/libraries/Validation.sol";
import {Liquidation} from "contracts/libraries/Liquidation.sol";
import {ICallback} from "contracts/interfaces/callbacks/IAmmalgamCallee.sol";

import {DEFAULT_MID_TERM_INTERVAL, MINIMUM_LONG_TERM_TIME_UPDATE_CONFIG} from "contracts/libraries/constants.sol";

import {getCreate2Address} from "test/shared/utilities.sol";

contract PoC is Test {
    IAmmalgamFactory factory;
    AmmalgamPair pair;
    IERC20 tokenX;
    IERC20 tokenY;

    address alice = address(0x1);
    address bob = address(0x2);
    address charlie = address(0x3);

    FactoryPairTestFixture fixture;

    function setUp() public {
        fixture = new FactoryPairTestFixture(
            MAX_TOKEN,
            MAX_TOKEN,
            false,
            false
        );
        factory = fixture.factory();
        pair = AmmalgamPair(address(fixture.pair()));

        fixture.transferTokensTo(alice, 4e18, 20000e18);
        fixture.transferTokensTo(bob, 20000e18, 20000e18);
        fixture.transferTokensTo(charlie, 1000e18, 100000e18);
        tokenX = fixture.tokenX();
        tokenY = fixture.tokenY();
    }

    function test_PoC_roundingWithinSaturationPreventsRepay() public {
        vm.startPrank(bob);
        tokenX.transfer(address(pair), 2e18);
        tokenY.transfer(address(pair), 20000e18);
        pair.mint(bob);
        vm.stopPrank();
        vm.startPrank(alice);
        tokenX.transfer(address(pair), 1e18);
        tokenY.transfer(address(pair), 10000e18);
        pair.mint(alice);
        pair.borrow(alice, 1.05e18, 0, "");
        vm.stopPrank();

        vm.warp(block.timestamp + 1);
        console.log("X Token Pool", tokenX.balanceOf(address(pair)));
        console.log("Y Token Pool", tokenY.balanceOf(address(pair)));
        vm.startPrank(alice);
        uint256 debtToRepay = pair.tokens(BORROW_X).balanceOf(alice);
        pair.sync(); // This can be commented out as well, I thought however it might fix the issue; Spoiler: it did not
        tokenX.transfer(address(pair), debtToRepay);

        // vm.expectRevert();
        pair.repay(alice);
        vm.stopPrank();
    }

    function test_PoC_roundingPreventsRepaySelloffExtendedEdition() public {
        vm.startPrank(bob);
        tokenX.transfer(address(pair), 2e18);
        tokenY.transfer(address(pair), 20000e18);
        pair.mint(bob);
        vm.stopPrank();
        vm.startPrank(alice);
        tokenX.transfer(address(pair), 1e18);
        tokenY.transfer(address(pair), 10000e18);
        pair.mint(alice);
        pair.borrow(alice, 1.05e18, 0, "");
        vm.stopPrank();

        vm.startPrank(charlie);
        for (uint i; i < 35; i++) {
            vm.warp(block.timestamp + 1);
            tokenX.transfer(address(pair), 0.07e18);
            pair.swap(0, 266e18, charlie, "");
        }
        vm.stopPrank();

        console.log("X Token Pool", tokenX.balanceOf(address(pair)));
        console.log("Y Token Pool", tokenY.balanceOf(address(pair)));
        vm.startPrank(alice);
        uint256 debtToRepay = pair.tokens(BORROW_X).balanceOf(alice);
        tokenX.transfer(address(pair), debtToRepay);

        // vm.expectRevert();
        pair.repay(alice);
        vm.stopPrank();
    }

    function test_PoC_roundingWithinSaturationPreventsRepayTradingRefreshLimitedFeeWasteEdition()
        public
    {
        vm.startPrank(bob);
        tokenX.transfer(address(pair), 2e18);
        tokenY.transfer(address(pair), 20000e18);
        pair.mint(bob);
        vm.stopPrank();
        vm.startPrank(alice);
        tokenX.transfer(address(pair), 1e18);
        tokenY.transfer(address(pair), 10000e18);
        pair.mint(alice);
        pair.borrow(alice, 1.05e18, 0, "");
        vm.stopPrank();

        vm.warp(block.timestamp + 30);
        vm.startPrank(charlie);
        tokenX.transfer(address(pair), 2e18);
        pair.swap(0, 8000e18, charlie, "");
        vm.stopPrank();
        console.log("X Token Pool", tokenX.balanceOf(address(pair)));
        console.log("Y Token Pool", tokenY.balanceOf(address(pair)));
        vm.startPrank(alice);
        uint256 debtToRepay = pair.tokens(BORROW_X).balanceOf(alice);
        tokenX.transfer(address(pair), debtToRepay);

        // vm.expectRevert();
        pair.repay(alice);
        vm.stopPrank();
    }
}
```

Running above tests will produce the following logs:

```bash
forge test --mt test_PoC_roundingWithinSaturationPreventsRepay
[⠊] Compiling...
[⠊] Compiling 1 files with Solc 0.8.28
[⠒] Solc 0.8.28 finished in 1.92s
Compiler run successful!

Ran 1 test for test/PoC.t.sol:PoC
[PASS] test_PoC_roundingWithinSaturationPreventsRepay() (gas: 3051966)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 11.32ms (3.22ms CPU time)

Ran 1 test suite in 11.87ms (11.32ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

```bash
forge test --mt test_PoC_roundingPreventsRepaySelloffExtendedEdition
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/PoC.t.sol:PoC
[FAIL: PriceOutOfBounds()] test_PoC_roundingPreventsRepaySelloffExtendedEdition() (gas: 7374463)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 41.89ms (29.13ms CPU time)

Ran 1 test suite in 42.49ms (41.89ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/PoC.t.sol:PoC
[FAIL: PriceOutOfBounds()] test_PoC_roundingPreventsRepaySelloffExtendedEdition() (gas: 7374463)

Encountered a total of 1 failing tests, 0 tests succeeded
```

```bash
forge test --mt test_PoC_roundingWithinSaturationPreventsRepayTradingRefreshLimitedFeeWasteEdition
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/PoC.t.sol:PoC
[FAIL: PriceOutOfBounds()] test_PoC_roundingWithinSaturationPreventsRepayTradingRefreshLimitedFeeWasteEdition() (gas: 3961721)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 33.89ms (14.24ms CPU time)

Ran 1 test suite in 34.64ms (33.89ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/PoC.t.sol:PoC
[FAIL: PriceOutOfBounds()] test_PoC_roundingWithinSaturationPreventsRepayTradingRefreshLimitedFeeWasteEdition() (gas: 3961721)

Encountered a total of 1 failing tests, 0 tests succeeded
```

Showcasing that expected reverts (even on an extended period, during a selloff period) is indeed happening.

### Recommendation

A possible solution for the rounding issue would be to simply pass in `true` however I did not investigate other implications:

```diff
unchecked {
            inputQ64 =
                Convert.mulDiv(
                    netDebtXorYAssets * SATURATION_TIME_BUFFER_IN_MAG2,
                    baseToPowerOfCount * Q32,
                    (baseToPowerOfCount - Q72) *
                        activeLiquidityAssets *
                        desiredThresholdMag2,
-                       false
+                       true
                ) *
                Q32;
        }
```

### Additional PoC for Liquidation

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {IERC20} from "@openzeppelin/contracts/interfaces/IERC20.sol";
import {StubERC20} from "./shared/StubErc20.sol";
import {AmmalgamPair} from "contracts/AmmalgamPair.sol";
import {AmmalgamFactory, IPairFactory, PairFactory} from "contracts/factories/AmmalgamFactory.sol";
import {PluginRegistry} from "contracts/tokens/PluginRegistry.sol";
import {IAmmalgamPair} from "contracts/interfaces/IAmmalgamPair.sol";
import {AmmalgamPair} from "contracts/AmmalgamPair.sol";
import {IAmmalgamERC20} from "contracts/interfaces/tokens/IAmmalgamERC20.sol";
import {DEPOSIT_X, DEPOSIT_Y, DEPOSIT_L, BORROW_X, BORROW_Y, BORROW_L} from "contracts/interfaces/tokens/ITokenController.sol";
import {IAmmalgamFactory} from "contracts/interfaces/factories/IAmmalgamFactory.sol";
import {ITokenFactory} from "contracts/interfaces/factories/ITokenFactory.sol";
import {INewTokensFactory} from "contracts/interfaces/factories/INewTokensFactory.sol";
import {deployFactory, deployTokenFactory} from "contracts/utils/deployHelper.sol";
import {SaturationAndGeometricTWAPState} from "contracts/SaturationAndGeometricTWAPState.sol";
import {FactoryPairTestFixture, MAX_TOKEN} from "test/shared/FactoryPairTestFixture.sol";
import {Validation} from "contracts/libraries/Validation.sol";
import {Liquidation} from "contracts/libraries/Liquidation.sol";
import {ICallback} from "contracts/interfaces/callbacks/IAmmalgamCallee.sol";

import {DEFAULT_MID_TERM_INTERVAL, MINIMUM_LONG_TERM_TIME_UPDATE_CONFIG} from "contracts/libraries/constants.sol";

import {getCreate2Address} from "test/shared/utilities.sol";

contract Liquidator is ICallback {
    FactoryPairTestFixture fixture;

    constructor(FactoryPairTestFixture _fixture) {
        fixture = _fixture;
    }

    function ammalgamSwapCallV1(
        address,
        uint256,
        uint256,
        bytes calldata
    ) external {}

    function ammalgamBorrowCallV1(
        address,
        uint256,
        uint256,
        uint256,
        uint256,
        bytes calldata
    ) external {}

    function ammalgamBorrowLiquidityCallV1(
        address,
        uint256,
        uint256,
        uint256,
        bytes calldata
    ) external {}

    function ammalgamLiquidateCallV1(
        uint256 repayXInXAssets,
        uint256 repayYInYAssets
    ) external {
        fixture.transferTokensTo(
            fixture.pairAddress(),
            repayXInXAssets,
            repayYInYAssets
        );
    }
}

contract PoC is Test {
    IAmmalgamFactory factory;
    AmmalgamPair pair;
    IERC20 tokenX;
    IERC20 tokenY;

    address alice = address(0x1);
    address bob = address(0x2);
    address charlie = address(0x3);

    FactoryPairTestFixture fixture;
    Liquidator liquidator;

    function setUp() public {
        fixture = new FactoryPairTestFixture(
            MAX_TOKEN,
            MAX_TOKEN,
            false,
            false
        );
        factory = fixture.factory();
        pair = AmmalgamPair(address(fixture.pair()));
        liquidator = new Liquidator(fixture);

        fixture.transferTokensTo(alice, 1e18, 10000e18);
        fixture.transferTokensTo(bob, 2e18, 20000e18);
        fixture.transferTokensTo(address(liquidator), 0, 0);
        tokenX = fixture.tokenX();
        tokenY = fixture.tokenY();
    }

    function test_PoC_roundingWithinSaturationPreventsLiquidation() public {
        vm.startPrank(bob);
        tokenX.transfer(address(pair), 2e18);
        tokenY.transfer(address(pair), 20000e18);
        pair.mint(bob);
        vm.stopPrank();
        vm.startPrank(alice);
        tokenX.transfer(address(pair), 1e18);
        tokenY.transfer(address(pair), 10000e18);
        pair.mint(alice);
        pair.borrow(alice, 1.05e18, 0, "");
        vm.stopPrank();

        vm.warp(block.timestamp + 1);

        vm.startPrank(address(liquidator));
        uint256 debtToRepay = pair.tokens(BORROW_X).balanceOf(alice);
        pair.liquidate(
            alice,
            address(liquidator),
            0,
            0,
            0,
            0,
            0,
            debtToRepay,
            0,
            Liquidation.HARD
        );
        vm.stopPrank();
    }
}
```

---

# Informational Findings

## <a id='I-01'></a>I-01. Repayment of debt blocked incl. liquidations

### Summary

The repayment of assets can fail due to outdated state within `_repay()` DoSing core functionality, accidentally or intentionally griefing legitimate repayments and/or preventing liquidations.

### Finding Description

During execution of the `_repay()` function, used during `repay()` and `liquidate()` "extra funds" directly send to the contract (PoC 2 below), or accidentally left within after a swap, or similar (PoC 1 below), will not be accounted for during `getNetBalances`, since `_reserveXAssets` and `_reserveYAssets` have not been accurately updated with potential donations, resulting in a `ERC20InsufficientBalance()` revert when `burnId()` is called. A malicious actor could DoS liquidations and repays for as little as `1 wei`.

#### Execution Flow

Consider the following code within `AmmalgamPair.sol`:

```solidity
function _repay(
        address onBehalfOf
    ) private returns (uint256 repayXInXAssets, uint256 repayYInYAssets) {
        uint256 _reserveXAssets;
        uint256 _reserveYAssets;
        (
            _reserveXAssets,
            _reserveYAssets,
            repayXInXAssets,
            repayYInYAssets
@>      ) = accrueSaturationPenaltiesAndInterest(onBehalfOf);

        (uint256 _missingXAssets, uint256 _missingYAssets) = missingAssets();
```

The call to `accrueSaturationPenaltiesAndInterest` is supposed to update the reserves and figure out the actual amount which has been repaid. However, looking now into `getNetBalances` downstream the call, we can see that `getNetBalances` uses manipulable `balanceOf()` accounting:

```solidity
function getNetBalances(
        uint256 _reserveXAssets,
        uint256 _reserveYAssets
    ) internal view returns (uint256, uint256) {
        (IERC20 _tokenX, IERC20 _tokenY) = underlyingTokens();

        return (
@>          _tokenX.balanceOf(address(this)) + rawTotalAssets(BORROW_X) - rawTotalAssets(DEPOSIT_X) - _reserveXAssets,
@>          _tokenY.balanceOf(address(this)) + rawTotalAssets(BORROW_Y) - rawTotalAssets(DEPOSIT_Y) - _reserveYAssets
        );
    }
```

The issue arising with this is that later in the call chain within the `repayHelper` function these potentially wrong values will be used to calculate the amount of shares to burn here:

```solidity
uint256 repayInShares = Convert.toShares(
            netRepayInAssets,
            rawTotalAssets(borrowTokenType),
            totalShares(borrowTokenType),
            !ROUNDING_UP
        );
```

The now calculated amount of `repayInShares` will then be passed into `burnId` causing a revert `ERC20InsufficientBalance` as soon as intentionally (by donating to the pool) or by accident any unaccounted balances remain. The probably most severe impact is that a malicious user could avoid/DoS complete liquidation of his position by simply donating 1 wei to the pool (as shown in the 2nd PoC). However, as PoC 1 shows, this can even happen by accident, when let's say a user swaps just a tiny amount more In that what he would get out.

### Impact Explanation

While this issue would be resolved by calling sync() on the pool, the fact, however remains, that it can happen by accident or intent DoSing core functionality using `_repay()` including `repay()` and `liquidate()` should definitively justify a High Impact.

### Likelihood Explanation

It's an unhandled error which can even happen during normal operations, and depleted asset scenarios are to be expected. Likelihood is High.

### Proof of Concept

#### Repayment Grief - Accidental setup

Copy and Paste the following code into a new file `./test/PoC.t.sol` and run `forge test --mt test_PoC_repaymentFailsDueToInsufficientDebtTokenBalance`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {IERC20} from "@openzeppelin/contracts/interfaces/IERC20.sol";
import {StubERC20} from "./shared/StubErc20.sol";

import {AmmalgamPair} from "contracts/AmmalgamPair.sol";
import {AmmalgamFactory, IPairFactory, PairFactory} from "contracts/factories/AmmalgamFactory.sol";
import {PluginRegistry} from "contracts/tokens/PluginRegistry.sol";
import {IAmmalgamPair} from "contracts/interfaces/IAmmalgamPair.sol";
import {AmmalgamPair} from "contracts/AmmalgamPair.sol";
import {IAmmalgamERC20} from "contracts/interfaces/tokens/IAmmalgamERC20.sol";
import {DEPOSIT_X, DEPOSIT_Y, DEPOSIT_L, BORROW_X, BORROW_Y, BORROW_L} from "contracts/interfaces/tokens/ITokenController.sol";
import {IAmmalgamFactory} from "contracts/interfaces/factories/IAmmalgamFactory.sol";
import {ITokenFactory} from "contracts/interfaces/factories/ITokenFactory.sol";
import {INewTokensFactory} from "contracts/interfaces/factories/INewTokensFactory.sol";
import {deployFactory, deployTokenFactory} from "contracts/utils/deployHelper.sol";
import {SaturationAndGeometricTWAPState} from "contracts/SaturationAndGeometricTWAPState.sol";
import {FactoryPairTestFixture, MAX_TOKEN} from "test/shared/FactoryPairTestFixture.sol";
import {Validation} from "contracts/libraries/Validation.sol";
import {Liquidation} from "contracts/libraries/Liquidation.sol";
import {ICallback} from "contracts/interfaces/callbacks/IAmmalgamCallee.sol";

import {DEFAULT_MID_TERM_INTERVAL, MINIMUM_LONG_TERM_TIME_UPDATE_CONFIG} from "contracts/libraries/constants.sol";

import {getCreate2Address} from "test/shared/utilities.sol";

contract PoC is Test {
    IAmmalgamFactory factory;
    AmmalgamPair pair;
    IERC20 tokenX;
    IERC20 tokenY;

    address alice = address(0x1);
    address bob = address(0x2);
    address charlie = address(0x3);

    FactoryPairTestFixture fixture;

    function setUp() public {
        fixture = new FactoryPairTestFixture(
            MAX_TOKEN,
            MAX_TOKEN,
            false,
            false
        );
        factory = fixture.factory();
        pair = AmmalgamPair(address(fixture.pair()));

        fixture.transferTokensTo(alice, 70e18, 10000e18);
        fixture.transferTokensTo(bob, 20000e18, 20000e18);
        fixture.transferTokensTo(charlie, 1000e18, 100000e18);
        tokenX = fixture.tokenX();
        tokenY = fixture.tokenY();
    }

    function test_PoC_repaymentFailsDueToInsufficientDebtTokenBalance() public {
        vm.startPrank(bob);
        tokenX.transfer(address(pair), 2e18);
        tokenY.transfer(address(pair), 20000e18);
        pair.mint(bob);
        vm.stopPrank();
        vm.startPrank(alice);
        tokenX.transfer(address(pair), 1e18);
        tokenY.transfer(address(pair), 10000e18);
        pair.mint(alice);
        pair.borrow(alice, 1.05e18, 0, "");

        tokenX.transfer(address(pair), 0.5e18);
        pair.swap(0, 4000e18, alice, "");
        tokenX.transfer(address(pair), 0.55e18);
        tokenY.transfer(address(pair), 4000e18);
        pair.mint(alice);
        pair.borrow(alice, 0.22e18, 0, "");
        tokenX.transfer(address(pair), 0.11e18);
        pair.swap(0, 600e18, alice, "");
        tokenX.transfer(address(pair), 0.11e18);
        tokenY.transfer(address(pair), 600e18);
        vm.stopPrank();

        vm.startPrank(alice);
        uint256 debtToRepay = pair.tokens(BORROW_X).balanceOf(alice);
        // pair.sync(); // if calling sync() before sending tokens and repaying, the repay will succeed
        tokenX.transfer(address(pair), debtToRepay);
        pair.repay(alice);
    }
}
```

which will produce the following terminal output:

```bash
forge test --mt test_PoC_repaymentFailsDueToInsufficientDebtTokenBalance
[⠊] Compiling...
[⠃] Compiling 1 files with Solc 0.8.28
[⠊] Solc 0.8.28 finished in 1.76s
Compiler run successful!

Ran 1 test for test/PoC.t.sol:PoC
[FAIL: ERC20InsufficientBalance(0x0000000000000000000000000000000000000001, 100000000000000000000 [1e20], 10000000000000000000000 [1e22])] test_PoC_repaymentFailsDueToInsufficientDebtTokenBalance() (gas: 1921026)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 7.94ms (826.08µs CPU time)

Ran 1 test suite in 8.56ms (7.94ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/PoC.t.sol:PoC
[FAIL: ERC20InsufficientBalance(0x0000000000000000000000000000000000000001, 100000000000000000000 [1e20], 10000000000000000000000 [1e22])] test_PoC_repaymentFailsDueToInsufficientDebtTokenBalance() (gas: 1921026)

Encountered a total of 1 failing tests, 0 tests succeeded
```

showcasing clearly the revert reason, the contract thinks the user should have a higher balance of debt tokens, than he/she actually has.

#### Liquidation DoS - Intentional donation

Within `./test/LiquidationTests.sol` replace the `testLiquidateHardBadDebtYAgainstLAndX` with (or alternatively just paste the highlighted transfer line in):

```diff
function testLiquidateHardBadDebtYAgainstLAndX() public {
        (
            ,
            Validation.InputParams memory borrowerUnderWater
        ) = createBadDebtWithYAgainstLAndX();

        // Reserves after
        (, uint256 reserveYInYAssets, ) = pair.getReserves();
        uint128[6] memory totalAssets = pair.totalAssets();

        uint256 repayBorrowedY = (Math.ceilDiv(
            (2 *
                borrowerUnderWater.userAssets[DEPOSIT_L] +
                Math.ceilDiv(
                    borrowerUnderWater.userAssets[DEPOSIT_X] * Q72,
                    borrowerUnderWater.sqrtPriceMinInQ72
                )) * Q72,
            borrowerUnderWater.sqrtPriceMinInQ72
        ) * 94) /
            // Double the deposit L because value wise 1 units of L is worth twice the corresponding amount of X & Y
            // ) * BIPS / maxPremiumInBips; // 11350
            100;

        assertLt(
            repayBorrowedY,
            borrowerUnderWater.userAssets[BORROW_Y],
            "Repay should be less than borrower debt"
        );

        // liquidate by repaying Y only
        vm.startPrank(liquidator);
+       tokenX.transfer(address(pair), 1);
        pair.liquidate(
            borrower,
            liquidator,
            borrowerUnderWater.userAssets[DEPOSIT_L],
            borrowerUnderWater.userAssets[DEPOSIT_X],
            0,
            0,
            0,
            0,
            repayBorrowedY,
            Liquidation.HARD
        );
        vm.stopPrank();

        // asserts
        ...
    }
```

executing above shows a similar output to the first scenario:

```bash
forge test --mt testLiquidateHardBadDebtYAgainstLAndX
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/LiquidationTests.sol:LiquidationTests
[FAIL: ERC20InsufficientBalance(0x2B5AD5c4795c026514f8317c7a215E218DcCD6cF, 0, 1)] testLiquidateHardBadDebtYAgainstLAndX() (gas: 1323720)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 117.49ms (4.54ms CPU time)
```

Clearly showcasing the liquidation DoS.

### Recommendation

Limit the amount passed into `burnId` by fetching the users actual balance of debt tokens and if any calculated debt would exceed said balance, use the maximum possible:

```diff
		///// ***** SNIP ***** /////

+		uint256 actualBorrowerShares = balanceOf(onBehalfOf, borrowTokenType);

        // Ensure we don't try to burn more shares than the borrower actually has
+       if (repayInShares > actualBorrowerShares) {
+           repayInShares = actualBorrowerShares;
+           netRepayInAssets = Convert.toAssets(
+               repayInShares,
+               rawTotalAssets(borrowTokenType),
+               totalShares(borrowTokenType),
+               ROUNDING_UP
+           );
+       }

        burnId(
            borrowTokenType,
            msg.sender,
            onBehalfOf,
            netRepayInAssets,
            repayInShares
        );
```
