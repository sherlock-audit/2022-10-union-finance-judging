Bnke0x0

medium

# repay modules can fail on zero amount transfers if payer fee is set to zero

## Summary
payer fee can be zero, while repay modules do attempt to send it in such a case anyway as there is no check in place. Some ERC20 tokens do not allow zero-value transfers, reverting such attempts.



## Vulnerability Detail

## Impact
repay modules can fail on zero amount transfers if payer fee is set to zero
## Code Snippet
https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/market/UToken.sol#L642

     'IERC20Upgradeable(underlying).safeTransferFrom(payer, address(this), repayAmount);'

## Tool used

Manual Review

## Recommendation
Consider checking the treasury fee amount and do transfer only when it is positive.
Now:

      'IERC20Upgradeable(underlying).safeTransferFrom(payer, address(this), repayAmount);'

To be:

     'if (repayAmount > 0) {
          IERC20Upgradeable(underlying).safeTransferFrom(payer, address(this), 
           repayAmount);'
