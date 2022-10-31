GimelSec

medium

# The protocol doesn't handle fee-on-transfer tokens

## Summary

The protocol will be broken if supported ERC20 tokens are fee-on-transfer tokens.

## Vulnerability Detail

UserManager and AssetManager doesn't handle fee-on-transfer tokens. We take UserManager as an example to illustrate the fee-on-transfer issue.

Assume `stakingToken` is fee-on-transfer token and the token will take 10% fees. If a user want to `stake(1000)`, the protocol will first transfer tokens to UserManager in [L676](https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L676). Due to the fees, UserManager will only get `900` tokens.

Then UserManager will call `deposit()` of AssetManager in [L680](https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L680), AssetManager will call `poolToken.safeTransferFrom(msg.sender, address(moneyMarket), amount)`. But the amount variable is `1000` and UserManager only has `900` tokens, the transaction will be reverted.

## Impact

The transaction will be reverted if the protocol uses a fee-on-transfer token.
In AssetManager, `deposit()` function will record wrong `totalPrincipal` values, users will not be able to loan because of the wrong value from `getLoanableAmount()`.

## Code Snippet

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L673-L680
https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/asset/AssetManager.sol#L273-L274

## Tool used

Manual Review

## Recommendation

Record `balanceAfter - balanceBefore` rather than using `amount` directly.

```solidity
function retrieveTokens(address sender, uint256 amount) public {
    uint256 balanceBefore = deflationaryToken.balanceOf(address(this));
    deflationaryToken.safeTransferFrom(sender, address(this), amount);
    uint256 balanceAfter = deflationaryToken.balanceOf(address(this));

    amount = balanceAfter.sub(balanceBefore);
}
```
