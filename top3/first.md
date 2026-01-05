# Solo Medium - Naturally occurring DoS of _repay() due to rounding in calcTrancheAtStartOfLiquidation()

## Summary

`calcTranchAtStartOfLiquidation` can produce `inputQ64 = 0` during repayments due to a rounding issue, preventing users to repay their debt, or being liquidated, if `netDebtXorYAssets` becomes way smaller than `activeLiquidityAssets`. This will happen during rapid market movements (consecutive buy or sell periods), or can happen when the pool constellation drastically changes.

## Finding Description

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

## Impact Explanation

DoS of the `AmmalgamPair._repay()` function can prevent users from repaying their debt, making them lose their collateral, it can freeze the collateral due to inability to repay, and affect liquidations also dependent on the `_repay()` logic. A high impact seems accurate.

## Likelihood Explanation

The pair configuration, seen in the PoC below, is nothing special and is basically using the protocol as it is intended. Therefore no pre-conditions, and not even malicious intend, is met. Likelihood is therefore High.

## Proof of Concept

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
        pair.sync();
        tokenX.transfer(address(pair), debtToRepay);

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

## Recommendation

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


## Additional PoC for Liquidation

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
