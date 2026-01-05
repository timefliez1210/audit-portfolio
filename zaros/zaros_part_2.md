# Zaros Part 2 - Findings Report

## Table of Contents
- High Risk Findings
    - [H-01. getAutoDeleverageFactor in Market.sol can return 1 even if it should never, while deleverage mode is triggered](#H-01)
- Medium Risk Findings
    - [M-01. Assuming the proposed initial curve in UsdTokenSwapConfig for getPremiumDiscountFactor will get applied it will DoS the whole functions execution as soon as vaultDebtUsdX18 has a negative value](#M-01)

---

## Contest Summary

**Sponsor:** Zaros

**Dates:** Jan 20th, 2025 - Feb 6th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-01-zaros-part-2)

---

## Results Summary

| Severity | Count |
|----------|-------|
| High     | 1     |
| Medium   | 1     |
| Low      | 0     |

---

# High Risk Findings

## <a id='H-01'></a>H-01. getAutoDeleverageFactor in Market.sol can return 1 even if it should never, while deleverage mode is triggered

### Summary

`Market::getAutoDeleverageFactor` as described in `CreditDelegationBranch::withdrawUsdTokenFromMarket` should never return 1, but in fact it can return 1.

### Finding Description

`Market::getAutoDeleverageFactor` as described in `CreditDelegationBranch::withdrawUsdTokenFromMarket`:

```solidity
// if the market is in the ADL state, it reduces the requested USD
// Token amount by multiplying it by the ADL factor, which must be < 1
```

should never return 1, but in fact it can return 1.

### Vulnerable Code

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

### Impact Explanation

While it is under these circumstances possible that the market is in a state of auto deleveraging, that above function returns a value which explicitly does not deleverage anything, the impact of this is medium, since it only affects a discrete point on the curve. While it would be possible though, that if the Registered Engine calls `CreditDelegationBranch::withdrawUsdTokenFromMarket` with a considerably large withdrawal, which should have been already deleveraged, it would be processed without deleveraging. Therefore it can not be entirely ignored.

### Proof of Concept

N/A - Logic error is evident from code inspection.

### Recommendation

An easy mitigation would be to subtract an `epsilon` in the numerator if `autoDeleverageEndThresholdX18` was returned, with epsilon being a very small number (e.g., 1e-18 in the UD60x18 fixed-point representation) that does not materially affect normal operations but guarantees the factor is always less than 1.

---

# Medium Risk Findings

## <a id='M-01'></a>M-01. Assuming the proposed initial curve in UsdTokenSwapConfig for getPremiumDiscountFactor will get applied it will DoS the whole functions execution as soon as vaultDebtUsdX18 has a negative value

### Summary

In `UsdTokenSwapConfig::getPremiumDiscountFactor` the proposed initial curve will cause a DoS when `vaultDebtUsdX18` has a negative value.

### Finding Description

In `UsdTokenSwapConfig::getPremiumDiscountFactor` the proposed initial curve for
`f(x) = y_min + Δy * ((x - x_min) / (x_max - x_min))^z | x ∈ [x_min, x_max]`
as mentioned is:
`f(x) = 1 + 9 * ((x - 0.3) / 0.5)^3`. The arising issue of this proposition is that at any point if y\_min >= 1 the function will automatically revert, if `vaultDebtUsdX18 < 0`, since

```solidity
premiumDiscountFactorX18 =
            vaultDebtUsdX18.lt(SD59x18_ZERO) ? UD60x18_UNIT.sub(pdCurveYX18) : UD60x18_UNIT.add(pdCurveYX18);
```

will try to assign a negative value to `UD60x18 result` in `Helpers.sol` which is logically defined as `UD60x18`.

### Math

Generally speaking the function design of `f(x) = y_min + Δy * ((x - x_min) / (x_max - x_min))^z | x ∈ [x_min, x_max]` is inherently flawed in it's execution. In depth looking into an example with a negative `vaultDebtUsdX18` would look like this:

Proposed Base Configuration:

f(x) = 1 + 9 \* ((x - 0.3) / 0.5)^3

Let's assume the vault has a vaultDebtUsdX18 = -1800 and holds a total value of 3000 so
we would get:

```
f(0.6) = 1 + 9 * ((0.6 - 0.3) / 0.5)^3
       = 1 + 9 * (0.6)^3
       = 1 + 9 * 0.216
       = 1 + 1.944
       = 2.944
```

which would in conclusion result in a discount factor of
`result = 1 - 2.944 = -1.944` which will cause a revert, since we try to assign a negative value into an `UD60x18`.

In the proposed standard configuration explicitly, but more generally in the whole function, any value of y\_min >= 1 will cause reverts by default if `vaultDebtUsdX18` is negative. Even with
y\_min < 1, it would still depend on y\_max or delta\_y and the result within the brackets.

### Impact Explanation

In the current version and with proposed base configuration the protocol basically DoS'es itself at any point `vaultDebtUsdX18` is negative. Therefore this function prevents the execution of `StabilityBranch::fulfillSwap` and `StabilityBranch::initiateSwap` locking user funds in the protocol until `vaultDebtUsdX18` is positive and those functions become executable.

Since the issues arising are temporary I rate this as Medium.

### Likelihood Explanation

This issue occurs whenever the vault debt becomes negative, which is an expected state in normal protocol operations. Likelihood is Medium.

### Proof of Concept

N/A - Logic error is evident from mathematical analysis.

### Recommendation

Depending on the fact how this curve and functionality is intended, it would be a solution to use the absolute of the value, like in previous code with a different .sub() method, or simply by executing the subtraction conditional since `| x - y | = | y - x |`.

Otherwise another option would be to make the curve itself more rigid in design with for example:
`f(x) = (y_min / Δy) * ((x - x_min) / (x_max - x_min))^z | x ∈ [x_min, x_max]`
with `y_min / Δy = k` | k ∈ ]0, 1\[ (exclusive)

The third, and best mitigation, would be to clamp the result of f(x) so `pdCurveYX18 <= 1 ` for `vaultDebtUsdX18 < 0`. The code would look like this:

```diff
function getPremiumDiscountFactor(
        Data storage self,
        UD60x18 vaultAssetsValueUsdX18,
        SD59x18 vaultDebtUsdX18
    )
        internal
        view
        returns (UD60x18 premiumDiscountFactorX18)
{
....
/// snip /// 
UD60x18 pdCurveYX18 = pdCurveYMinX18.add(
            pdCurveYMaxX18.sub(pdCurveYMinX18).mul(        pdCurveXX18.sub(pdCurveXMinX18).div(pdCurveXMaxX18.sub(pdCurveXMinX18)).pow(pdCurveZX18)
            )
 );
+if(vaultDebtUsdX18.lt(SD59x18_ZERO) && pdCurveYX18.gt(UD60x18_UNIT)) {
+            premiumDiscountFactorX18 = UD60x18_UNIT; //*add here the value*
+            return premiumDiscountFactorX18;
+}
premiumDiscountFactorX18 =
            vaultDebtUsdX18.lt(SD59x18_ZERO) ? UD60x18_UNIT.sub(pdCurveYX18) : UD60x18_UNIT.add(pdCurveYX18);
    }
```
