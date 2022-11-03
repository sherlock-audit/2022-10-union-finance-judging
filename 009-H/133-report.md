hyh

high

# repayBorrow is inaccessible by overdue borrowers

## Summary

User facing UToken's `repayBorrow() -> _repayBorrowFresh()` will be reverted when borrower is overdue, making it permanently impossible to pay the debt within the system.

## Vulnerability Detail

UToken's _repayBorrowFresh() will attempt to update frozen info when borrower is overdue, but the corresponding function of UserManager, updateFrozenInfo(), is `onlyComptroller`, so the call will be reverted.

As repayBorrow() with payment exceeding the total interest is the only way for a borrower to repay their loan either partially or fully, for the lenders of this borrower this means that the debt will never be repaid as there is no technical possibility to do it. I.e. all notional of all debts whose borrowers are overdue are permanently frozen.

Such borrowers are stuck with the overdue status and can only pay partial interest payments.

## Impact

The total impact is permanent fund freeze for the lenders of any overdue borrowers.

The only prerequisite is borrower turning overdue, which is a pretty common business case, so setting the severity to be **high**.

## Code Snippet

_repayBorrowFresh() calls `IUserManager(userManager).updateFrozenInfo(borrower, 0)` when `checkIsOverdue(borrower)` is true:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/market/UToken.sol#L621-L625

```solidity
            if (isOverdue) {
                // For borrowers that are paying back overdue balances we need to update their
                // frozen balance and the global total frozen balance on the UserManager
                IUserManager(userManager).updateFrozenInfo(borrower, 0);
            }
```

updateFrozenInfo() is `onlyComptroller`:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L881-L883

```solidity
    function updateFrozenInfo(address staker, uint256 pastBlocks) external onlyComptroller returns (uint256, uint256) {
        return _updateFrozen(staker, pastBlocks);
    }
```

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L305-L308

```solidity
    modifier onlyComptroller() {
        if (address(comptroller) != msg.sender) revert AuthFailed();
        _;
    }
```

This means that any overdue borrower is stuck, being unable neither to remove overdue status, nor to repay the debt as it is the very same piece of logic, that reverts whenever called given that borrower is overdue:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/market/UToken.sol#L573-L626

```solidity
    function _repayBorrowFresh(
        address payer,
        address borrower,
        uint256 amount
    ) internal {
        if (!accrueInterest()) revert AccrueInterestFailed();

        uint256 interest = calculatingInterest(borrower);
        uint256 borrowedAmount = borrowBalanceStoredInternal(borrower);
        uint256 repayAmount = amount > borrowedAmount ? borrowedAmount : amount;
        if (repayAmount == 0) revert AmountZero();

        uint256 toReserveAmount;
        uint256 toRedeemableAmount;

        if (repayAmount >= interest) {
            // If the repayment amount is greater than the interest (min payment)
            bool isOverdue = checkIsOverdue(borrower);

            // Interest is split between the reserves and the uToken minters based on
            // the reserveFactorMantissa When set to WAD all the interest is paid to teh reserves.
            // any interest that isn't sent to the reserves is added to the redeemable amount
            // and can be redeemed by uToken minters.
            toReserveAmount = (interest * reserveFactorMantissa) / WAD;
            toRedeemableAmount = interest - toReserveAmount;

            // Update the total borrows to reduce by the amount of principal that has
            // been paid off
            totalBorrows -= (repayAmount - interest);

            // Update the account borrows to reflect the repayment
            accountBorrows[borrower].principal = borrowedAmount - repayAmount;
            accountBorrows[borrower].interest = 0;

            if (getBorrowed(borrower) == 0) {
                // If the principal is now 0 we can reset the last repaid block to 0.
                // which indicates that the borrower has no outstanding loans.
                accountBorrows[borrower].lastRepay = 0;
            } else {
                // Save the current block number as last repaid
                accountBorrows[borrower].lastRepay = getBlockNumber();
            }

            // Call update locked on the userManager to lock this borrowers stakers. This function
            // will revert if the account does not have enough vouchers to cover the repay amount. ie
            // the borrower is trying to repay more than is locked (owed)
            IUserManager(userManager).updateLocked(borrower, uint96(repayAmount - interest), false);

            if (isOverdue) {
                // For borrowers that are paying back overdue balances we need to update their
                // frozen balance and the global total frozen balance on the UserManager
                IUserManager(userManager).updateFrozenInfo(borrower, 0);
            }
        }
```

I.e. once borrower is overdue the only way to remove that status is to run `accountBorrows[borrower].lastRepay = getBlockNumber()` line of _repayBorrowFresh(), as there are no other ways to do that in the system.

In the same time when the borrower is overdue the code will always revert on `IUserManager(userManager).updateFrozenInfo(borrower, 0)` call as it's `onlyComptroller`.

## Tool used

Manual Review

## Recommendation

Consider updating it to be either UToken or Comptroller as both contracts use it:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L881-L883

```solidity
-   function updateFrozenInfo(address staker, uint256 pastBlocks) external onlyComptroller returns (uint256, uint256) {
+   function updateFrozenInfo(address staker, uint256 pastBlocks) external returns (uint256, uint256) {
+		if (address(uToken) != msg.sender || address(comptroller) != msg.sender) revert AuthFailed();
        return _updateFrozen(staker, pastBlocks);
    }
```