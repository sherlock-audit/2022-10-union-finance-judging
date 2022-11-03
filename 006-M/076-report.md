hyh

medium

# Priority withdrawal sequence array will grow infinitely over time

## Summary

Withdraw sequence reduction operation isn't present on adapter removal, while new withdraw sequence is required to have the same length as the old one if set directly. I.e. this withdraw sequence length is kept as current invariant, which only increases on each adapter addition, this way growing indefinitely over time.

## Vulnerability Detail

Each `removeAdapter() -> addAdapter()` operation sequence makes `withdrawSeq` array to be `1` item longer, while `moneyMarkets` array preserves its length. As `withdrawSeq` is just an ordering of `moneyMarkets`, this is not desirable, but cannot be directly fixed as administrative setWithdrawSequence() requires the length to be preserved, while additional addAdapter() operations keep the length difference.

As an example, suppose the system operates long enough and now there are `5` adapters, while removeAdapter() was run 50 times since the contract deployment. `withdrawSeq` will have length of `55` and it is impossible to reduce it.

## Impact

All the users end up paying increased gas costs as AssetManager's withdraw() iterates over the full `withdrawSeq` array.

Over time the length of this array will increase to make withdraw() requiring too much gas, making the operation oftentimes economically non-viable. This is temporal funds freeze, i.e. a rational user will have to delay the withdrawal up to the moment when bloated gas cost will become justified, not being able to withdraw funds without an additional loss bigger than an acceptable threshold until then.

With the further growth of the `withdrawSeq`, the block gas limit can be surpassed, making the operation overall forbidden. This is permanent fund freeze impact conditional on system being in production long enough.

## Code Snippet

setWithdrawSequence() requires new sequence array to be the same size as the current `withdrawSeq`:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/asset/AssetManager.sol#L135-L142

```solidity
    /**
     *  @dev Set withdraw sequence
     *  @param newSeq priority sequence of money market indices to be used while withdrawing
     */
    function setWithdrawSequence(uint256[] calldata newSeq) external override onlyAdmin {
        if (newSeq.length != withdrawSeq.length) revert NotParity();
        withdrawSeq = newSeq;
    }
```

removeAdapter() reduces the length of the `moneyMarkets` array, but not of the `withdrawSeq` array:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/asset/AssetManager.sol#L436-L457

```solidity
    /**
     *  @dev Remove a adapter for the underlying lending protocol
     *  @param adapterAddress adapter address
     */
    function removeAdapter(address adapterAddress) external override onlyAdmin {
        bool isExist = false;
        uint256 index;
        uint256 moneyMarketsLength = moneyMarkets.length;

        for (uint256 i = 0; i < moneyMarketsLength; i++) {
            if (adapterAddress == address(moneyMarkets[i])) {
                isExist = true;
                index = i;
                break;
            }
        }

        if (isExist) {
            moneyMarkets[index] = moneyMarkets[moneyMarketsLength - 1];
            moneyMarkets.pop();
        }
    }
```

In the same time addAdapter() increases both `moneyMarkets` and `withdrawSeq` arrays by one when being called with `adapterAddress` that's not already present:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/asset/AssetManager.sol#L416-L434

```solidity
    /**
     *  @dev Add a new adapter for the underlying lending protocol
     *  @param adapterAddress adapter address
     */
    function addAdapter(address adapterAddress) external override onlyAdmin {
        bool isExist = false;
        uint256 moneyMarketsLength = moneyMarkets.length;

        for (uint256 i = 0; i < moneyMarketsLength; i++) {
            if (adapterAddress == address(moneyMarkets[i])) isExist = true;
        }

        if (!isExist) {
            moneyMarkets.push(IMoneyMarketAdapter(adapterAddress));
            withdrawSeq.push(moneyMarkets.length - 1);
        }

        approveAllTokensMax(adapterAddress);
    }
```

withdraw() iterates over the full `withdrawSeq` array each time:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/asset/AssetManager.sol#L345-L359

```solidity
        if (isMarketSupported(token)) {
            uint256 withdrawSeqLength = withdrawSeq.length;
            // iterate markets according to defined sequence and withdraw
            for (uint256 i = 0; i < withdrawSeqLength && remaining > 0; i++) {
                IMoneyMarketAdapter moneyMarket = moneyMarkets[withdrawSeq[i]];
                if (!moneyMarket.supportsToken(token)) continue;

                uint256 supply = moneyMarket.getSupply(token);
                if (supply == 0) continue;

                uint256 withdrawAmount = supply < remaining ? supply : remaining;
                remaining -= withdrawAmount;
                moneyMarket.withdraw(token, account, withdrawAmount);
            }
        }
```

## Tool used

Manual Review

## Recommendation

Consider updating the setWithdrawSequence() logic to require new sequence array to be the same size as `moneyMarkets` whose ordering it represents:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/asset/AssetManager.sol#L135-L142

```solidity
    /**
     *  @dev Set withdraw sequence
     *  @param newSeq priority sequence of money market indices to be used while withdrawing
     */
    function setWithdrawSequence(uint256[] calldata newSeq) external override onlyAdmin {
-       if (newSeq.length != withdrawSeq.length) revert NotParity();
+       if (newSeq.length != moneyMarkets.length) revert NotParity();
        withdrawSeq = newSeq;
    }
```