joestakey

high

# A malicious borrower can never repay, effectively stealing from the protocol

## Summary
In the docs, it is mentioned a `member` can borrow. This is determined by how much DAI a `voucher` decides to trust them to borrow and repay.

The issue is that there is no way for the protocol to force a malicious borrower to repay their debt, meaning members can steal `underlying` without any capital risk.

## Vulnerability Detail
This can be illustrated with an example: 

- Alice becomes a member. She decides to borrow `borrowAmount` of `DAI` by calling `uToken.borrow(ALICE, borrowAmount)`.
- She decides never to repay her debt. This grieves `depositors` of `DAI`, more directly Alice's staker as their stake is currently locked.


This is essentially what happens in this [test](https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/test/foundry/uToken/TestBorrowRepay.t.sol#L134-L152), in a situation where Bob is Alice's Staker and calls `repayBorrow` to unlock his stake, having to repay for the `DAI` Alice stole.

## Impact
Borrowers can grieve stakers, who will be forced to repay the stolen amount.

## Code Snippet
https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/market/UToken.sol#L513-L554

## Tool used
Manual Review, Foundry

## Recommendation
This is by design as members are 'trusted' by stakers, but any form of undercollateralized loan (here with no collateral) means it is essentially possible for malicious `members` to rug others.