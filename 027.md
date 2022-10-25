lemonmon

medium

# `AssetManager::withdraw` will not return false, when fail to send the withdraw amount

## Summary

When `UserManager` or `UToken` calls on `AssetManager::withdraw`, the given `amount` should be transferred to `account`, either from `AssetManager` itself or by `moneyMarket`s. However, when there is not enough asset to be transferred from `AssetManager` itself or from `moneyMarket`s, it returns true, without reverting.

Since other contracts, who is using the `AssetManager::withdraw`, assumes that the `amount` is transferred, it causes problems such as:
  - `UserManager::unstake`: the user might get less than the amount the user unstaked
  - `UToken::borrow`: the user might get less than the amount the user borrowed
  - `UToken::redeem`: the user might get less than the amount the user redeemed
  - `UToken::removeReserves`: the admin might transfer out less than intended
  - In some cases above will also result in accounting error

## Vulnerability Detail

The `AssetManager::withdraw` function will transfer the given `amount` from `AssetManager` itself or from `moneyMarket`, since the values are distributed. The function keeps track of the remaining values to be transferred in `remaining` local variable. However, the function returns `true` even if there are some non zero `remaining` left to transfer.

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/asset/AssetManager.sol#L328-L370

Other contracts who can call the function `AssetManager::withdraw` are `UserManager` and `UToken`. They assume that if the return from `AssetManager::withdraw` is `true`, the whole `amount` is transferred to the recipient, and update accounting accordingly. As the result, it will cause some cases that less `amount` is transferred out, yet the transaction does not fail.

The functions who uses `AssetManager::withdraw` are following:
  - `UserManager::unstake`
    - the staker's `stakedAmount` will be decreased by `amount`, but the staker might get less than actually unstaked amount
  - `UToken::borrow`: the user might get less than the amount the user borrowed
    - the user's `principal` will be increased by the borrowed `amount` and `fee`, but the user might get less than the asked `amount`
  - `UToken::redeem`: the user might get less than the amount the user redeemed
    - the user's `UToken` is burned, but the user might get less than the burned amount.
  - `UToken::removeReserves`: the admin might transfer out less than intended
    - the receiver might get less than what admin intended. The `totalReserves` might be reduced more than what was transferred out.


## Impact

Users might get less amount transferred from the `AssetManager` than they should get

## Code Snippet

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/asset/AssetManager.sol#L328-L370

## Tool used

Manual Review

## Recommendation

Check the `remaining` to be transferred and return `false` if it is bigger than zero

