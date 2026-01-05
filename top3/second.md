# High - getAutoDeleverageFactor in Market.sol can return 1 even if it should never

## Summary

`Market::getAutoDeleverageFactor` as described in `CreditDelegationBranch::withdrawUsdTokenFromMarket` should never return 1, but in fact it can return 1.

## Finding Description

`Market::getAutoDeleverageFactor` as described in `CreditDelegationBranch::withdrawUsdTokenFromMarket`:

```solidity
// if the market is in the ADL state, it reduces the requested USD
// Token amount by multiplying it by the ADL factor, which must be < 1
```

should never return 1, but in fact it can return 1.

## Vulnerable Code

`Market::getAutoDeleverageFactor`:

```solidity
function getAutoDeleverageFactor(
        Data storage self,
        UD60x18 delegatedCreditUsdX18,
        SD59x18 totalDebtUsdX18
    )
        internal
        view
        returns (UD60x18 autoDeleverageFactorX18)
    {
        SD59x18 sdDelegatedCreditUsdX18 = delegatedCreditUsdX18.intoSD59x18();
        if (sdDelegatedCreditUsdX18.lte(totalDebtUsdX18) || sdDelegatedCreditUsdX18.isZero()) {
            return UD60x18_UNIT;
        }
        UD60x18 marketDebtRatio = totalDebtUsdX18.div(sdDelegatedCreditUsdX18).intoUD60x18();
        UD60x18 autoDeleverageStartThresholdX18 = ud60x18(self.autoDeleverageStartThreshold);
        UD60x18 autoDeleverageEndThresholdX18 = ud60x18(self.autoDeleverageEndThreshold);
        UD60x18 autoDeleverageExponentZX18 = ud60x18(self.autoDeleverageExponentZ);
@>      UD60x18 unscaledDeleverageFactor = Math.min(marketDebtRatio, autoDeleverageEndThresholdX18).sub(
            autoDeleverageStartThresholdX18
        ).div(autoDeleverageEndThresholdX18.sub(autoDeleverageStartThresholdX18));
        autoDeleverageFactorX18 = unscaledDeleverageFactor.pow(autoDeleverageExponentZX18);

    }
```

As you can see in above marked part in the function, the unscaledDeleverageFactor is defined as in the natspec. The issue arises if marketDebtRatio > autoDeleverageEndThreshooldX18, since numerator and denominator would simplify each other out, resulting in `UD60x18 autoDeleverageFactorX18` returned as equal to 1.

## Impact Explanation

While it is under these circumstances possible that the market is in a state of auto deleveraging, that above function returns a value which explicitly does not deleverage anything, the impact of this is medium, since it only affects a discrete point on the curve. While it would be possible though, that if the Registered Engine calls `CreditDelegationBranch::withdrawUsdTokenFromMarket` with a considerably large withdrawal, which should have been already deleveraged, it would be processed without deleveraging. Therefore it can not be entirely ignored.

## Used Tools

Manual Review

## Recommendation

An easy mitigation would be to subtract an `epsilon` in the numerator if `autoDeleverageEndThresholdX18` was returned, with epsilon being a very small number (e.g., 1e-18 in the UD60x18 fixed-point representation) that does not materially affect normal operations but guarantees the factor is always less than 1.
