ctf_sec

medium

# Unsafe downcasting arithmetic operation in UserManager related contract and in UToken.sol

## Summary

The value is unsafely downcasted and truncated from uint256 to uint96 or uint128 in UserManager related contract and in UToken.sol.

## Vulnerability Detail

value can unsafely downcasted. let us look at it cast by cast.

In UserManagerDAI.sol 

```solidity
  function stakeWithPermit(
      uint256 amount,
      uint256 nonce,
      uint256 expiry,
      uint8 v,
      bytes32 r,
      bytes32 s
  ) external whenNotPaused {
      IDai erc20Token = IDai(stakingToken);
      erc20Token.permit(msg.sender, address(this), nonce, expiry, true, v, r, s);

      stake(uint96(amount));
  }
```
as we can see, the user's staking amount is downcasted from uint256 to uint96.

the same issue exists in UserManagerERC20.sol

In the context of UToken.sol, a bigger issue comes.

User invokes the borrow function in UToken.sol

```solidity
   function borrow(address to, uint256 amount) external override onlyMember(msg.sender) whenNotPaused nonReentrant {
```

and

```solidity
  // Withdraw the borrowed amount of tokens from the assetManager and send them to the borrower
  if (!assetManagerContract.withdraw(underlying, to, amount)) revert WithdrawFailed();

  // Call update locked on the userManager to lock this borrowers stakers. This function
  // will revert if the account does not have enough vouchers to cover the borrow amount. ie
  // the borrower is trying to borrow more than is able to be underwritten
  IUserManager(userManager).updateLocked(msg.sender, uint96(amount + fee), true);
```

note when we withdraw fund from asset Manager, we use a uint256 amount, but we downcast it to uint96(amount + fee) when updating the locked. The accounting would be so broken if the amount + fee is a larger than uint96 number.

Same issue in the function UToken.sol# _repayBorrowFresh

```solidity
    function _repayBorrowFresh(
        address payer,
        address borrower,
        uint256 amount
    ) internal {
```

and 

```solidity
  // Update the account borrows to reflect the repayment
  accountBorrows[borrower].principal = borrowedAmount - repayAmount;
  accountBorrows[borrower].interest = 0;
```

and

```sollidity
 IUserManager(userManager).updateLocked(borrower, uint96(repayAmount - interest), false);
```

we use a uint256 number for borrowedAmount - repayAmount, but downcast it to uint96(repayAmount - interest) when updating the lock!

Note there are index-related downcasting, the damage is small , comparing the accounting related downcasting.because it is difference to have uint128 amount of vouch, but I still want to mention it: the index is unsafely downcasted from uint256 to uint128

```solidity
  // Get the new index that this vouch is going to be inserted at
  // Then update the voucher indexes for this borrower as well as
  // Adding the Vouch the the vouchers array for this staker
  uint256 voucherIndex = vouchers[borrower].length;
  voucherIndexes[borrower][staker] = Index(true, uint128(voucherIndex));
  vouchers[borrower].push(Vouch(staker, trustAmount, 0, 0));

  // Add the voucherIndex of this new vouch to the vouchees array for this
  // staker then update the voucheeIndexes with the voucheeIndex
  uint256 voucheeIndex = voucheesLength;
  vouchees[staker].push(Vouchee(borrower, uint96(voucherIndex)));
  voucheeIndexes[borrower][staker] = Index(true, uint128(voucheeIndex));
```

There are block.number related downcasting, which is a smaller issue.

```solidity
vouch.lastUpdated = uint64(block.number);
```

## Impact

The damage level from the number truncation is rated by:

UToken borrow and repaying downcasting > staking amount downcating truncation > the vouch index related downcasting. > block.number casting.

## Code Snippet

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L550-L563

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManagerDAI.sol#L28

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManagerERC20.sol#L26

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/market/UToken.sol#L552

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/market/UToken.sol#L619

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L827

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L840

## Tool used

Manual Review

## Recommendation

Just use uint256, or use openzepplin safeCasting.

https://docs.openzeppelin.com/contracts/3.x/api/utils#SafeCast