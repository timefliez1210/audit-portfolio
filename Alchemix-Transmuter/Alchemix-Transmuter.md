# Alchemix Transmuter - Findings Report

- ## Medium Risk Findings
    - ### [M-01. _harvestAndReport returns incorrect value](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: Alchemix

### Dates: Dec 16th, 2024 - Dec 23rd, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-12-alchemix)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 1
- Low: 0



    
# Medium Risk Findings


## <a id='M-01'></a>M-01. _harvestAndReport returns incorrect value            



## Description

According to the docs `StrategyOp::_harvestAndReport`, `StrategyArb::_harvestAndReport` and `StrategyMainnet::_harvestAndReport` are crucial for accounting purposes and therefor should return an accurate value of all funds currently held by the strategy.\
The issue arises since the sum in the return Value simply forgets to add `uint256 claimable`

## Vulnerable Code:

`StrategyOp::_harvestAndReport`, `StrategyArb::_harvestAndReport` and `StrategyMainnet::_harvestAndReport`:

```javascript
    function _harvestAndReport()
        internal
        override
        returns (uint256 _totalAssets)
    {
@>      uint256 claimable = transmuter.getClaimableBalance(address(this));        
        uint256 unexchanged = transmuter.getUnexchangedBalance(address(this));
        // NOTE : possible some dormant WETH that isn't swapped yet
        uint256 underlyingBalance = underlying.balanceOf(address(this));
@>      _totalAssets = unexchanged + asset.balanceOf(address(this)) + underlyingBalance;
    }
```

As you can see on the highlighted lines of code, `claimable` correctly gets fetched, but is than forgotten in the return statement of `_totalAssets` potentially critically harming relying functions with an incorrect return value.

## Impact:

According to the Documentation the `_harvestAndReport` function is considered as THE source of truth for accounting purposes (quote: "A trusted and accurate account for the total amount of 'asset' the strategy currently holds including idle funds."), therefore the impact of returning an incomplete value should be considered High by default.

Likelihood: High\
Impact: High

Severity: High

## Tools Used:

Manual review.

## Recommended Mitigation:

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
-       _totalAssets = unexchanged + asset.balanceOf(address(this)) + underlyingBalance;
+       _totalAssets = unexchanged + asset.balanceOf(address(this)) + underlyingBalance + claimable;
    }
```





