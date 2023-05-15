# Low issues

## 1. The first staker of the wxETH can get all the unlocked rewards immediately in the same block.
code lines: https://github.com/code-423n4/2023-05-xeth/blob/main/src/wxETH.sol#L202-L216

For the first staker of the wxETH, the totalSupply of the wxETH is 0. So he can wrap the xETH to wxETH as 1:1.
```
function exchangeRate() public view returns (uint256) {
    /// @dev if there are no tokens minted, return the initial exchange rate
    uint256 _totalSupply = totalSupply();
    if (_totalSupply == 0) {
        return INITIAL_EXCHANGE_RATE;
    }
```

If the drip is started before the first staker, the unlocked funds can be withdrew immediately in the same block by calling unstake.

## 2. Lack of initialization check about cvxPoolInfo in the CVXStaker
code lines: https://github.com/code-423n4/2023-05-xeth/blob/main/src/CVXStaker.sol#L126-L132

If the cvxPoolInfo has not been initialized before calling `CVXStaker.depositAndStake`, which means the `cvxPoolInfo.pid = 0`, some unexpected token transactions will be made. 

Because the pid 0 is a valid and active pool in the CVX booster.
```
        if (!isCvxShutdown()) {
            clpToken.safeIncreaseAllowance(address(booster), amount);
            booster.deposit(cvxPoolInfo.pId, amount, true);
        }
```

## 3. CVXStaker.setCvxPoolInfo lacks address validation
code lines: https://github.com/code-423n4/2023-05-xeth/blob/main/src/CVXStaker.sol#L62-L72 

In fact, the `token` and `rewards` address can be got by calling `booster.poolInfo(cvxPoolInfo.pId)`. It ensures that the addresses are valid and consistent.


# Non-critical issue
## Conceptual error in document comments

code lines: https://github.com/code-423n4/2023-05-xeth/blob/main/src/CVXStaker.sol#L55-L61

CVXStaker.setCvxPoolInfo:
```
    /**
     * @dev Sets the CVX pool information.
     * @param _pId The pool ID of the CVX pool.
     * @param _token The address of the CLP token.
     * @param _rewards The address of the CVX reward pool.
     * Only the contract owner can call this function.
     */
    function setCvxPoolInfo(
        uint32 _pId,
        address _token,
        address _rewards
    )
```
The param `_token` should be the wrapped DepositToken of the CVX pool, instead of the LP token.