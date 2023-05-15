## Redundant store
`lastReport = block.number;` [in `wxETH.stopDrip()`](https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/wxETH.sol#L191) is redundant. This is already done in the `drip` modifier.
It is uneccesary to update `lastReport` [in the constructor](https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/wxETH.sol#L73), since `dripEnabled` is `false` by default and has to be enabled by calling `startDrip()`, which immediately sets `lastReport`.
For the same reason it is not necessary to update `lastReport` when stopping the drip. The next time it's started it will update.
In general, only update when starting and dripping. The other functions don't care what `lastReport` was or will be.

## Redundant function
[`AMO2.addLiquidityOnlyStETH()`](https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/AMO2.sol#L561-L578) can be removed if [`AMO2.addLiquidity()`](https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/AMO2.sol#L538) is modified
```diff
+ if (xETHAmount > 0)
xETH.mintShares(xETHAmount);
```
so that it may be called with `xETHAmount == 0` without reverting. `addLiquidityOnlyStETH(stETHAmount, minLpOut)` can then be achieved by calling `addLiquidity(stETHAmount, 0, minLpOut)`.

## `isXETHToken0` can be a `uint256`
[In the constructor of AMO2.sol](https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/AMO2.sol#L196-L197) we can refactor
```diff
constructor(
...
    - bool isXETHToken0
    + uint256 _xETHIndex
) {
...
    - xETHIndex = isXETHToken0 ? 0 : 1;
    - stETHIndex = isXETHToken0 ? 1 : 0;
    + xETHIndex = _xETHIndex;
    + stETHIndex = 1 - _xETHIndex;
```
This reverts if `_xETHIndex` isn't `0` or `1`, and is cheaper.