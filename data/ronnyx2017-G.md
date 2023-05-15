## 1. No need to set the lastReport in the wxETH.constructor

    code lines: https://github.com/code-423n4/2023-05-xeth/blob/main/src/wxETH.sol#L73

    Because the `dripEnabled` is false as default, the drip won't be started until the owner calls the startDrip function, which will set the drip lastReport to current block.

## 2. No need to call the drip modifier in the `startDrip` function.

 code lines: https://github.com/code-423n4/2023-05-xeth/blob/main/src/wxETH.sol#L170-L174

If `dripEnabled` is true, the first line of the `startDrip` will revert directly:
```
function startDrip() external onlyOwner drip {
    /// @dev if the drip is already running, revert
    if (dripEnabled) revert DripAlreadyRunning();
```

So the `dripEnabled` must be false. If `dripEnabled` is false, the drip modifier will return directly without any changes:
```
function _accrueDrip() private {
    /// @dev if drip is disabled, no need to accrue
    if (!dripEnabled) return;
```

## 3. Unused variable in the storage

code lines: https://github.com/code-423n4/2023-05-xeth/blob/main/src/CVXStaker.sol#LL68C36-L68C36

The global var `cvxPoolInfo.token` is never used in the CVXStaker contract.

## 4. CVXStaker.getReward should check if the reward balance is 0 before calling external erc20 transfer.

code lines: https://github.com/code-423n4/2023-05-xeth/blob/main/src/CVXStaker.sol#L191-L196

If the reward balance is 0, it's no need to call external erc20 transfer function.

## 5. Should check lockedFunds & dripRatePerBlock != 0 before startDrip

code lines: https://github.com/code-423n4/2023-05-xeth/blob/main/src/wxETH.sol#L222-L256

If lockedFunds is 0, dripEnabled will be set to false again at next drip.

If dripRatePerBlock is 0, drip can't unlock any funds but increaces gas consumption after startDrip.