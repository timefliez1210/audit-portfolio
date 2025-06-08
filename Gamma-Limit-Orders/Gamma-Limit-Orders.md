# Non Inclusion of actual target tick in calculated tick range causes spread on limit orders

## Summary  

During the calculation of the ```bottomTick``` and ```topTick``` in ```LimitOrderManager::createLimitOrder``` the actual ```targetTick``` is not included inside the range of ```[bottomTick; topTick]``` causing "spread" during the execution of such orders.
  
## Finding Description  

```TickLibrary::getValidTickRange```:
```javascript
function getValidTickRange(
	int24 currentTick,
	int24 targetTick,
	int24 tickSpacing,
	bool isToken0
) public pure returns (int24 bottomTick, int24 topTick) {

	/////////////////////////////////////////////////
	///////////////// SNIP //////////////////////////
	/////////////////////////////////////////////////

	if(isToken0) {
		if(roundedCurrentTick >= roundedTargetTick)
		revert RoundedTargetTickLessThanRoundedCurrentTick(currentTick, roundedCurrentTick, targetTick, roundedTargetTick);
1. @>	topTick = roundedTargetTick;
2. @>	bottomTick = topTick - tickSpacing;
	} else {
		if(roundedCurrentTick <= roundedTargetTick)
		revert RoundedTargetTickGreaterThanRoundedCurrentTick(currentTick, roundedCurrentTick, targetTick, roundedTargetTick);
		bottomTick = roundedTargetTick;
		topTick = bottomTick + tickSpacing;
	}
	/////////////////////////////////////////////////
	///////////////// SNIP //////////////////////////
	/////////////////////////////////////////////////
}
```

```TickLibrary::getRoundedTargetTick```:

```javascript
function getRoundedTargetTick(
	int24 targetTick,
	bool isToken0,
	int24 tickSpacing
) internal pure returns(int24 roundedTargetTick) {
	if (isToken0) {
		roundedTargetTick = targetTick >= 0 ?
3. @>	(targetTick / tickSpacing) * tickSpacing :
		((targetTick % tickSpacing == 0) ? targetTick : ((targetTick / tickSpacing) - 1) * tickSpacing);
	} else {
		roundedTargetTick = targetTick < 0 ?
		(targetTick / tickSpacing) * tickSpacing :
		((targetTick % tickSpacing == 0) ? targetTick : ((targetTick / tickSpacing) + 1) * tickSpacing);
	}
}
```

As you can see under 3. in the provided code above, we calculate the rounded target tick by dividing the ```targetTick``` with the ```tickSpacing``` which will produce the lower end of the tick boundary of the target tick.

However, considering now 1. and 2. in provided code above, this result is treated as the upper bound of the valid tick range, effectively excluding the user selected ```targetTick``` from the tick range the order will be executed in.

###### Consider the following scenario (provided in the PoC below):

A user wants to swap 20e18 tokens in currency 0 in a 1:1 initiated pool, tick spacing 60.
He sets a price limit of 1.2e18 per token0.

If the current tick is expected to be Tick 0, this would result in his target tick being 1799.

Following above quoted codes math:

```
1799 / 60 = 29 (in solidity)
29 * 60   = 1740 <= rounded target tick 
```

Now in 1. and 2. in above code we will now deduct the tick spacing to calculate the valid range, reulsting in a tick range of ```[1680;1740]``` excluding the users target price and target tick.

The Result is that the user will receive less amounts of token1 than expected and set in the limit order.

## Impact Explanation  

The spread showcased in the PoC below might not seem like a lot, however, since we are talking about a Limit Order system, not an AMM, it can not be neglected. Users of limit orders actually do so to avoid the spreads of market buy/sell orders, therefore it is crucial that orders will actually by executed on the ```topTick``` within the range, which does include the selected ```targetTick```.

A Medium Impact seems justified considering a relatively low spread.
  
## Likelihood Explanation  

The likelihood is High, since no preconditions need to be met and this issue occurs every time a limit order is created as long as ```targetTick % tickSpacing != 0```.
  
## Proof of Concept  

Please copy the following code into a file ```PoC.t.sol``` within the ```./test``` directory and execute:
```forge test --mt test_targetTickNotIncludedInRange -vv```:

```javascript
//SPDX-License-Identifier: MIT  

pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import {Deployers} from "@uniswap/v4-core/test/utils/Deployers.sol";
import {PoolSwapTest} from "v4-core/test/PoolSwapTest.sol";
import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";
import {PoolId, PoolIdLibrary} from "v4-core/types/PoolId.sol";
import {Currency, CurrencyLibrary} from "v4-core/types/Currency.sol";
import {StateLibrary} from "v4-core/libraries/StateLibrary.sol";
import {TickMath} from "v4-core/libraries/TickMath.sol";
import {LimitOrderHook} from "src/LimitOrderHook.sol";
import {LimitOrderManager} from "src/LimitOrderManager.sol";
import {Hooks} from "v4-core/libraries/Hooks.sol";
import {IERC20Minimal} from "v4-core/interfaces/external/IERC20Minimal.sol";
import {BalanceDelta} from "v4-core/types/BalanceDelta.sol";
import {LiquidityAmounts} from "@uniswap/v4-core/test/utils/LiquidityAmounts.sol";
import {ILimitOrderManager} from "src/ILimitOrderManager.sol";
import {LimitOrderLens} from "src/LimitOrderLens.sol";
import "../src/TickLibrary.sol";
import {ERC20Mock} from "@openzeppelin/contracts/mocks/token/ERC20Mock.sol"; 

contract PoC is Test, Deployers {
	
	using PoolIdLibrary for PoolKey;
	using CurrencyLibrary for Currency;
	uint256 public HOOK_FEE_PERCENTAGE = 50000;
	uint256 public constant FEE_DENOMINATOR = 100000;
	uint256 internal constant Q128 = 1 << 128;
	LimitOrderHook hook;
	ILimitOrderManager limitOrderManager;
	LimitOrderManager orderManager;
	LimitOrderLens lens;
	address public treasury;
	PoolKey poolKey;
	
	function setUp() public {
		deployFreshManagerAndRouters();
		(currency0, currency1) = deployMintAndApprove2Currencies();
		treasury = makeAddr("treasury");
		orderManager = new LimitOrderManager(
			address(manager), 
			treasury,
			address(this) 
		);
		uint160 flags = uint160(
			Hooks.BEFORE_SWAP_FLAG |
			Hooks.AFTER_SWAP_FLAG
		);
		address hookAddress = address(flags);
		deployCodeTo(
			"LimitOrderHook.sol",
			abi.encode(address(manager), address(orderManager), address(this)),
			hookAddress
		);
		hook = LimitOrderHook(hookAddress);
		limitOrderManager = ILimitOrderManager(address(orderManager));
		lens = new LimitOrderLens(
			address(orderManager),
			address(this) 
		);
		limitOrderManager.setExecutablePositionsLimit(5);
		limitOrderManager.setHook(address(hook));
		(poolKey,) = initPool(currency0, currency1, hook, 3000, SQRT_PRICE_1_1);
		orderManager.setWhitelistedPool(poolKey.toId(), true);
		IERC20Minimal(Currency.unwrap(currency0)).approve(address(limitOrderManager), type(uint256).max);
		IERC20Minimal(Currency.unwrap(currency1)).approve(address(limitOrderManager), type(uint256).max);
	modifyLiquidityRouter.modifyLiquidity(
		poolKey,
		IPoolManager.ModifyLiquidityParams({
			tickLower: -887220,
			tickUpper: 887220,
			liquidityDelta: 100 ether,
			salt: bytes32(0)
		}),
		""
		);
	}
	  
	function test_targetTickNotIncludedInRange() public {
		address user = makeAddr("user");
		ERC20Mock token0 = ERC20Mock(Currency.unwrap(currency0));
		ERC20Mock token1 = ERC20Mock(Currency.unwrap(currency1));
		token0.mint(user, 20e18);
		assertEq(20e18, token0.balanceOf(user));
		assertEq(0, token1.balanceOf(user));
		uint256 price = 1.2e18;
		uint256 amount = 20e18;
		uint256 roundedPrice = TickLibrary.getRoundedPrice(price, poolKey, true);
		int24 targetTick = TickMath.getTickAtSqrtPrice(TickLibrary.getSqrtPriceFromPrice(roundedPrice));
		vm.startPrank(user);
		token0.approve(address(limitOrderManager), 20e18);
		LimitOrderManager.CreateOrderResult memory result = limitOrderManager.createLimitOrder(true, targetTick, amount, poolKey);
		vm.stopPrank();
		console.log("==========================================================================================");
		console.log("Target Tick found corresponding to price: ", targetTick);
		console.log("Calculated Bottom Tick for Limit Order: ", result.bottomTick);
		console.log("Calculated Top Tick for Limit Order: ", result.topTick);
		console.log("==========================================================================================");
		
		swapRouter.swap(
		poolKey,
		IPoolManager.SwapParams({
		zeroForOne: false,
		amountSpecified: -40 ether,
		sqrtPriceLimitX96: TickMath.getSqrtPriceAtTick(result.topTick + 100)
		}),
		PoolSwapTest.TestSettings({
		takeClaims: false,
		settleUsingBurn: false
		}),
		""
		);
		LimitOrderManager.PositionInfo[] memory position = limitOrderManager.getUserPositions(user, poolKey.toId());
		vm.prank(user);
		limitOrderManager.claimOrder(poolKey, position[0].positionKey, user);
		console.log("==========================================================================================");
		console.log("Expected Amount out in currency 1 for set limit order: ", (amount * price) / 1e18);
		console.log("Actual Amount out received in currency 1 for set limit order: ", token1.balanceOf(user));
		console.log("Collected Treasury Fees for limit order execution: ", token1.balanceOf(treasury));
		console.log("Sanity Check to showcase its not the fees building the difference: ", 24e18 - token1.balanceOf(user) - token1.balanceOf(treasury));
		console.log("==========================================================================================");
	}
}
```

Executing above code will produce the following log:

```javascript
Ran 1 test for test/PoC.t.sol:PoC
[PASS] test_targetTickNotIncludedInRange() (gas: 830839)
Logs:
  ==========================================================================================
  Target Tick found corresponding to price:  1799
  Calculated Bottom Tick for Limit Order:  1680
  Calculated Top Tick for Limit Order:  1740
  ==========================================================================================
  ==========================================================================================
  Expected Amount out in currency 1 for set limit order:  24000000000000000000
  Actual Amount out received in currency 1 for set limit order:  23765313627633905976
  Collected Treasury Fees for limit order execution:  35701522725537307
  Sanity Check to showcase its not the fees building the difference:  198984849640556717
  ==========================================================================================

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.47ms (862.21µs CPU time)

Ran 1 test suite in 93.81ms (3.47ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Showcasing that indeed in this scenario a >1% spread (even after fee deduction) occured.

## Recommendation  

The issue could potentially be fixed in 2 places
a) ```TickLibrary::getValidTickRange```
b) ```TickLibrary::getRoundedTargetTick```

However, fixing it within ```getRoundedTargetTick``` could have implications on other parts within the protocol, therefore I will focus on ```getValidTickRange```:

```diff
function getValidTickRange(
	int24 currentTick,
	int24 targetTick,
	int24 tickSpacing,
	bool isToken0
) public pure returns (int24 bottomTick, int24 topTick) {
	if(isToken0 && currentTick >= targetTick)
	revert WrongTargetTick(currentTick, targetTick, true);
	if(!isToken0 && currentTick <= targetTick)
	revert WrongTargetTick(currentTick, targetTick, false);

	int24 roundedTargetTick = getRoundedTargetTick(targetTick, isToken0, tickSpacing);
	int24 roundedCurrentTick = getRoundedCurrentTick(currentTick, isToken0, tickSpacing);
	int24 tickDiff = roundedCurrentTick > roundedTargetTick ?
	roundedCurrentTick - roundedTargetTick :
roundedTargetTick - roundedCurrentTick;
	if(tickDiff < tickSpacing)
	revert RoundedTicksTooClose(currentTick, roundedCurrentTick, targetTick, roundedTargetTick, isToken0);
	if(isToken0) {
		if(roundedCurrentTick >= roundedTargetTick)
		revert RoundedTargetTickLessThanRoundedCurrentTick(currentTick, roundedCurrentTick, targetTick, roundedTargetTick);
-		topTick = roundedTargetTick;
-		bottomTick = topTick - tickSpacing;
+		bottomTick = roundedTargetTick;
+		topTick = roundedTargetTick + tickSpacing;
	} else {
		if(roundedCurrentTick <= roundedTargetTick)
		revert RoundedTargetTickGreaterThanRoundedCurrentTick(currentTick, roundedCurrentTick, targetTick, roundedTargetTick);
		bottomTick = roundedTargetTick;
		topTick = bottomTick + tickSpacing;
	}
	if(bottomTick >= topTick)
	revert SingleTickWrongTickRange(bottomTick, topTick, currentTick, targetTick, isToken0);
	if (bottomTick < minUsableTick(tickSpacing) || topTick > maxUsableTick(tickSpacing))
	revert TickOutOfBounds(targetTick);
}
```

This fix will ensure orders will be executed more towards the users expectations, reducing the spread and boosting protocol revenue.