hyh

medium

# It's impossible to writing off any vouch fully for an outside actor

## Summary

UserManager's debtWriteOff() being called by a non-staker/borrower (after all overdue period thresholds pass) to remove bad debt will revert if the corresponding voucher is to be written off fully, i.e. has no trust left after the write-off.

In other words public debt writing off reverts in the full vouch removal case.

## Vulnerability Detail

debtWriteOff() becomes public after all overdue delays pass, but when vouch write off is to be full, it calls vouch cancellation function, which aren't accommodated for it and reverts in this case. The reason is the more restrictive access controls in cancelVouch(), which do not utilize additional `available by the public if the loan is overdue` logic of debtWriteOff().

As a workaround debtWriteOff() can be called with not full amount, leaving dust in the vouch, so almost all amount itself can be written off from the system. However, such vouches will end up open forever as long as the corresponding stakers/borrowers do not act.

## Impact

Bad debt vouchers cannot be fully removed if staker/borrower isn't active, i.e. the immediate impact is unavailability of such public debt write-off.

Over time such bad debt vouchers with dust amounts will pile up, increasing operational costs for the functions that go through all existing vouches: UserManager's getCreditLimit() and getFrozenInfo(), UToken's borrow() and repayBorrow() via UserManager's updateLocked(). Over time this gas cost increase will apply to the substantial part of system users (i.e. as user link coverage grows and time passes the probability of a user affected by such zombie bad debt vouchers will slowly rise as well). 

Long-term total impact can be up to the permanent funds freeze due to gas limit breaching and locking of the mentioned functions.

## Code Snippet

debtWriteOff() is public when `block.number > lastRepay + overdueBlocks + maxOverdueBlocks`, but calls cancelVouch() when `vouch.trust == 0`:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L717-L778

```solidity
    /**
     *  @notice Write off a borrowers debt
     *  @dev    Used the stakers locked stake to write off the loan, transfering the
     *          Stake to the AssetManager and adjusting balances in the AssetManager
     *          and the UToken to repay the principal
     *  @dev    Emits {LogDebtWriteOff} event
     *  @param borrower address of borrower
     *  @param amount amount to writeoff
     */
    function debtWriteOff(
        address staker,
        address borrower,
        uint96 amount
    ) external {
        if (amount == 0) revert AmountZero();
        uint256 overdueBlocks = uToken.overdueBlocks();
        uint256 lastRepay = uToken.getLastRepay(borrower);

        // This function is only callable by the public if the loan is overdue by
        // overdue blocks + maxOverdueBlocks. This stops the system being left with
        // debt that is overdue indefinitely and no ability to do anything about it.
        if (block.number <= lastRepay + overdueBlocks + maxOverdueBlocks) {
            if (staker != msg.sender) revert AuthFailed();
        }

		...
        // update vouch trust amount
        vouch.trust -= amount;
        vouch.locked -= amount;
        ...

        if (vouch.trust == 0) {
            cancelVouch(staker, borrower);
        }
```

cancelVouch() allows `msg.sender` to be `borrower` or `staker` only:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L567-L578

```solidity
    /**
     *  @dev Remove voucher for memeber
     *  Can be called by either the borrower or the staker. It will remove the voucher from
     *  the voucher array by replacing it with the last item of the array and reseting the array
     *  size to -1 by poping off the last item
     *  Only callable by a member when the contract is not paused
     *  Emit {LogCancelVouch} event
     *  @param staker Staker address
     *  @param borrower borrower address
     */
    function cancelVouch(address staker, address borrower) public onlyMember(msg.sender) whenNotPaused {
        if (staker != msg.sender && borrower != msg.sender) revert AuthFailed();
```

This will fail debtWriteOff() calls from any third parties, forcing them to leave dust in `vouch.trust`, choosing `amount = vouch.trust - dust` even when a vouch to be removed fully, which looks to be the frequent use case (bad debtors tend to fully utilize their credit lines).

## Tool used

Manual Review

## Recommendation

Consider the following, as an example:

Update cancelVouch() to allow for calls from self:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L577-L578

```solidity
+   function cancelVouch(address staker, address borrower) public whenNotPaused {
+		bool selfCall = address(this) == msg.sender;
+       if ((!checkIsMember(msg.sender) && !selfCall) || (staker != msg.sender && borrower != msg.sender && !selfCall)) revert AuthFailed();
-   function cancelVouch(address staker, address borrower) public onlyMember(msg.sender) whenNotPaused {
-       if (staker != msg.sender && borrower != msg.sender) revert AuthFailed();
```

Call it from itself in debtWriteOff() as a special case:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L776-L778

```solidity
        if (vouch.trust == 0) {
+           IUserManager(address(this)).cancelVouch(staker, borrower);
-           cancelVouch(staker, borrower);
        }
```