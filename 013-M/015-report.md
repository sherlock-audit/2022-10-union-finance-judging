0x0

high

# Incorrect Vouch Creation Timestamp

## Summary

Vouches are used within the project to establish the amount of assets a borrower is trusted to borrow. When a vouch is created there is a member of the `Vouch` struct that is used to hold the creation block height. This is not being set correctly.

## Vulnerability Detail

`UserManager.updateTrust`

When a vouch is created via `updateTrust` a record is stored in a `Vouch` struct. This has a member named `lastUpdated` which a docstring indicates is used to store the block number of the last update.

On [line 555](https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L555) this is currently being set statically to `0` instead of the block height at the time of execution. This is used by `getFrozenInfo` when calculating `memberTotalFrozen` and `memberFrozenCoinAge` and these are being calculated incorrectly.

## Impact

More token rewards from the Comptroller will be awarded than anticipated due to a larger block range being calculated.

## Code Snippet

```solidity
vouchers[borrower].push(Vouch(staker, trustAmount, 0, 0));
```

## Tool used

Manual Review

## Recommendation

Modify the last argument in this call to be the block height at the time of execution:

```solidity
vouchers[borrower].push(Vouch(staker, trustAmount, 0, block.number));
```
