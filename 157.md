hyh

high

# Vouchers and vouchees indices become corrupted by UserManager's cancelVouch

## Summary

UserManager's cancelVouch() doesn't update `voucherIndexes` and `voucheeIndexes` entries for the last `vouchers` and `vouchees` arrays element that was moved to the position of the deleted element, keeping the non-existent link that can be filled with new data thereafter, making old element pointing to the new trust/locked values, that are incorrect for it.

## Vulnerability Detail

Broken entries that is created this way has the immediate effect of unavailability of the several vouch operating functions for the corresponding staker and borrower. I.e. these functions obtain an index, which is not longer valid (index is the current `length` of the array), and revert on trying to access the corresponding element.

Moreover, if an additional voucher either for the borrower or staker be created with updateTrust(), two elements of the corresponding Indices array will point to the same new element, which has generally different locked/trust amount than the old one, whose pointer was previously lost. This way the old element will have incorrect trust/locked amount returned by these functions.

## Impact

Immediate impact is unavailability of getLockedStake(), getVouchingAmount(), updateTrust(), cancelVouch() and debtWriteOff() for the borrower-staker combination that was this last element cancelVouch() moved.

Furthermore, if a new entry is placed, which is a high probability event being a part of normal activity, then old entry will point to incorrect staked/locked amount, and a various violations of the related constrains become possible.

Some examples are: trust can be updated via updateTrust() to be less then real locked amount; vouch cancellation can become blocked (say new voucher borrower gains a long-term lock and old lock with misplaced index cannot be cancelled until new lock be fully cleared, i.e. potentially for a long time). 

The total impact is up to freezing the corresponding staker's funds as this opens up a way for various exploitations, for example a old entry's borrower can use the situation of unremovable trust and utilize this trust that staker would remove in a normal course of operations, locking extra funds of the staker this way.

The whole issue is a violation of the core UNION vouch accounting logic, with misplaced indices potentially piling up. Placing overall severity to be **high**.

## Code Snippet

cancelVouch() do not update voucherIndexes[][] entry for the last voucher (sitting at `vouchers[borrower].length - 1`) that was moved to the `idx` index of the deleted one:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L587-L598

```solidity
        // Remove borrower from vouchers array by moving the last item into the position
        // of the index being removed and then poping the last item off the array
        vouchers[borrower][voucherIndex.idx] = vouchers[borrower][vouchers[borrower].length - 1];
        vouchers[borrower].pop();
        delete voucherIndexes[borrower][staker];

        // Remove borrower from vouchee array by moving the last item into the position
        // of the index being removed and then poping the last item off the array
        Index memory voucheeIndex = voucheeIndexes[borrower][staker];
        vouchees[staker][voucheeIndex.idx] = vouchees[staker][vouchees[staker].length - 1];
        vouchees[staker].pop();
        delete voucheeIndexes[borrower][staker];
```

In other words, `vouchers` and `vouchees` arrays do not perform full `voucherIndexes` and `voucheeIndexes` indices update, missing the update for the moved element. This leads to the discrepancies in `voucherIndexes` and `voucheeIndexes` data as the `voucherIndexes[borrower][staker]` entry corresponding to the last vocher becomes incorrect, pointing to the non-existing element at the position equal to the length of the array.

As a result once cancelVouch() be completed, the getLockedStake(), getVouchingAmount(), updateTrust(), cancelVouch() and debtWriteOff() become unavailable for these borrower and staker. For example, `voucherIndexes[borrower][vouchers[borrower][voucherIndex.idx].staker]` will point to `vouchers[borrower].length` and all `vouchers[borrower][index.idx]` operations will revert with `index out of bounds`:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L476-L486

```solidity
    /**
     *  @dev Get staker locked stake for a borrower
     *  @param staker Staker address
     *  @param borrower Borrower address
     *  @return LockedStake
     */
    function getLockedStake(address staker, address borrower) external view returns (uint256) {
        Index memory index = voucherIndexes[borrower][staker];
        if (!index.isSet) return 0;
        return vouchers[borrower][index.idx].locked;
    }
```

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L488-L499

```solidity
    /**
     *  @dev Get vouching amount
     *  @param _staker Staker address
     *  @param borrower Borrower address
     */
    function getVouchingAmount(address _staker, address borrower) external view returns (uint256) {
        Index memory index = voucherIndexes[borrower][_staker];
        Staker memory staker = stakers[_staker];
        if (!index.isSet) return 0;
        uint96 trustAmount = vouchers[borrower][index.idx].trust;
        return trustAmount < staker.stakedAmount ? trustAmount : staker.stakedAmount;
    }
```

If another vouch be added to Bob, instead of unavailability situation it becomes the incorrect vouch info used in the functions above situation. I.e. the array element for that stale index will be present, but the data there be from the newest entry, i.e. it will be available, but incorrect for the old entry.

This way, for example, incorrect `locked` allows to update the `trust` to be less than real `vouch.locked`:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L533-L539

```solidity
        Index memory index = voucherIndexes[borrower][staker];
        if (index.isSet) {
            // Update existing record checking that the new trust amount is
            // not less than the amount of stake currently locked by the borrower
            Vouch storage vouch = vouchers[borrower][index.idx];
            if (trustAmount < vouch.locked) revert TrustAmountLtLocked();
            vouch.trust = trustAmount;
```

As another example using `locked > 0` from the newest entry will prohibit the cancellation of the non-used old one:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L577-L585

```solidity
    function cancelVouch(address staker, address borrower) public onlyMember(msg.sender) whenNotPaused {
        if (staker != msg.sender && borrower != msg.sender) revert AuthFailed();

        Index memory voucherIndex = voucherIndexes[borrower][staker];
        if (!voucherIndex.isSet) revert VoucherNotFound();

        // Check that the locked amount for this vouch is 0
        Vouch memory vouch = vouchers[borrower][voucherIndex.idx];
        if (vouch.locked > 0) revert LockedStakeNonZero();
```

In this case borrower from the old entry can use this to utilized trust that meant to be cancelled by the staker, effectively freezing the corresponding funds.


## Tool used

Manual Review

## Recommendation

Consider adding the update for the elements moved:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L587-L598

```solidity
        // Remove borrower from vouchers array by moving the last item into the position
        // of the index being removed and then poping the last item off the array
        vouchers[borrower][voucherIndex.idx] = vouchers[borrower][vouchers[borrower].length - 1];
+	voucherIndexes[borrower][vouchers[borrower][voucherIndex.idx].staker].idx = voucherIndex.idx;  
        vouchers[borrower].pop();
        delete voucherIndexes[borrower][staker];

        // Remove borrower from vouchee array by moving the last item into the position
        // of the index being removed and then poping the last item off the array
        Index memory voucheeIndex = voucheeIndexes[borrower][staker];
        vouchees[staker][voucheeIndex.idx] = vouchees[staker][vouchees[staker].length - 1];
+	voucheeIndexes[vouchees[staker][voucheeIndex.idx].borrower][staker].idx = voucheeIndex.idx;
        vouchees[staker].pop();
	delete voucheeIndexes[borrower][staker]; 
```
