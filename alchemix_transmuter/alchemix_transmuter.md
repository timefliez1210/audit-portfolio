# Alchemix Transmuter - Findings Report

## Table of Contents
- Medium Risk Findings
    - [M-01. _harvestAndReport returns incorrect value](#M-01)

---

## Contest Summary

**Sponsor:** Alchemix

**Dates:** Dec 16th, 2024 - Dec 23rd, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-12-alchemix)

---

## Results Summary

| Severity | Count |
|----------|-------|
| High     | 0     |
| Medium   | 1     |
| Low      | 0     |

---

# Medium Risk Findings

## <a id='M-01'></a>M-01. _harvestAndReport returns incorrect value

### Summary

According to the docs `StrategyOp::_harvestAndReport`, `StrategyArb::_harvestAndReport` and `StrategyMainnet::_harvestAndReport` are crucial for accounting purposes and therefore should return an accurate value of all funds currently held by the strategy. The issue arises since the sum in the return value simply forgets to add `uint256 claimable`.

### Finding Description

`StrategyOp::_harvestAndReport`, `StrategyArb::_harvestAndReport` and `StrategyMainnet::_harvestAndReport`:

```solidity
function _harvestAndReport()
    internal
    override
    returns (uint256 _totalAssets)
{
@>      uint256 claimable = transmuter.getClaimableBalance(address(this));        
    uint256 unexchanged = transmuter.getUnexchangedBalance(address(this));
    // NOTE : possible some dormant WETH that isn't swapped yet
    uint256 underlyingBalance = underlying.balanceOf(address(this));
@>      _totalAssets = unexchanged + asset.balanceOf(address(this)) + underlyingBalance;
}
```

As you can see on the highlighted lines of code, `claimable` correctly gets fetched, but is then forgotten in the return statement of `_totalAssets` potentially critically harming relying functions with an incorrect return value.

### Impact Explanation

According to the Documentation the `_harvestAndReport` function is considered as THE source of truth for accounting purposes (quote: "A trusted and accurate account for the total amount of 'asset' the strategy currently holds including idle funds."), therefore the impact of returning an incomplete value should be considered High by default.

### Likelihood Explanation

Likelihood: High

### Proof of Concept

N/A - Logic error is evident from code inspection.

### Recommendation

```diff
function _harvestAndReport()
    internal
    override
    returns (uint256 _totalAssets)
{
    uint256 claimable = transmuter.getClaimableBalance(address(this));        
    uint256 unexchanged = transmuter.getUnexchangedBalance(address(this));
    // NOTE : possible some dormant WETH that isn't swapped yet
    uint256 underlyingBalance = underlying.balanceOf(address(this));
-       _totalAssets = unexchanged + asset.balanceOf(address(this)) + underlyingBalance;
+       _totalAssets = unexchanged + asset.balanceOf(address(this)) + underlyingBalance + claimable;
}
```
