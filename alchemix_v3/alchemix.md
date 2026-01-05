# Alchemix V3 - Findings Report

## Table of Contents
- High Risk Findings
    - [H-01. Liquidation of users can cause absolute protocol insolvency, even towards healthy positions](#H-01)
- Medium Risk Findings
    - [M-01. Arithmetic Underflow in _subDebt causes liquidations to be unexecutable beyond a certain price drop](#M-01)

---

## Contest Summary

**Sponsor:** Alchemix

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

## <a id='H-01'></a>H-01. Liquidation of users can cause absolute protocol insolvency, even towards healthy positions

### Summary

During the liquidation of a user, should a de-peg of the underlying asset occur, the AlchemistV3::_liquidate function will attempt to transfer an amount of yield token to the transmuter to liquidate existing under collateralized positions, however the amount being transferred is not capped towards the deposited underlying yield token, potentially causing users being unable to claim their not liquidated positions, therefore making the AlchemistV3 insolvent.

### Finding Description

Consider the following code within AlchemistV3::_liquidate:

```solidity
function _liquidate(uint256 accountId) internal returns (uint256 debtAmount, uint256 feeInYield, uint256 feeInUnderlying) {

///// ***** SNIP ***** /////
collateralInUnderlying = totalValue(accountId);
collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

if (collateralizationRatio <= collateralizationLowerBound) {
	uint256 alchemistCurrentCollateralization = normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt;

	(uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee) = calculateLiquidation(
	collateralInUnderlying, account.debt, minimumCollateralization, alchemistCurrentCollateralization, globalMinimumCollateralization, liquidatorFee
	);

	uint256 feeBonus = debtToBurn * liquidatorFee / BPS;
	uint256 adjustedLiquidationAmount = convertDebtTokensToYield(liquidationAmount);
@>	uint256 adjustedDebtToBurn = convertDebtTokensToYield(debtToBurn);
	debtAmount = adjustedLiquidationAmount;
	feeInYield = convertDebtTokensToYield(baseFee);
	// update user balance (denominated in yield tokens)
	account.collateralBalance = account.collateralBalance > adjustedLiquidationAmount ? account.collateralBalance - adjustedLiquidationAmount : 0;

	// Update users debt (denominated in debt tokens)
	_subDebt(accountId, debtToBurn);
	// send liquidation amount - any fee to the transmuter. the transmuter only accepts yield tokens
@>	TokenUtils.safeTransfer(yieldToken, transmuter, adjustedDebtToBurn);

	if (feeInYield > 0) {
		// send base fee in yield tokens to liquidator
@>		TokenUtils.safeTransfer(yieldToken, msg.sender, feeInYield);
	}

	///// ***** SNIP ***** /////
}
```

and AlchemistV3::convertUnderlyingTokensToYield

```solidity
function convertUnderlyingTokensToYield(uint256 amount) public view returns (uint256) {
	uint8 decimals = TokenUtils.expectDecimals(yieldToken);
	return amount * 10 ** decimals / ITokenAdapter(tokenAdapter).price();
}
```

If the yield token de-pegs from it's original value the Alchemist will transfer out an unsanitized amount of yield tokens towards the transmuter, which can easily be higher than the originally deposited amount (the accounts collateral CDP). By doing so, the Alchemist actively pushes himself into insolvency regarding not liquidated, healthy positions. In short, by liquidating unhealthy positions the Alchemist becomes insolvent towards healthy positions, as shown in the PoC below.

### Impact Explanation

Described issue directly effects the solvency invariant of the protocol and furthermore shows that the liquidation logic, which is meant to protect protocol solvency, even makes it worse. Therefore the impact is high.

### Likelihood Explanation

Since the protocol expects de-pegs and no pre-condition have to be met the likelihood is high.

### Proof of Concept

Create a file ./src/test/PoC.t.sol, copy and paste below contents into it and execute `forge test --fork-url ${YOUR_RPC_URL} --mt test_PoC_insolvencyThroughLiquidation --fork-block-number 21835200`

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later

pragma solidity 0.8.26;

import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import {TransparentUpgradeableProxy} from "../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {SafeCast} from "../libraries/SafeCast.sol";
import {Test} from "../../lib/forge-std/src/Test.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {console} from "../../lib/forge-std/src/console.sol";
import {AlchemistV3} from "../AlchemistV3.sol";
import {AlchemicTokenV3} from "../test/mocks/AlchemicTokenV3.sol";
import {Transmuter} from "../Transmuter.sol";
import {AlchemistV3Position} from "../AlchemistV3Position.sol";
import {Whitelist} from "../utils/Whitelist.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {TestYieldToken} from "./mocks/TestYieldToken.sol";
import {TokenAdapterMock} from "./mocks/TokenAdapterMock.sol";
import {IAlchemistV3, IAlchemistV3Errors, AlchemistInitializationParams} from "../interfaces/IAlchemistV3.sol";
import {ITransmuter} from "../interfaces/ITransmuter.sol";
import {ITestYieldToken} from "../interfaces/test/ITestYieldToken.sol";
import {InsufficientAllowance} from "../base/Errors.sol";
import {Unauthorized, IllegalArgument, IllegalState, MissingInputData} from "../base/Errors.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";
import {IAlchemistV3Position} from "../interfaces/IAlchemistV3Position.sol";
import {AggregatorV3Interface} from "../../lib/chainlink-brownie-contracts/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";
import {TokenUtils} from "../libraries/TokenUtils.sol";
import {AlchemistTokenVault} from "../AlchemistTokenVault.sol";

contract PoC is Test {
	// Callable contract variables
	AlchemistV3 alchemist;
	Transmuter transmuter;
	AlchemistV3Position alchemistNFT;
	AlchemistTokenVault alchemistFeeVault;
	// // Proxy variables
	TransparentUpgradeableProxy proxyAlchemist;
	TransparentUpgradeableProxy proxyTransmuter;
	// // Contract variables
	// CheatCodes cheats = CheatCodes(HEVM_ADDRESS);
	AlchemistV3 alchemistLogic;
	Transmuter transmuterLogic;
	AlchemicTokenV3 alToken;
	Whitelist whitelist;
	// Token addresses
	TestERC20 fakeUnderlyingToken;
	TestYieldToken fakeYieldToken;
	// Total minted debt
	uint256 public minted;
	// Total debt burned
	uint256 public burned;
	// Total tokens sent to transmuter
	uint256 public sentToTransmuter;
	// Parameters for AlchemicTokenV2
	string public _name;
	string public _symbol;
	uint256 public _flashFee;
	address public alOwner;
	mapping(address => bool) users;
	uint256 public constant FIXED_POINT_SCALAR = 1e18;
	uint256 public liquidatorFeeBPS = 300; // in BPS, 3%
	uint256 public minimumCollateralization = uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 9e17;
	// ----- Variables for deposits & withdrawals -----
	// account funds to make deposits/test with
	uint256 accountFunds = 2_000_000_000e18;
	// large amount to test with
	uint256 whaleSupply = 20_000_000_000e18;
	// amount of yield/underlying token to deposit
	uint256 depositAmount = 100_000e18;
	// minimum amount of yield/underlying token to deposit
	uint256 minimumDeposit = 1000e18;
	// minimum amount of yield/underlying token to deposit
	uint256 minimumDepositOrWithdrawalLoss = FIXED_POINT_SCALAR;
	// random EOA for testing
	address externalUser = address(0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420);
	// another random EOA for testing
	address anotherExternalUser = address(0x420Ab24368E5bA8b727E9B8aB967073Ff9316969);
	// another random EOA for testing
	address yetAnotherExternalUser = address(0x520aB24368e5Ba8B727E9b8aB967073Ff9316961);
	// another random EOA for testing
	address someWhale = address(0x521aB24368E5Ba8b727e9b8AB967073fF9316961);
	// WETH address
	address public weth = address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);
	// Mock the price feed call
	address ETH_USD_PRICE_FEED_MAINNET = 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419;
	// Mock the price feed call
	uint256 ETH_USD_UPDATE_TIME_MAINNET = 3600 seconds;

	function setUp() external {
		// test maniplulation for convenience
		address caller = address(0xdead);
		address proxyOwner = address(this);
		vm.assume(caller != address(0));
		vm.assume(proxyOwner != address(0));
		vm.assume(caller != proxyOwner);
		vm.startPrank(caller);
		// Fake tokens
		fakeUnderlyingToken = new TestERC20(100e18, uint8(18));
		fakeYieldToken = new TestYieldToken(address(fakeUnderlyingToken));
		alToken = new AlchemicTokenV3(_name, _symbol, _flashFee);
		ITransmuter.TransmuterInitializationParams memory transParams = ITransmuter.TransmuterInitializationParams({
		syntheticToken: address(alToken),
		feeReceiver: address(this),
		timeToTransmute: 5_256_000,
		transmutationFee: 10,
		exitFee: 20,
		graphSize: 52_560_000
		});
		// Contracts and logic contracts
		alOwner = caller;
		transmuterLogic = new Transmuter(transParams);
		alchemistLogic = new AlchemistV3();
		whitelist = new Whitelist();
		// AlchemistV3 proxy
		AlchemistInitializationParams memory params = AlchemistInitializationParams({
		admin: alOwner,
		debtToken: address(alToken),
		underlyingToken: address(fakeUnderlyingToken),
		yieldToken: address(fakeYieldToken),
		blocksPerYear: 2_600_000,
		depositCap: type(uint256).max,
		minimumCollateralization: minimumCollateralization,
		collateralizationLowerBound: 1_052_631_578_950_000_000, // 1.05 collateralization
		globalMinimumCollateralization: 1_111_111_111_111_111_111, // 1.1
		tokenAdapter: address(fakeYieldToken),
		transmuter: address(transmuterLogic),
		protocolFee: 0,
		protocolFeeReceiver: address(10),
		liquidatorFee: liquidatorFeeBPS
		});
		bytes memory alchemParams = abi.encodeWithSelector(AlchemistV3.initialize.selector, params);
		proxyAlchemist = new TransparentUpgradeableProxy(address(alchemistLogic), proxyOwner, alchemParams);
		alchemist = AlchemistV3(address(proxyAlchemist));
		// Whitelist alchemist proxy for minting tokens
		alToken.setWhitelist(address(proxyAlchemist), true);
		whitelist.add(address(0xbeef));
		whitelist.add(externalUser);
		whitelist.add(anotherExternalUser);
		transmuterLogic.setAlchemist(address(alchemist));
		transmuterLogic.setDepositCap(uint256(type(int256).max));
		alchemistNFT = new AlchemistV3Position(address(alchemist));
		alchemist.setAlchemistPositionNFT(address(alchemistNFT));
		alchemistFeeVault = new AlchemistTokenVault(address(fakeUnderlyingToken), address(alchemist), alOwner);
		alchemistFeeVault.setAuthorization(address(alchemist), true);
		alchemist.setAlchemistFeeVault(address(alchemistFeeVault));
		vm.stopPrank();
		// Add funds to test accounts
		deal(address(fakeYieldToken), address(0xbeef), accountFunds);
		deal(address(fakeYieldToken), address(0xdad), accountFunds);
		deal(address(fakeYieldToken), externalUser, accountFunds);
		deal(address(fakeYieldToken), yetAnotherExternalUser, accountFunds);
		deal(address(fakeYieldToken), anotherExternalUser, accountFunds);
		deal(address(alToken), address(0xdad), 1000e18);
		deal(address(alToken), address(anotherExternalUser), accountFunds);
		deal(address(fakeUnderlyingToken), address(0xbeef), accountFunds);
		deal(address(fakeUnderlyingToken), externalUser, accountFunds);
		deal(address(fakeUnderlyingToken), yetAnotherExternalUser, accountFunds);
		deal(address(fakeUnderlyingToken), anotherExternalUser, accountFunds);
		deal(address(fakeUnderlyingToken), alchemist.alchemistFeeVault(), 10_000 ether);
		vm.startPrank(anotherExternalUser);
		SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), accountFunds);
		vm.stopPrank();
		vm.startPrank(yetAnotherExternalUser);
		SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), accountFunds);
		vm.stopPrank();
		vm.startPrank(someWhale);
		deal(address(fakeYieldToken), someWhale, whaleSupply);
		deal(address(fakeUnderlyingToken), someWhale, whaleSupply);
		SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), whaleSupply + 100e18);
		vm.stopPrank();
	}

	function test_PoC_insolvencyThroughLiquidation() public {
		// 0xBeef as the position to be liquidated
		// Deposit and Mint position
		vm.startPrank(address(0xbeef));
		fakeYieldToken.approve(address(alchemist), type(uint256).max);
		alchemist.deposit(1e18, address(0xbeef), 0);
		alchemist.mint(1, 9e17, address(0xbeef));
		vm.stopPrank();

		// External User as Secondary Depositor(victim)
		vm.startPrank(externalUser);
		fakeYieldToken.approve(address(alchemist), type(uint256).max);
		alchemist.deposit(1e18, externalUser, 0);
		vm.stopPrank();

		// Manipulate the Price to make 0xBeef under collateralized
		(uint256 cdpCollateralShares, , ) = alchemist.getCDP(1);
		uint256 newEffectiveSupply;
		deal(address(fakeUnderlyingToken), address(fakeYieldToken), cdpCollateralShares);
		newEffectiveSupply = cdpCollateralShares * 115/100;
		fakeYieldToken.updateMockTokenSupply(newEffectiveSupply);

		// Actually Liquidate 0xBeef NFT position
		vm.prank(anotherExternalUser);
		alchemist.liquidate(1);

		// Here External User will attempt the withdrawal of his position, however it reverts
		// with 0xe450d38c, ERC20CallFailed (Insufficient Balance)
		vm.prank(externalUser);
		vm.expectRevert();
		alchemist.withdraw(1e18, externalUser , 2);

		// Assert complete liquidation of 0xBeef => Collateral should be 0
		(uint256 collateralId1,,) = alchemist.getCDP(1);
		(uint256 collateralId2,,) = alchemist.getCDP(2);
		assertEq(collateralId1, 0);

		// This assert is directly in conflict with the protocol solvency invariant
		// Sum of all deposited collateral CDPs == alchemist.getTotalDeposited()
		assertGt(collateralId2, alchemist.getTotalDeposited());
		// assertEq(collateralId2, alchemist.getTotalDeposited()); // <= Actual Invariant, will obviously revert
	}
}
```

Running above test, will simply showcase the test succeeding. The asserts, vm.expectRevert() and comments should speak for themselves.

### Recommendation

Protect the solvency of the AlchemistV3 towards solvent positions by capping the amount transferred to the Transmuter to a maximum of user deposited assets of insolvent positions like such:

```diff
function _liquidate(uint256 accountId) internal returns (uint256 debtAmount, uint256 feeInYield, uint256 feeInUnderlying) {

	///// ***** SNIP ***** /////

+	if(adjustedDebtToBurn + feeInYield > account.collateralBalance) {
+		adjustedDebtToBurn = collateralInUnderlying - feeInYield;
+	}
	// update user balance (denominated in yield tokens)

	account.collateralBalance = account.collateralBalance > adjustedLiquidationAmount ? account.collateralBalance - adjustedLiquidationAmount : 0;
	// Update users debt (denominated in debt tokens)
	_subDebt(accountId, debtToBurn);
	// send liquidation amount - any fee to the transmuter. the transmuter only accepts yield tokens
	TokenUtils.safeTransfer(yieldToken, transmuter, adjustedDebtToBurn);

	///// ***** SNIP ***** /////

}
```

---

# Medium Risk Findings

## <a id='M-01'></a>M-01. Arithmetic Underflow in _subDebt causes liquidations to be unexecutable beyond a certain price drop

### Summary

An arithmetic underflow within the AlchemixV3::_subDebt function, used during liquidation, can prevent users from being liquidated, if the yield token de-pegs beyond a certain threshold.

### Finding Description

Consider the following code snippets:

```solidity
function _subDebt(uint256 tokenId, uint256 amount) internal {
	Account storage account = _accounts[tokenId];
	account.debt -= amount;
	totalDebt -= amount;
	// Update collateral variables
@>	uint256 toFree = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
	// For cases when someone above minimum LTV gets liquidated.
	if (toFree > _totalLocked) {
	toFree = _totalLocked;
	}
	_totalLocked -= toFree;
@>	account.rawLocked -= toFree;
	account.freeCollateral += toFree;
}
```

and:

```solidity
function convertYieldTokensToUnderlying(uint256 amount) public view returns (uint256) {
	uint8 decimals = TokenUtils.expectDecimals(yieldToken);
	// @audit note this is sus
@>	return (amount * ITokenAdapter(tokenAdapter).price()) / 10 ** decimals;
}
```

A sudden price drop (increased token supply) in yield token, can cause tooFree to overflow account.rawLocked, if a user holds multiple positions, resulting in liquidations failing during a period where they certainly should not fail.

### Impact Explanation

Failure to liquidate bad debt as generally high impact, since it will just continue to accumulate over time.

### Likelihood Explanation

The Likelihood however, is quite subjective, since it is a relatively sharp price drop and protocols Alchemix integrates with (presumably) are considered relatively safe. Therefore I would rate it is a medium.

### Proof of Concept

Create a file ./src/test/PoC.t.sol, copy and paste below contents into it and execute `forge test --fork-url ${YOUR_RPC_URL} --mt test_PoC_underflowInLiquidation --fork-block-number 21835200`

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later

pragma solidity 0.8.26;

import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import {TransparentUpgradeableProxy} from "../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {SafeCast} from "../libraries/SafeCast.sol";
import {Test} from "../../lib/forge-std/src/Test.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {console} from "../../lib/forge-std/src/console.sol";
import {AlchemistV3} from "../AlchemistV3.sol";
import {AlchemicTokenV3} from "../test/mocks/AlchemicTokenV3.sol";
import {Transmuter} from "../Transmuter.sol";
import {AlchemistV3Position} from "../AlchemistV3Position.sol";
import {Whitelist} from "../utils/Whitelist.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {TestYieldToken} from "./mocks/TestYieldToken.sol";
import {TokenAdapterMock} from "./mocks/TokenAdapterMock.sol";
import {IAlchemistV3, IAlchemistV3Errors, AlchemistInitializationParams} from "../interfaces/IAlchemistV3.sol";
import {ITransmuter} from "../interfaces/ITransmuter.sol";
import {ITestYieldToken} from "../interfaces/test/ITestYieldToken.sol";
import {InsufficientAllowance} from "../base/Errors.sol";
import {Unauthorized, IllegalArgument, IllegalState, MissingInputData} from "../base/Errors.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";
import {IAlchemistV3Position} from "../interfaces/IAlchemistV3Position.sol";
import {AggregatorV3Interface} from "../../lib/chainlink-brownie-contracts/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";
import {TokenUtils} from "../libraries/TokenUtils.sol";
import {AlchemistTokenVault} from "../AlchemistTokenVault.sol";

contract PoC is Test {
	// Callable contract variables
	AlchemistV3 alchemist;
	Transmuter transmuter;
	AlchemistV3Position alchemistNFT;
	AlchemistTokenVault alchemistFeeVault;
	// // Proxy variables
	TransparentUpgradeableProxy proxyAlchemist;
	TransparentUpgradeableProxy proxyTransmuter;
	// // Contract variables
	// CheatCodes cheats = CheatCodes(HEVM_ADDRESS);
	AlchemistV3 alchemistLogic;
	Transmuter transmuterLogic;
	AlchemicTokenV3 alToken;
	Whitelist whitelist;
	// Token addresses
	TestERC20 fakeUnderlyingToken;
	TestYieldToken fakeYieldToken;
	// Total minted debt
	uint256 public minted;
	// Total debt burned
	uint256 public burned;
	// Total tokens sent to transmuter
	uint256 public sentToTransmuter;
	// Parameters for AlchemicTokenV2
	string public _name;
	string public _symbol;
	uint256 public _flashFee;
	address public alOwner;
	mapping(address => bool) users;
	uint256 public constant FIXED_POINT_SCALAR = 1e18;
	uint256 public liquidatorFeeBPS = 300; // in BPS, 3%
	uint256 public minimumCollateralization = uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 9e17;
	// ----- Variables for deposits & withdrawals -----
	// account funds to make deposits/test with
	uint256 accountFunds = 2_000_000_000e18;
	// large amount to test with
	uint256 whaleSupply = 20_000_000_000e18;
	// amount of yield/underlying token to deposit
	uint256 depositAmount = 100_000e18;
	// minimum amount of yield/underlying token to deposit
	uint256 minimumDeposit = 1000e18;
	// minimum amount of yield/underlying token to deposit
	uint256 minimumDepositOrWithdrawalLoss = FIXED_POINT_SCALAR;
	// random EOA for testing
	address externalUser = address(0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420);
	// another random EOA for testing
	address anotherExternalUser = address(0x420Ab24368E5bA8b727E9B8aB967073Ff9316969);
	// another random EOA for testing
	address yetAnotherExternalUser = address(0x520aB24368e5Ba8B727E9b8aB967073Ff9316961);
	// another random EOA for testing
	address someWhale = address(0x521aB24368E5Ba8b727e9b8AB967073fF9316961);
	// WETH address
	address public weth = address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);
	// Mock the price feed call
	address ETH_USD_PRICE_FEED_MAINNET = 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419;
	// Mock the price feed call
	uint256 ETH_USD_UPDATE_TIME_MAINNET = 3600 seconds;

	function setUp() external {
		// test maniplulation for convenience
		address caller = address(0xdead);
		address proxyOwner = address(this);
		vm.assume(caller != address(0));
		vm.assume(proxyOwner != address(0));
		vm.assume(caller != proxyOwner);
		vm.startPrank(caller);
		// Fake tokens
		fakeUnderlyingToken = new TestERC20(100e18, uint8(18));
		fakeYieldToken = new TestYieldToken(address(fakeUnderlyingToken));
		alToken = new AlchemicTokenV3(_name, _symbol, _flashFee);
		ITransmuter.TransmuterInitializationParams memory transParams = ITransmuter.TransmuterInitializationParams({
		syntheticToken: address(alToken),
		feeReceiver: address(this),
		timeToTransmute: 5_256_000,
		transmutationFee: 10,
		exitFee: 20,
		graphSize: 52_560_000
		});
		// Contracts and logic contracts
		alOwner = caller;
		transmuterLogic = new Transmuter(transParams);
		alchemistLogic = new AlchemistV3();
		whitelist = new Whitelist();
		// AlchemistV3 proxy
		AlchemistInitializationParams memory params = AlchemistInitializationParams({
		admin: alOwner,
		debtToken: address(alToken),
		underlyingToken: address(fakeUnderlyingToken),
		yieldToken: address(fakeYieldToken),
		blocksPerYear: 2_600_000,
		depositCap: type(uint256).max,
		minimumCollateralization: minimumCollateralization,
		collateralizationLowerBound: 1_052_631_578_950_000_000, // 1.05 collateralization
		globalMinimumCollateralization: 1_111_111_111_111_111_111, // 1.1
		tokenAdapter: address(fakeYieldToken),
		transmuter: address(transmuterLogic),
		protocolFee: 0,
		protocolFeeReceiver: address(10),
		liquidatorFee: liquidatorFeeBPS
		});
		bytes memory alchemParams = abi.encodeWithSelector(AlchemistV3.initialize.selector, params);
		proxyAlchemist = new TransparentUpgradeableProxy(address(alchemistLogic), proxyOwner, alchemParams);
		alchemist = AlchemistV3(address(proxyAlchemist));
		// Whitelist alchemist proxy for minting tokens
		alToken.setWhitelist(address(proxyAlchemist), true);
		whitelist.add(address(0xbeef));
		whitelist.add(externalUser);
		whitelist.add(anotherExternalUser);
		transmuterLogic.setAlchemist(address(alchemist));
		transmuterLogic.setDepositCap(uint256(type(int256).max));
		alchemistNFT = new AlchemistV3Position(address(alchemist));
		alchemist.setAlchemistPositionNFT(address(alchemistNFT));
		alchemistFeeVault = new AlchemistTokenVault(address(fakeUnderlyingToken), address(alchemist), alOwner);
		alchemistFeeVault.setAuthorization(address(alchemist), true);
		alchemist.setAlchemistFeeVault(address(alchemistFeeVault));
		vm.stopPrank();
		// Add funds to test accounts
		deal(address(fakeYieldToken), address(0xbeef), accountFunds);
		deal(address(fakeYieldToken), address(0xdad), accountFunds);
		deal(address(fakeYieldToken), externalUser, accountFunds);
		deal(address(fakeYieldToken), yetAnotherExternalUser, accountFunds);
		deal(address(fakeYieldToken), anotherExternalUser, accountFunds);
		deal(address(alToken), address(0xdad), 1000e18);
		deal(address(alToken), address(anotherExternalUser), accountFunds);
		deal(address(fakeUnderlyingToken), address(0xbeef), accountFunds);
		deal(address(fakeUnderlyingToken), externalUser, accountFunds);
		deal(address(fakeUnderlyingToken), yetAnotherExternalUser, accountFunds);
		deal(address(fakeUnderlyingToken), anotherExternalUser, accountFunds);
		deal(address(fakeUnderlyingToken), alchemist.alchemistFeeVault(), 10_000 ether);
		vm.startPrank(anotherExternalUser);
		SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), accountFunds);
		vm.stopPrank();
		vm.startPrank(yetAnotherExternalUser);
		SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), accountFunds);
		vm.stopPrank();
		vm.startPrank(someWhale);
		deal(address(fakeYieldToken), someWhale, whaleSupply);
		deal(address(fakeUnderlyingToken), someWhale, whaleSupply);
		SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), whaleSupply + 100e18);
		vm.stopPrank();
	}

	function test_anotherLiquidationIssue() public {
		vm.startPrank(externalUser);
		fakeYieldToken.approve(address(alchemist), type(uint256).max);
		alchemist.deposit(5, externalUser, 0);
		alchemist.deposit(100e18, externalUser, 0);
		alchemist.mint(1, 2, externalUser);
		alchemist.mint(2, 90e18, externalUser);
		(uint256 cdpCollateralSharesOne, , ) = alchemist.getCDP(1);
		(uint256 cdpCollateralSharesTwo, , ) = alchemist.getCDP(2);
		uint256 cdpCollateralShares = cdpCollateralSharesOne + cdpCollateralSharesTwo;
		uint256 newEffectiveSupply;
		deal(address(fakeUnderlyingToken), address(fakeYieldToken), cdpCollateralShares);
		newEffectiveSupply = cdpCollateralShares * 180/100; // smaller numbers
		fakeYieldToken.updateMockTokenSupply(newEffectiveSupply);
		alchemist.liquidate(2);
	}
}
```

Running the test above as described will output:

```solidity
Ran 1 test for src/test/PoC.t.sol:PoC
[FAIL: panic: arithmetic underflow or overflow (0x11)] test_PoC_underflowInLiquidation() (gas: 1102723)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 12.01ms (1.44ms CPU time)

Ran 1 test suite in 1.28s (12.01ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in src/test/PoC.t.sol:PoC
[FAIL: panic: arithmetic underflow or overflow (0x11)] test_PoC_underflowInLiquidation() (gas: 1102723)

Encountered a total of 1 failing tests, 0 tests succeeded
```

Showcasing the above described underflow.

### Recommendation

Safeguard AlchemixV3::_subDebt against this case like:

```diff
function _subDebt(uint256 tokenId, uint256 amount) internal {
	Account storage account = _accounts[tokenId];
	account.debt -= amount;
	totalDebt -= amount;
	// Update collateral variables
	uint256 toFree = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
	// For cases when someone above minimum LTV gets liquidated.
	if (toFree > _totalLocked) {
	toFree = _totalLocked;
	}
+	if (toFree > account.rawLocked) {
+	toFree = account.rawLocked;
+	}
	_totalLocked -= toFree;
	account.rawLocked -= toFree;
	account.freeCollateral += toFree;
}
```
