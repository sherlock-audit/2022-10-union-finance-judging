hyh

medium

# Maximal approvals remain for the AssetManager's adapters and tokens after removal

## Summary

While adding an adapter and a token to the AssetManager the system provides for the unlimited approvals for the current list of tokens and markets correspondingly, while the removal of the adapter and the token does not clear these approvals, which can be exploited thereafter as the contract balance does hold intermediary funds.

## Vulnerability Detail

If the adapter or the token are being removed due to it being found to be malicious or contain a critical issue, treating downstream systems, currently there is no way to remove the exposure to it. This allowance can be exploited to drain AssetManager's balance.

That's a long term threat as market adapter or token being removed keep unlimited allowances forever as there is no mechanics to remove or limit those. I.e. some critical vulnerability can be found maybe substantially later in some adapter that was used by AssetManager and was removed long time ago. Its allowances remain and can be exploited to extract any funds held in the tokens from the list that was actual back then.

## Impact

A malicious token or adapter can steal all the approved token's balances of the AssetManager. Some assets are routinely residing on the AssetManager's balance: as an example, when the funds, being deposited, can't be placed to the markets right away, they are left on the contract balance until next admin's rebalance() call.

An attacker can setup a bot that tracks this balance and exploit some vulnerability in an already removed adapter to fully drain the balance. Setting the severity to be **medium** due to the preconditions described.

## Code Snippet

removeAdapter() leaves all the infinite approvals:

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

Similarly, removeToken() leaves all markets' unlimited approvals with the token intact:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/asset/AssetManager.sol#L392-L414

```solidity
    /**
     *  @dev Remove a ERC20 token to support in AssetManager
     *  @param tokenAddress ERC20 token address
     */
    function removeToken(address tokenAddress) external override onlyAdmin {
        bool isExist = false;
        uint256 index;
        uint256 supportedTokensLength = supportedTokensList.length;

        for (uint256 i = 0; i < supportedTokensLength; i++) {
            if (tokenAddress == address(supportedTokensList[i])) {
                isExist = true;
                index = i;
                break;
            }
        }

        if (isExist) {
            supportedTokensList[index] = supportedTokensList[supportedTokensLength - 1];
            supportedTokensList.pop();
            supportedMarkets[tokenAddress] = false;
        }
    }
```

AssetManager's balance aren't necessary empty as some funds are left until manual placement:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/asset/AssetManager.sol#L261-L316

```solidity
    function deposit(address token, uint256 amount)
        external
        override
        whenNotPaused
        onlyAuth(token)
        nonReentrant
        returns (bool)
    {
        IERC20Upgradeable poolToken = IERC20Upgradeable(token);
        if (amount == 0) revert AmountZero();

        if (!_isUToken(msg.sender, token)) {
            balances[msg.sender][token] += amount;
            totalPrincipal[token] += amount;
        }

        bool remaining = true;
        if (isMarketSupported(token)) {
			...
        }

        if (remaining) {
            poolToken.safeTransferFrom(msg.sender, address(this), amount);
        }

        emit LogDeposit(token, msg.sender, amount);
```

## Tool used

Manual Review

## Recommendation

Consider introducing approvals clearing function and run it on adapter removal, for example:

```solidity
+   /**
+    *  @dev Removal of the allowances for all underlying tokens
+    *  @param adapterAddress Address of adapter being removed
+    */
+   function removeApprovals(address adapterAddress) internal override {
+       uint256 supportedTokensLength = supportedTokensList.length;
+       for (uint256 i = 0; i < supportedTokensLength; ++i) {
+           IERC20Upgradeable poolToken = IERC20Upgradeable(supportedTokensList[i]);
+           poolToken.safeApprove(adapterAddress, 0);
+       }
+   }
```

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
+           removeApprovals(adapterAddress);
        }
    }
```

The same can be done for token removal, for example:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/asset/AssetManager.sol#L392-L414

```solidity
    /**
     *  @dev Remove a ERC20 token to support in AssetManager
     *  @param tokenAddress ERC20 token address
     */
    function removeToken(address tokenAddress) external override onlyAdmin {
        bool isExist = false;
        uint256 index;
        uint256 supportedTokensLength = supportedTokensList.length;

        for (uint256 i = 0; i < supportedTokensLength; i++) {
            if (tokenAddress == address(supportedTokensList[i])) {
                isExist = true;
                index = i;
                break;
            }
        }

        if (isExist) {
            supportedTokensList[index] = supportedTokensList[supportedTokensLength - 1];
            supportedTokensList.pop();
            supportedMarkets[tokenAddress] = false;
+           removeTokenApprovals(tokenAddress);            
        }
    }
```

There new removeTokenApprovals() function similarly to approveAllMarketsMax() cycles across all available markets, setting `IERC20Upgradeable(tokenAddress).safeApprove(address(moneyMarkets[i]), 0)`.