lemonmon

medium

# `Comptroller::withdrawRewards` accounting error results in incorrect inflation index

## Summary

In `Comptroller::withdrawRewards`, `totalFrozen` was subtracted twice from `totalStaked`, which will update `Comptroller::gInflationIndex` based on incorrect information.
Also, if more than half of `totalStaked` is frozen, the `Comptroller::withdrawRewards` will revert, so no one can call `UserManager::stake` or `UserManager::unstake`.

## Vulnerability Detail

In `Comptroller:withdrawRewards` calls `_getUserManagerState` and saves it as `userManagerState`:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/token/Comptroller.sol?plain=1#L248

Note that returned value of `userManagerState.totalStaked` is equivalent to `userManager.totalStaked() - userManager.totalFrozen()`:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/token/Comptroller.sol?plain=1#L306-L317

However, in the `Comptroller::withdrawRewards` function, the returned value `userManagerState.totalStaked` will be subtracted by totalFrozen again:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/token/Comptroller.sol?plain=1#L260-L261

So, the `totalStaked_` is equivalent to `userManager.totalStaked() - 2 * userManager.totalFrozen()`, which was used to calculate `gInflationIndex` in the line 261 of `Comptroller.sol`. It will result in incorrect update of the `gInflationIndex`.

Moreover, if more than half of total staked values are frozen, the line 260 in Comptroller.sol will revert from underflow. The `Comptroller::withdrawRewards` function is used in `UserManager::stake`, `UserManager::unstake` and `UserManager::withdrawRewards`, thus all of these function will stop working when the condition is met.

## Impact

- Updated to incorrect `gInflationIndex`
- Revert when more than half of total staked is frozen

## Code Snippet

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/token/Comptroller.sol?plain=1#L248
https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/token/Comptroller.sol?plain=1#L306-L317
https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/token/Comptroller.sol?plain=1#L260-L261

## Tool used

Manual Review

## Recommendation

the following line 

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/token/Comptroller.sol?plain=1#L248

should be:

```solidity
        uint256 totalStaked_ = userManagerState.totalStaked;
```

