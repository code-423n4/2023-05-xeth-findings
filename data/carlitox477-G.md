# `xETH`, `CVXStaker`: Can make constructor and functions payable to save gas
Marking the constructor as payable will lower the gas cost for legitimate callers because the compiler will not include checks for whether a payment was provided.

Next functions require especial roles to be called and can use the payable modifier in order to exclude checks for whether a payment was provided:
* `xETH::mintShares`
* `xETH::burnShares`
* `CVXStaker::setCvxPoolInfo`
* `CVXStaker::recoverToken`
* `CVXStaker::withdrawAndUnwrap`
* `CVXStaker::withdrawAllAndUnwrap`

# `xETH::setAMO`: Cache `curveAMO` and use `newAmo` to save gas
In this way we avoid 2 storage copy to registries.

```diff
    function setAMO(address newAMO) external onlyRole(DEFAULT_ADMIN_ROLE) {
        /// @dev if the new AMO is address(0), revert
        if (newAMO == address(0)) {
            revert AddressZeroProvided();
        }

+       address _curveAMO = curveAMO;
        /// @dev if there was a previous AMO, revoke it's powers
-       if (curveAMO != address(0)) {
-           _revokeRole(MINTER_ROLE, curveAMO);
+       if (_curveAMO != address(0)) {
+           _revokeRole(MINTER_ROLE, _curveAMO);
        }

        // @todo call marker method to check if amo is responding

        /// @dev set the new AMO
        curveAMO = newAMO;

        /// @dev grant it the MINTER_ROLE
-       grantRole(MINTER_ROLE, curveAMO);
+       grantRole(MINTER_ROLE, newAMO);
    }
```

# `wxETH::_accrueDrip`: Cache `lockedFunds` to save gas
This would avoid multiple access to storage variable.
```diff
    function _accrueDrip() private {
        /// @dev if drip is disabled, no need to accrue
        if (!dripEnabled) return;

        /// @dev blockDelta is the difference between now and last accrual
        uint256 blockDelta = block.number - lastReport;

        if (blockDelta != 0) {
            /// @dev calculate dripAmount using blockDelta and dripRatePerBlock
            uint256 dripAmount = blockDelta * dripRatePerBlock;

            /// @dev We can only drip what we have
            /// @notice if the dripAmount is greater than the lockedFunds
            /// @notice then we set the dripAmount to the lockedFunds
-           if (dripAmount > lockedFunds) dripAmount = lockedFunds;
+           uint256 _lockedFunds = lockedFunds;
+           if (dripAmount > _lockedFunds) dripAmount = _lockedFunds;

            /// @dev unlock the dripAmount from the lockedFunds
            /// @notice so that it reflects the amount of xETH that is available
            /// @notice and the exchange rate shows that
-           lockedFunds -= dripAmount;
+           _lockedFunds -= dripAmount;

            /// @dev set the lastReport to the current block
            lastReport = block.number;

            /// @notice if there are no remaining locked funds
            /// @notice the drip must be stopped.
-           if (lockedFunds == 0) {
+           if (_lockedFunds == 0) {
                dripEnabled = false;
                emit DripStopped();
            }
+           lockedFunds = _lockedFunds;

            /// @dev emit succesful drip event with dripAmount, lockedFunds and xETH balance
-           emit Drip(dripAmount, lockedFunds, xETH.balanceOf(address(this)));
+           emit Drip(dripAmount, _lockedFunds, xETH.balanceOf(address(this)));
        }
    }
```

# `wxETH::addLockedFunds` can avoid 1 storage access to `lockedFunds`
```diff
    function addLockedFunds(uint256 amount) external onlyOwner drip {
        /// @dev if amount or _dripRatePerBlock is 0, revert.
        if (amount == 0) revert AmountZeroProvided();

        /// @dev transfer xETH from the user to the contract
        xETH.safeTransferFrom(msg.sender, address(this), amount);

        /// @dev add the amount to the locked funds variable
-       lockedFunds += amount;
+       uint256 _lockedFunds = lockedFunds + amount;
+       lockedFunds = _lockedFunds;

-       emit LockedFundsAdded(amount, lockedFunds);
+       emit LockedFundsAdded(amount, _lockedFunds);
    }
```

# `CVX::getReward` can cache `rewardsRecipient` to save gas in for loop
This would change a storage access for a memory access in each iteration
```diff
-   if (rewardsRecipient != address(0)) {
+   address _rewardsRecipient = rewardsRecipient;
+   if (_rewardsRecipient != address(0)) {
        for (uint i = 0; i < rewardTokens.length; i++) {
            uint256 balance = IERC20(rewardTokens[i]).balanceOf(
                address(this)
            );
-           IERC20(rewardTokens[i]).safeTransfer(rewardsRecipient, balance);
+           IERC20(rewardTokens[i]).safeTransfer(_rewardsRecipient, balance);
        }
    }
```


# `CVXStaker::getReward` use storage pointer and cache `rewardTokens[i]` to save gas

```diff
+   address[] storage _rewardTokens = rewardTokens;
    for (uint i = 0; i < rewardTokens.length; i++) {
+       address rewardToken = _rewardTokens[i]
-       uint256 balance = IERC20(rewardTokens[i]).balanceOf(
+       uint256 balance = IERC20(rewardToken).balanceOf(
            address(this)
        );
-       IERC20(rewardTokens[i]).safeTransfer(rewardsRecipient, balance);
+       IERC20(rewardToken).safeTransfer(rewardsRecipient, balance);
    }
```

# `CVXStaker::getReward` can check `claimExtras` to avoid unnecessary execution of for loop
If no claim of rewards is done then there is no nees of rewards transfer, saving gas in the function execution. This means:
```diff
    function getReward(bool claimExtras) external {
        IBaseRewardPool(cvxPoolInfo.rewards).getReward(
            address(this),
            claimExtras
        );
-       if (rewardsRecipient != address(0)) {
+       if (rewardsRecipient != address(0) && claimExtras) {
            for (uint i = 0; i < rewardTokens.length; i++) {
                uint256 balance = IERC20(rewardTokens[i]).balanceOf(
                    address(this)
                );
                IERC20(rewardTokens[i]).safeTransfer(rewardsRecipient, balance);
            }
        }
    }
```

# `AMO2::rebalanceUp` can cache `cvxStaker` to save gas
This would avoid one storage access
```diff
    CVXStaker memory _cvxStaker = cvxStaker;
-   uint256 amoLpBal = cvxStaker.stakedBalance();
+   uint256 amoLpBal = _cvxStaker.stakedBalance();

    // if (amoLpBal == 0 || quote.lpBurn > amoLpBal) revert LpBalanceTooLow();
    if (quote.lpBurn > amoLpBal) revert LpBalanceTooLow();

-   cvxStaker.withdrawAndUnwrap(quote.lpBurn, false, address(this));
+   _cvxStaker.withdrawAndUnwrap(quote.lpBurn, false, address(this));
```

# `AMO2.cooldownBlocks`, `AMO2.lastRebalanceBlock` `AMO2.CooldownBlocksUpdated` can use uint40 type instead of uint256
Given that each block in ETH needs about 12 seconds to be validated, and that  $\frac{2^40}{365 * 24 * 60 * 60} = 34865.285$ we can use `uin40` instead of `uint256` to save `AMO2.cooldownBlocks`, `AMO2.lastRebalanceBlock` values, and then situating them next to address variable in order to package them.

This type modification can also be done to event `CooldownBlocksUpdated` in order to save gas.

# `AMO2.maxSlippageBPS`, `AMO2.MaxSlippageBPSUpdated` can use uint40 type instead of uint256
Given invariant `0.0006e18 <= maxSlippageBPS <0.15e18 < 15 * 10^16` and that:
$$15 \times  10^{16}$$
$$(5 \times 3) \times  (2 \times 5)^{16}$$
$$5 \times 3 \times  2^{16} \times 5^{16}$$
$$2^{16} \times 3 \times 5^{17}  $$
$$2^{16} \times 3 \times 5^{17} < 2^{16} \times 2^{4} \times (2^{3})^{17}$$
$$2^{16} \times 3 \times 5^{17} < 2^{20} \times 2^{51}$$
$$2^{16} \times 3 \times 5^{17}< 2^{71}$$

Then we can use type `uint128` to storage `maxSlippageBPS` and modify event parameter types of  `AMO2.MaxSlippageBPSUpdated`

# `AMO2::setRebalanceDefender` can cache `defender` to save gas
This would avoid 3 storage access
```diff
    function setRebalanceDefender(
        address newDefender
    ) external onlyRole(DEFAULT_ADMIN_ROLE) {
        if (newDefender == address(0)) revert ZeroAddressProvided();

+       address _defender = defender;

-       if (defender != address(0)) {
-           _revokeRole(REBALANCE_DEFENDER_ROLE, defender);
+       if (_defender != address(0)) {
+           _revokeRole(REBALANCE_DEFENDER_ROLE, _defender);
        }

-       emit DefenderUpdated(defender, newDefender);
+       emit DefenderUpdated(_defender, newDefender);

        defender = newDefender;
        _grantRole(REBALANCE_DEFENDER_ROLE, newDefender);
    }
```


# `AMO2::addLiquidity` and `AMO2::addLiquidityOnlyStETH` can cache `cvxStaker` to save gas
This would avoid 1 storage access
```diff
+   address _cvxStaker = cvxStaker;
-   IERC20(address(curvePool)).safeTransfer(address(cvxStaker), lpOut);
-   cvxStaker.depositAndStake(lpOut);
+   IERC20(address(curvePool)).safeTransfer(address(_cvxStaker), lpOut);
+   _cvxStaker.depositAndStake(lpOut);
```