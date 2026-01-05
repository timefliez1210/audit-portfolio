# Jigsaw - Findings Report

## Table of Contents
- High Risk Findings
    - [H-01. Users can unfairly be liquidated](#H-01)
    - [H-02. Permissionless Liquidators can profitably liquidate bad debt positions](#H-02)
    - [H-03. Partial Liquidation of liquidatable positions can worsen the position health](#H-03)
- Low Risk Findings
    - [L-01. Protocol will be stuck with Bad Debt](#L-01)

---

## Contest Summary

**Sponsor:** Jigsaw

**Dates:** TBD

---

## Results Summary

| Severity | Count |
|----------|-------|
| High     | 3     |
| Medium   | 0     |
| Low      | 1     |

---

# High Risk Findings

## <a id='H-01'></a>H-01. Users can unfairly be liquidated

### Summary

In the Jigsaw protocol users are encouraged to invest into strategies, while using their initial investment as collateral to borrow jUsd. However, failure to account for earnt yield within Jigsaws Strategies leads to unfair liquidations of actual solvent positions.

### Finding Description

Consider the following code in `StablesManager` and `LiquidationManager`:

```solidity
function isLiquidatable(address _token, address _holding) public view override returns (bool) {
	require(_holding != address(0), "3031");
	ISharesRegistry registry = _getRegistry(_token);
	require(address(registry) != address(0), "3008");
	if (registry.borrowed(_holding) == 0) return false;
	// Compute threshold for specified collateral
	ISharesRegistry.RegistryConfig memory registryConfig = registry.getConfig();
	uint256 threshold = registryConfig.collateralizationRate + registryConfig.liquidationBuffer;
	// Returns true when the ratio is below the liquidation threshold
	return getRatio({ _holding: _holding, registry: registry, rate: threshold })
	<= registry.borrowed(_holding).mulDiv(manager.getJUsdExchangeRate(), manager.EXCHANGE_RATE_PRECISION());
}
```

```solidity
function liquidate(
	address _user,
	address _collateral,
	uint256 _jUsdAmount,
	uint256 _minCollateralReceive,
	LiquidateCalldata calldata _data
)
external
override
nonReentrant
whenNotPaused
validAddress(_collateral)
validAmount(_jUsdAmount)
returns (uint256 collateralUsed)
{
	// Get protocol's required contracts to interact with.
	IHoldingManager holdingManager = _getHoldingManager();
	IStablesManager stablesManager = _getStablesManager();
	// Get address of the user's Holding involved in liquidation.
	address holding = holdingManager.userHolding(_user);
	// Get configs for collateral used for liquidation.
	(bool isRegistryActive, address registryAddress) = stablesManager.shareRegistryInfo(_collateral);
	// Perform sanity checks.
	require(isRegistryActive, "1200");
	require(holdingManager.isHolding(holding), "3002");
	require(_jUsdAmount <= ISharesRegistry(registryAddress).borrowed(holding), "2003");
	require(stablesManager.isLiquidatable({ _token: _collateral, _holding: holding }), "3073");
	// Calculate collateral required for the specified `_jUsdAmount`.
	collateralUsed = _getCollateralForJUsd({
		_collateral: _collateral,
		_jUsdAmount: _jUsdAmount,
		_exchangeRate: ISharesRegistry(registryAddress).getExchangeRate()
	});
	// Update the required collateral amount if there's liquidator bonus.
	collateralUsed += _user == msg.sender
		? 0
		: collateralUsed.mulDiv(
		ISharesRegistry(registryAddress).getConfig().liquidatorBonus, LIQUIDATION_PRECISION, Math.Rounding.Ceil
		);
	// If strategies are provided, retrieve collateral from strategies if needed.
	if (_data.strategies.length > 0) {
		_retrieveCollateral({
		_token: _collateral,
		_holding: holding,
		_amount: collateralUsed,
		_strategies: _data.strategies,
		_strategiesData: _data.strategiesData,
		useHoldingBalance: true
		});
	}
	// Check whether the holding actually has enough collateral to pay liquidator bonus.
	collateralUsed = Math.min(IERC20(_collateral).balanceOf(holding), collateralUsed);
	// Ensure the liquidator will receive at least as much collateral as expected when sending the tx.
	require(collateralUsed >= _minCollateralReceive, "3097");
	// Emit event indicating successful liquidation.
	emit Liquidated({ holding: holding, token: _collateral, amount: _jUsdAmount, collateralUsed: collateralUsed });
	// Repay user's debt with jUSD owned by the liquidator.
	stablesManager.repay({ _holding: holding, _token: _collateral, _amount: _jUsdAmount, _burnFrom: msg.sender });
	// Remove collateral from holding.
	stablesManager.forceRemoveCollateral({ _holding: holding, _token: _collateral, _amount: collateralUsed });
	// Send the liquidator the freed up collateral and bonus.
	IHolding(holding).transfer({ _token: _collateral, _to: msg.sender, _amount: collateralUsed });
}
```

Inspecting the functions above, it is plain as day that neither one of them is implementing any query regarding the rewards.

### Impact Explanation

The impact is to be considered high, since the users will lost a big amount of his collateral and any deposits into these strategies will be withdrawn, so he will immediately stop earning future yield, until manual interaction.

### Likelihood Explanation

Likelihood is high as well, the user in the scenario is using the protocol as it is intended, deposits assets, invests and loans. The failure to check accrued yield before liquidation is a critical oversight.

### Proof of Concept

Copy and paste the following code into `./test/PoC.t.sol` and run `forge test --mp PoC.t.sol`:

```solidity
//SPDX-License-Identifier: MIT

pragma solidity ^0.8.26;

import {BasicContractsFixture} from "./fixtures/BasicContractsFixture.t.sol";
import { ILiquidationManager } from "../src/interfaces/core/ILiquidationManager.sol";
import { ISharesRegistry } from "../src/interfaces/core/ISharesRegistry.sol";
import { IStrategyManager } from "../src/interfaces/core/IStrategyManager.sol";
import { IHoldingManager } from "../src/interfaces/core/IHoldingManager.sol";
import {console} from "forge-std/console.sol";
import "forge-std/Test.sol";
import { IERC20, IERC20Metadata } from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";
import {StrategyWithRewardsMock} from "./utils/mocks/StrategyWithRewardsMock.sol";

contract PoC is Test, BasicContractsFixture {
address user1 = payable(0x70997970C51812dc3A010C7d01b50e0d17dc79C8);
uint256 constant USER1_PK = 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d;
address user1Holding;
address liquidator = makeAddr("liquidator");
uint256 totalJusdSupply;
StrategyWithRewardsMock strategyRewards;

function setUp() public {
	init();
	vm.deal(user1, 50e18);
	deal(address(usdc), user1, 1000e18);

	vm.startPrank(OWNER, OWNER);
	jUsd.mint(OWNER, 500e18);
	jUsd.approve(address(liquidationManager), type(uint256).max);
	strategyRewards = new StrategyWithRewardsMock({
		_manager: address(manager),
		_tokenIn: address(usdc),
		_tokenOut: address(usdc),
		_rewardToken: address(usdc),
		_receiptTokenName: "RUsdc-Mock",
		_receiptTokenSymbol: "RUSDCM"
	});
	strategyManager.addStrategy(address(strategyRewards));
	vm.stopPrank();

	vm.startPrank(user1, user1);
	user1Holding = holdingManager.createHolding();
	usdc.approve(address(holdingManager), 1000e18);
	holdingManager.deposit(address(usdc), 1000e18);
	uint256 jusdAmountBorrowed = holdingManager.borrow(address(usdc), 500e18, 0, true);
	strategyManager.invest(
		address(usdc),
		address(strategyRewards),
		1000e18,
		1000e18,
		""
	);
	vm.stopPrank();
	usdcOracle.setPrice(8.5e17);
	assertTrue(stablesManager.isLiquidatable(address(usdc), user1Holding));
}

function test_PoC_unfairLiquidation() public {
	vm.startPrank(OWNER, OWNER);
	ILiquidationManager.LiquidateCalldata memory _data;
	address[] memory strategyAddresses = new address[](1);
	bytes[] memory strategyDataElements = new bytes[](1);
	strategyAddresses[0] = address(strategyRewards);
	strategyDataElements[0] = "";
	_data = ILiquidationManager.LiquidateCalldata(strategyAddresses, strategyDataElements);
	uint256 collaterealSeized = liquidationManager.liquidate(user1, address(usdc), 500e18, 0, _data); // Would revert too, if "normally" liquidated for the same reason
	assertGt(collaterealSeized, 0);
	vm.stopPrank();
}

function test_PoC_controllExample() public {
	vm.startPrank(user1, user1);
	strategyManager.claimRewards(address(strategyRewards), "");
	assertFalse(stablesManager.isLiquidatable(address(usdc), user1Holding));
	vm.stopPrank();
	vm.startPrank(OWNER, OWNER);
	ILiquidationManager.LiquidateCalldata memory _data;
	address[] memory strategyAddresses = new address[](1);
	bytes[] memory strategyDataElements = new bytes[](1);
	strategyAddresses[0] = address(strategyRewards);
	strategyDataElements[0] = "";
	_data = ILiquidationManager.LiquidateCalldata(strategyAddresses, strategyDataElements);
	vm.expectRevert();
	uint256 collateralSeized = liquidationManager.liquidate(user1, address(usdc), 500e18, 0, _data); // Would revert too, if "normally" liquidated for the same reason
	assertEq(collateralSeized, 0);
	vm.stopPrank();
}

}
```

Executing above test will proof that the users is undergoing successful liquidation in `test_PoC_unfairLiquidation`, even though he is not "insolvent" per se as seen in `test_PoC_controllExample`.

### Recommendation

Implement logic which either queries or withdraws the rewards before actual liquidation to ensure a user is really insolvent/liquidatable.

---

## <a id='H-02'></a>H-02. Permissionless Liquidators can profitably liquidate bad debt positions

### Summary

If any position during rapid market movement would enter a literal bad debt position it is possible for external liquidators to partially liquidate a position, worsening the collateralization ratio and leaving the protocol with even worse bad debt.

### Finding Description

During the liquidation call the `LiquidationManager` fails to check, if the position has entered a state of bad debt, therefore a permissionless, external liquidator can partially and most importantly profitably liquidate a position before the admin can call `liquidateBadDebt`. This scenario will significantly worsen the positions health factor and collateralization ratio, causing even more loss liquidating the remaining debt from the protocol side.

### Impact Explanation

Directly affecting the solvency of the protocol, syphoning value from bad debt positions is to be rated high.

### Likelihood Explanation

The only pre-conditions which have to be met are external. Internally the scenario exists that positions can enter bad debt state for several reasons. Therefore I rate the likelihood here high as well, since it is core functionality and it is expected that liquidators will act within there own interest.

### Proof of Concept

Create a file `./test/PoC.t.sol` copy and paste the following code into it and execute `forge test --mt test_pocProfitablyLiquidatingBadDebt -vv`:

```solidity
//SPDX-License-Identifier: MIT

pragma solidity ^0.8.26;

import {BasicContractsFixture} from "./fixtures/BasicContractsFixture.t.sol";
import { ILiquidationManager } from "../src/interfaces/core/ILiquidationManager.sol";
import { ISharesRegistry } from "../src/interfaces/core/ISharesRegistry.sol";
import { IStrategyManager } from "../src/interfaces/core/IStrategyManager.sol";
import { IHoldingManager } from "../src/interfaces/core/IHoldingManager.sol";
import {console} from "forge-std/console.sol";
import "forge-std/Test.sol";
import { IERC20, IERC20Metadata } from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";

contract PoC is Test, BasicContractsFixture {
	address user1 = payable(0x70997970C51812dc3A010C7d01b50e0d17dc79C8);
	uint256 constant USER1_PK = 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d;
	address user2 = payable(0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC);
	uint256 constant USER2_PK = 0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a;
	address liquidator = makeAddr("liquidator");
	uint256 totalJusdSupply;

	function setUp() public {
		init();
		vm.deal(user1, 50e18);
		deal(address(usdc), user1, 20e20);
		deal(address(weth), user1, 5e20);
		vm.deal(user2, 50e18);
		deal(address(usdc), user2, 10e20);
		deal(address(weth), user2, 5e20);
		wethOracle.setPrice(2500e18);
		deal(address(usdc), liquidator, 1e27);
		vm.startPrank(liquidator, liquidator);
		holdingManager.createHolding();
		usdc.approve(address(holdingManager), 1e27);
		holdingManager.deposit(address(usdc), 1e27);
		totalJusdSupply += holdingManager.borrow(address(usdc), 5e26, 5e26, true);
		jUsd.approve(address(liquidationManager), type(uint256).max);
		vm.stopPrank();
	}

	function test_pocProfitablyLiquidatingBadDebt() public {
		usdcOracle.setPrice(1e18);
		vm.startPrank(user1, user1);
		address user1Holding = holdingManager.createHolding();
		usdc.approve(address(holdingManager), 2000e18);
		holdingManager.deposit(address(usdc), 2000e18);
		uint256 jusdAmountBorrowed = holdingManager.borrow(address(usdc), 1000e18, 0, true); // always returns jusdMaxToBorrow
		vm.stopPrank();
		usdcOracle.setPrice(0.49e18);
		vm.startPrank(liquidator, liquidator);
		ILiquidationManager.LiquidateCalldata memory _data;
		address[] memory strategyAddresses = new address[](0);
		bytes[] memory strategyDataElements = new bytes[](0);
		_data = ILiquidationManager.LiquidateCalldata(strategyAddresses, strategyDataElements);
		uint256 collateralReceived = liquidationManager.liquidate(user1, address(usdc), 800e18, 0, _data);
		usdc.balanceOf(user1Holding);

		// Calculate the base collateral amount needed
		// 800 jUSD / 0.49 USD price = 1632.653... USDC
		uint256 baseCollateralAmount = (uint256(800e18) * uint256(1e18)) / uint256(49e16); // 800 / 0.49

		// Add liquidator bonus: 8000 / 100000 = 8%
		uint256 liquidatorBonus = (baseCollateralAmount * 8000) / 100000;
		uint256 expectedCollateralWithBonus = baseCollateralAmount + liquidatorBonus;
		assertApproxEqAbs(expectedCollateralWithBonus, collateralReceived, 10); // Allow up to 10 wei difference for rounding

		// Check position health before and after liquidation
		// Get current position state
		uint256 remainingCollateral = usdc.balanceOf(user1Holding);
		uint256 remainingDebt = 200e18; // 1000e18 - 800e18 = 200e18 debt remaining

		// Calculate collateral ratio before liquidation (at liquidation price 0.49)
		// Before: 2000 USDC * 0.49 = 980 USD collateral value vs 1000 jUSD debt
		uint256 collateralValueBefore = 2000e18 * 49e16 / 1e18; // 980 USD
		uint256 debtValueBefore = 1000e18; // 1000 jUSD
		uint256 healthRatioBefore = (collateralValueBefore * 1e18) / debtValueBefore; // Should be 0.98 (98%)
		// After liquidation: check remaining position
		uint256 collateralValueAfter = remainingCollateral * 49e16 / 1e18; // USD value
		uint256 debtValueAfter = remainingDebt; // 200 jUSD
		uint256 healthRatioAfter = (collateralValueAfter * 1e18) / debtValueAfter;
		console.log("=== LIQUIDATION ANALYSIS ===");
		console.log("Collateral removed (USDC):", collateralReceived / 1e18);
		console.log("Remaining collateral (USDC):", remainingCollateral / 1e18);
		console.log("Debt repaid (jUSD):", uint256(800));
		console.log("Remaining debt (jUSD):", remainingDebt / 1e18);
		console.log("");
		console.log("Health ratio before liquidation:", healthRatioBefore * 100 / 1e18, "%");
		console.log("Health ratio after liquidation:", healthRatioAfter * 100 / 1e18, "%");
		console.log("");
		console.log("Position improved?", healthRatioAfter > healthRatioBefore ? "YES" : "NO");
		// The key insight: did we extract more collateral value than debt we repaid?
		uint256 collateralValueExtracted = collateralReceived * 49e16 / 1e18; // USD value of extracted collateral
		uint256 debtRepaid = 800e18; // jUSD repaid
		console.log("Collateral value extracted (USD):", collateralValueExtracted / 1e18);
		console.log("Debt repaid (jUSD):", debtRepaid / 1e18);
		console.log("Profitable liquidation?", collateralValueExtracted > debtRepaid ? "YES" : "NO");
		vm.stopPrank();
	}
}
```

Execution of this testcase should produce a log showcasing the following:

```
Ran 1 test for test/PoC.t.sol:PoC
[PASS] test_pocProfitablyLiquidatingBadDebt() (gas: 557234)
Logs:
  === LIQUIDATION ANALYSIS ===
  Collateral removed (USDC): 1763
  Remaining collateral (USDC): 236
  Debt repaid (jUSD): 800
  Remaining debt (jUSD): 200

  Health ratio before liquidation: 98 %
  Health ratio after liquidation: 57 %

  Position improved? NO
  Collateral value extracted (USD): 864
  Debt repaid (jUSD): 800
  Profitable liquidation? YES

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.01ms (510.21µs CPU time)
```

Highlighting that the user indeed executed a profitable liquidation on the position while his profits are basically on the cost of the protocol and it's emergency fund.

### Recommendation

Implement a check in the liquidation logic to prevent partial liquidations of positions in bad debt as such:

```diff
function liquidate(
	address _user,
	address _collateral,
	uint256 _jUsdAmount,
	uint256 _minCollateralReceive,
	LiquidateCalldata calldata _data
)
external
override
nonReentrant
whenNotPaused
validAddress(_collateral)
validAmount(_jUsdAmount)
returns (uint256 collateralUsed)
{

	///// ***** SNIP ***** /////


	// Perform sanity checks.
	require(isRegistryActive, "1200");
	require(holdingManager.isHolding(holding), "3002");
	require(_jUsdAmount <= ISharesRegistry(registryAddress).borrowed(holding), "2003");
	require(stablesManager.isLiquidatable({ _token: _collateral, _holding: holding }), "3073");
+	uint256 totalCollateral = ISharesRegistry(registryAddress).collateral(holding);
+	if (
+	totalCollateral >= _getCollateralForJUsd({
+		_collateral: _collateral,
+		_jUsdAmount: _jUsdAmount,
+		_exchangeRate: ISharesRegistry(registryAddress).getExchangeRate()
+	})
+	) require(_jUsdAmount == ISharesRegistry(registryAddress).borrowed(holding));



	// Calculate collateral required for the specified `_jUsdAmount`.
	collateralUsed = _getCollateralForJUsd({
	_collateral: _collateral,
	_jUsdAmount: _jUsdAmount,
	_exchangeRate: ISharesRegistry(registryAddress).getExchangeRate()
	});

	///// ***** SNIP ***** /////
}
```

---

## <a id='H-03'></a>H-03. Partial Liquidation of liquidatable positions can worsen the position health

### Summary

Liquidations in Lending Protocols are meant to protect the protocol against bad debt and partial liquidations are supposed to protect the protocol from not liquidatable large positions, however it is crucial that the health factor increases during a partial liquidation, not decreases or even creates bad debt.

### Finding Description

The Jigsaw protocol lacks critical checks or "bad debt socialisation" to validate that a partial liquidation actually increased the health factor, therefore it is possible for partial liquidations to actually decrease the health factor and even create bad debt in certain scenarios, as shown in the PoC below.

### Impact Explanation

High, a decrease in health factor due to a partial liquidation disincentivizes following liquidations and can as shown in the attached PoC even create bad debt.

### Likelihood Explanation

High, partial liquidations are a healthy part of the ecosystem, and expected to occur regularly, especially on large "whale positions".

### Proof of Concept

Create a file `./test/PoC.t.sol` copy and paste the following code into it and execute `forge test --mt test_partialLiquidationWorsensHealthFactor -vvvvv`:

```solidity
//SPDX-License-Identifier: MIT

pragma solidity ^0.8.26;

import {BasicContractsFixture} from "./fixtures/BasicContractsFixture.t.sol";
import { ILiquidationManager } from "../src/interfaces/core/ILiquidationManager.sol";
import { ISharesRegistry } from "../src/interfaces/core/ISharesRegistry.sol";
import { IStrategyManager } from "../src/interfaces/core/IStrategyManager.sol";
import { IHoldingManager } from "../src/interfaces/core/IHoldingManager.sol";
import {console} from "forge-std/console.sol";
import "forge-std/Test.sol";
import { IERC20, IERC20Metadata } from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";

contract PoC is Test, BasicContractsFixture {
	address user1 = payable(0x70997970C51812dc3A010C7d01b50e0d17dc79C8);
	uint256 constant USER1_PK = 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d;
	address user2 = payable(0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC);
	uint256 constant USER2_PK = 0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a;
	address liquidator = makeAddr("liquidator");
	uint256 totalJusdSupply;

	function setUp() public {
		init();
		vm.deal(user1, 50e18);
		deal(address(usdc), user1, 20e20);
		deal(address(weth), user1, 5e20);
		vm.deal(user2, 50e18);
		deal(address(usdc), user2, 10e20);
		deal(address(weth), user2, 5e20);
		wethOracle.setPrice(2500e18);
		deal(address(usdc), liquidator, 1e27);
		vm.startPrank(liquidator, liquidator);
		holdingManager.createHolding();
		usdc.approve(address(holdingManager), 1e27);
		holdingManager.deposit(address(usdc), 1e27);
		totalJusdSupply += holdingManager.borrow(address(usdc), 5e26, 5e26, true);
		jUsd.approve(address(liquidationManager), type(uint256).max);
		vm.stopPrank();
	}

	function test_partialLiquidationWorsensHealthFactor() public {
		usdcOracle.setPrice(1e18);
		vm.startPrank(user1, user1);
		address user1Holding = holdingManager.createHolding();
		usdc.approve(address(holdingManager), 4000e18);
		holdingManager.deposit(address(usdc), 4000e18);
		uint256 jusdAmountBorrowed = holdingManager.borrow(address(usdc), 2000e18, 0, true);
		vm.stopPrank();
		// Price drops to make position liquidatable but close to bad debt
		// We need a price where partial liquidation + bonus extracts MORE value than debt repaid
		usdcOracle.setPrice(0.53e18);
		vm.startPrank(liquidator, liquidator);
		ILiquidationManager.LiquidateCalldata memory _data;
		address[] memory strategyAddresses = new address[](0);
		bytes[] memory strategyDataElements = new bytes[](0);
		_data = ILiquidationManager.LiquidateCalldata(strategyAddresses, strategyDataElements);
		// Calculate original health factor
		uint256 originalCollateralValue = 4000e18 * 53e16 / 1e18; // $510
		uint256 originalDebt = 2000e18;
		uint256 originalHealthFactor = (originalCollateralValue * 1e18) / originalDebt;
		console.log("=== BEFORE LIQUIDATION ===");
		console.log("Collateral value: $", originalCollateralValue / 1e18);
		console.log("Debt: $", originalDebt / 1e18);
		console.log("Health factor:", originalHealthFactor * 100 / 1e18, "%");
		// Partial liquidation of 300 jUSD (60% of debt)
		uint256 collateralReceived = liquidationManager.liquidate(user1, address(usdc), 1800e18, 0, _data);
		uint256 remainingCollateral = usdc.balanceOf(user1Holding);
		uint256 remainingDebt = 200e18;
		uint256 remainingCollateralValue = remainingCollateral * 53e16 / 1e18;
		uint256 newHealthFactor = (remainingCollateralValue * 1e18) / remainingDebt;
		// Assert the new collateralization ratio is below the original one
		assertLt(newHealthFactor, originalHealthFactor);
		vm.stopPrank();
	}
}
```

Execution of this testcase should produce a log showcasing the following:

```
@>  [0] VM::assertLt(879999999999999999 [8.799e17], 1060000000000000000 [1.06e18]) [staticcall]
    │   └─ ← [Return]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─  storage changes:
    │   @ 0xbcd530c794cc186a78002b1b01d50b5e4566a9ede0396037c8a97c0a0306e0ea: 0 → 0x0000000000000000000000000000000000000000000000d8d726b7177a800000
    │   @ 2: 2 → 1
    └─ ← [Stop]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 11.93ms (8.46ms CPU time)
```

As you can see above the collateralization ratio before the partial liquidation was above 1e18 which basically means it was recoverable without incurring loss, however, after this partial liquidation the ratio dropped below 1e18 which means, external liquidators have no benefit of liquidating such position, leaving the protocol with bad debt.

### Recommendation

Implement a check that liquidations definitively not make the collateralization ratio worse and increase the ratio in which a debt is considered bad by the percentage of liquidator bonus, since the issue is most severe within these percentiles.

---

# Low Risk Findings

## <a id='L-01'></a>L-01. Protocol will be stuck with Bad Debt

### Summary

In the unlikely event in which a Token Price would reach a literal 0, the protocol is unable to cover the position since `SharesRegistry::getExchangeRate` would revert with 2100, even when called during `LiquidationManager::liquidateBadDebt`.

### Finding Description

Consider the following code within `SharesRegistry`:

```solidity
function getExchangeRate() external view override returns (uint256) {
	(bool updated, uint256 rate) = oracle.peek(oracleData);
	require(updated, "3037");
@>	require(rate > 0, "2100");
	return rate;
}
```

While this line is a protection mechanism against "division by 0" errors and potential oracle failures, it also causes vital functions to revert, crucially, during times these functions have to be executable.

### Impact Explanation

The impact is High since any failure to liquidate bad debt directly causes protocol insolvency, jUsd will de-peg, and in the event a collateral token price is going to 0, the protocol would be stuck with the position.

### Likelihood Explanation

Regarding the likelihood I think it is vital to differentiate between internal logic and outside influences. Should a token price go to 0 for any reason the likelihood of this occurring is "always".
However, I would agree to a low likelihood after all, since this scenario would require a virtual black swan event, e.g. Circle going bankrupt, wBTC massively de-pegging etc.
It is however vital to realize that the protocol should not rely too heavily on assumptions outside it's influence, therefore the likelihood is not negligible.

### Proof of Concept

Create a file `./src/test/PoC.t.sol` copy and paste the attached code into it and run `forge test --mt test_pocBadDebtStuckInProtocol`. Since we are expecting the revert, the test is supposed to succeed.

```solidity
//SPDX-License-Identifier: MIT

pragma solidity ^0.8.26;

import {BasicContractsFixture} from "./fixtures/BasicContractsFixture.t.sol";
import { ILiquidationManager } from "../src/interfaces/core/ILiquidationManager.sol";
import { ISharesRegistry } from "../src/interfaces/core/ISharesRegistry.sol";
import { IStrategyManager } from "../src/interfaces/core/IStrategyManager.sol";
import { IHoldingManager } from "../src/interfaces/core/IHoldingManager.sol";
import {console} from "forge-std/console.sol";
import "forge-std/Test.sol";
import { IERC20, IERC20Metadata } from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";

contract PoC is Test, BasicContractsFixture {
	address user1 = payable(0x70997970C51812dc3A010C7d01b50e0d17dc79C8);
	uint256 constant USER1_PK = 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d;
	address user2 = payable(0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC);
	uint256 constant USER2_PK = 0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a;
	address liquidator = makeAddr("liquidator");
	uint256 totalJusdSupply;
	function setUp() public {
		init();
		usdcOracle.setPrice(1e18);
		vm.deal(user1, 50e18);
		deal(address(usdc), user1, 1000e18);
		vm.startPrank(OWNER, OWNER);
		jUsd.mint(OWNER, 200e18);
		jUsd.approve(address(liquidationManager), type(uint256).max);
		vm.stopPrank();
	}

	function test_pocBadDebtStuckInProtocol() public {
		vm.startPrank(user1, user1);
		address user1Holding = holdingManager.createHolding();
		usdc.approve(address(holdingManager), 1000e18);
		holdingManager.deposit(address(usdc), 1000e18);
		uint256 jusdAmountBorrowed = holdingManager.borrow(address(usdc), 500e18, 0, true);
		vm.stopPrank();

		usdcOracle.setPrice(0);

		vm.startPrank(OWNER, OWNER);
		ILiquidationManager.LiquidateCalldata memory _data;
		address[] memory strategyAddresses = new address[](0);
		bytes[] memory strategyDataElements = new bytes[](0);
		_data = ILiquidationManager.LiquidateCalldata(strategyAddresses, strategyDataElements);
		vm.expectRevert(); // 2100 Revert
		liquidationManager.liquidateBadDebt(user1, address(usdc), _data); // Would revert too, if "normally" liquidated for the same reason
		vm.stopPrank();
	}
}
```

### Recommendation

Remove the `require(price > 0)` statement in get exchange rate and reimplement it into the functions which rely on the price being >0 solely, basically doing a more conscious decision about this restriction.
