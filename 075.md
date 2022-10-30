hansfriese

medium

# `AssetManager.rebalance()` will revert when the balance of `tokenAddress` in the money market is 0.

## Summary
`AssetManager.rebalance()` will revert when the balance of `tokenAddress` in the money market is 0.

## Vulnerability Detail
[AssetManager.rebalance()](https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/asset/AssetManager.sol#L497) tries to withdraw tokens from each money market for rebalancing [here](https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/asset/AssetManager.sol#L510-L518).

```solidity
    // Loop through each money market and withdraw all the tokens
    for (uint256 i = 0; i < moneyMarketsLength; i++) {
        IMoneyMarketAdapter moneyMarket = moneyMarkets[i];
        if (!moneyMarket.supportsToken(tokenAddress)) continue;
        moneyMarket.withdrawAll(tokenAddress, address(this));

        supportedMoneyMarkets[supportedMoneyMarketsSize] = moneyMarket;
        supportedMoneyMarketsSize++;
    }
```

When the balance of the `tokenAddress` is 0, we don't need to call `moneyMarket.withdrawAll()` but it still tries to call.

But this will revert because Aave V3 doesn't allow to withdraw 0 amount [here](https://github.com/aave/aave-v3-core/blob/master/contracts/protocol/libraries/logic/ValidationLogic.sol#L87-L92).

```solidity
  function validateWithdraw(
    DataTypes.ReserveCache memory reserveCache,
    uint256 amount,
    uint256 userBalance
  ) internal pure {
    require(amount != 0, Errors.INVALID_AMOUNT);
```

So `AssetManager.rebalance()` will revert if one money market has zero balance of `tokenAddress`.

## Impact
The money markets can't be rebalanced if there is no balance in at least one market.

## Code Snippet
https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/asset/AssetManager.sol#L514

## Tool used
Manual Review

## Recommendation
I think we can modify [AaveV3Adapter.withdrawAll()](https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/asset/AaveV3Adapter.sol#L226) to work only when the balance is positive.

```solidity
    function withdrawAll(address tokenAddress, address recipient)
        external
        override
        onlyAssetManager
        checkTokenSupported(tokenAddress)
    {
        address aTokenAddress = tokenToAToken[tokenAddress];
        IERC20Upgradeable aToken = IERC20Upgradeable(aTokenAddress);
        uint256 balance = aToken.balanceOf(address(this));

        if (balance > 0) {
            lendingPool.withdraw(tokenAddress, type(uint256).max, recipient);
        }
    }
```