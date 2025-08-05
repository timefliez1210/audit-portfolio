## Description

In `UsdTokenSwapConfig::getPremiumDiscountFactor` the proposed initial curve for
`f(x) = y_min + Δy * ((x - x_min) / (x_max - x_min))^z | x ∈ [x_min, x_max]`
as mentioned is:
`f(x) = 1 + 9 * ((x - 0.3) / 0.5)^3`. The arising issue of this proposition is that at any point if y\_min >= 1 the function will automatically revert, if `vaultDebtUsdX18 < 0`, since

```javascript
premiumDiscountFactorX18 =
            vaultDebtUsdX18.lt(SD59x18_ZERO) ? UD60x18_UNIT.sub(pdCurveYX18) : UD60x18_UNIT.add(pdCurveYX18);
```

will try to assign a negative value to `UD60x18 result` in `Helpers.sol` which is logically defined as `UD60x18`.

## Math

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

## Impact

In the current version and with proposed base configuration the protocol basically DoS'es itself at any point `vaultDebtUsdX18` is negative. Therefore this function prevents the execution of `StabilityBranch::fulfillSwap` and `StabilityBranch::initiateSwap` locking user funds in the protocol until `vaultDebtUsdX18` is positive and those functions become executable.

Since the issues arising are temporary I rate this as Medium.

## Used Tools

Manual Review

## Recommended Mitigation

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
            pdCurveYMaxX18.sub(pdCurveYMinX18).mul(        pdCurveXX18.sub(pdCurveXMinX18).div(pdCurveXMaxX18.sub(pdCurveXMinX18)).pow(pdCurveZX18)
            )
 );
+if(vaultDebtUsdX18.lt(SD59x18_ZERO) && pdCurveYX18.gt(UD60x18_UNIT)) {
+            premiumDiscountFactorX18 = UD60x18_UNIT; //*add here the value*
+            return premiumDiscountFactorX18;
+}
premiumDiscountFactorX18 =
            vaultDebtUsdX18.lt(SD59x18_ZERO) ? UD60x18_UNIT.sub(pdCurveYX18) : UD60x18_UNIT.add(pdCurveYX18);
    }
```
