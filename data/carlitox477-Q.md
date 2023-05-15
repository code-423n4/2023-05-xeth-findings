# Function which are not called internally in the contract nor by inherithed contract should be external rather than public
This is the case of:
* ```xETH::mintShares```
* ```xETH::burnShares```

# `wxETH::previewStake` unnecessary revert
Given that `wxETH::previewStake` is a view function, line `if (xETHAmount == 0) revert AmountZeroProvided();` seems unnecessary.

# `CVXStaker::getReward`: Possible DOS if `rewardTokens` is enough large
## Line of codes
https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/CVXStaker.sol#L185-L198

## Description
If enough `rewardTokens` are added, then the transaction can reach block gas limit, produce a DOS of this function

## Impact
DOS of function `getReward` due to for loop over `rewardTokens`

## Mitigation steps
Remove for loop from `getReward` function and create anew function `claimRewards`:
```solidity
    function claimRewards(uint initialIndex, uint lastIndex) external {
        require(initialIndex < lastIndex);
        require(lastIndex <= rewardTokens.length);
        for (uint i = initialIndex; i < lastIndex; ) {
            uint256 balance = IERC20(rewardTokens[i]).balanceOf(
                address(this)
            );
            IERC20(rewardTokens[i]).safeTransfer(rewardsRecipient, balance);
            unchecked {++i};
        }
    }
```

# `CVXStaker::getReward`: unchecked `IBaseRewardPool::getReward` return value
`IBaseRewardPool::getReward` is expected to return a boolean. Without handling the return value explicitly, the transaction may risk fails silently.

# `AMO2::removeLiquidityOnlyStETH` does not return the stETH amount unstaked/unpaired
While `AMO2::removeLiquidity` return how much stETH as well as how much xETH we get from removing liquidity, `AMO2::removeLiquidityOnlyStETH`  does not. It seems incongruent given that they have a similar functionality.

# `CVXStake` centralization issues
## Impact
Contracts have special addresses with privileged rights to perform admin tasks and need to be trusted to not perform malicious updates or drain funds. In particular:
* `CVXStake::withdrawAndUnwrap` with `onlyOperatorOrOwner` modifier. This function allows the owner or the operator to get all the staked tokens if owner key are leaked.
* `CVXStake::depositAndStake` with `onlyOperator` modifier

Note: This centralization issues were not caught by the bot