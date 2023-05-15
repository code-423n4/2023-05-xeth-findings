## Precision loss
In [`AMO2.bestRebalanceUpQuote()`](https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/AMO2.sol#L343-L345) and [`AMO2.bestRebalanceDownQuote()`](https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/AMO2.sol#L368-L370) there are rounding losses of the form `(a/b)*c <= a*c/b`.
Refactor
```diff
bestQuote.min_xETHReceived = applySlippage(
-     (vp * defenderQuote.lpBurn) / BASE_UNIT
- );
+     (vp * defenderQuote.lpBurn)
+ ) / BASE_UNIT;
```
and
```diff
bestQuote.minLpReceived = applySlippage(
-     (BASE_UNIT * defenderQuote.xETHAmount) / vp
- ); 
+     (BASE_UNIT * defenderQuote.xETHAmount) 
+ ) / vp;
```

## No check that `REBALANCE_UP_THRESHOLD >= REBALANCE_DOWN_THRESHOLD`
It is assumed that `REBALANCE_UP_THRESHOLD >= REBALANCE_DOWN_THRESHOLD` (or even `>`), but when setting these values this is not guaranteed.
Consider checking that this is fulfilled when setting them in [`AMO2.setRebalanceUpThreshold](https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/AMO2.sol#L495-L504) and [`AMO2.setRebalanceUpThreshold`](https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/AMO2.sol#L512-L521).

## Redundant function
[`AMO2.addLiquidityOnlyStETH()`](https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/AMO2.sol#L561-L578) can be removed if [`AMO2.addLiquidity()`](https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/AMO2.sol#L538) is modified
```diff
+ if (xETHAmount > 0)
xETH.mintShares(xETHAmount);
```
so that it may be called with `xETHAmount == 0` without reverting. `addLiquidityOnlyStETH(stETHAmount, minLpOut)` can then be achieved by calling `addLiquidity(stETHAmount, 0, minLpOut)`.

## It's a percentage, not a ratio
What is called "ratio" in the comments in [`AMO2.preRebalanceCheck()`](https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/AMO2.sol#L202-L225) is actually a _percentage_, `xEthPct`. Consider amending the comments to reflect this fact.

## Remove commented-out code
[This commented-out line](https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/AMO2.sol#L258) is equivalent in its context to the line below. Remove it.