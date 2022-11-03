hyh

medium

# Partial withdrawals by AssetManager lead to user funds freeze

## Summary

AssetManager's withdraw() doesn't guarantee the retrieval of the full requested amount. However, all dependant function always treat such withdrawal call as if full amount was successfully sent to a recipient.

## Vulnerability Detail

Partial withdrawals by AssetManager are unaccounted in all withdraw initiating functions: user-facing UserManager's unstake(), UToken's borrow() and redeem(). This way the `remaining` amount withdraw() failed to obtain from the adapters is permanently lost for the withdrawal recipient as full amount is accounted each time.

Strategies AssetManager utilize via adapters can have temporal funds unavailability, say Aave and Compound can have liquidity squeezes. If a particular lending pool has liquidity shortage, i.e. almost all underlying is lent out, full withdrawal of the requested underlying token amount will not be possible at the moment. It doesn't mean the funds are lost, so writing the full amount off for a recipient isn't correct and is equivalent to user's fund freeze in the system.

## Impact

Net impact is permanent fund freeze for the users who were recipients for such withdrawals. I.e. when `remaining` funds become available later, there is no way to receive them for the users as their accounting was updated by full amounts already. This way such remaining funds were de facto wrote down for the users and became excess unallocated funds of a Strategy, i.e. profit for the system from AssetManager's funds allocation.

As this permanent fund freeze is conditional on Strategy liquidity squeeze, which is medium probability event, being a part of normal activity for lending protocols, setting the severity to be **medium**.

## Code Snippet

Accounting discrepancy is introduced each time as AssetManager's withdraw() do not guarantee full `amount` retrieval, always returns true and reduces the balances by `amount - remaining`:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/asset/AssetManager.sol#L328-L369

```solidity
    function withdraw(
        address token,
        address account,
        uint256 amount
    ) external override whenNotPaused nonReentrant onlyAuth(token) returns (bool) {
        if (!_checkSenderBalance(msg.sender, token, amount)) revert InsufficientBalance();

        uint256 remaining = amount;

        // If there are tokens in Asset Manager then transfer them on priority
        uint256 selfBalance = IERC20Upgradeable(token).balanceOf(address(this));
        if (selfBalance > 0) {
            uint256 withdrawAmount = selfBalance < remaining ? selfBalance : remaining;
            remaining -= withdrawAmount;
            IERC20Upgradeable(token).safeTransfer(account, withdrawAmount);
        }

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

        if (!_isUToken(msg.sender, token)) {
            balances[msg.sender][token] = balances[msg.sender][token] - amount + remaining;
            totalPrincipal[token] = totalPrincipal[token] - amount + remaining;
        }

        emit LogWithdraw(token, account, amount, remaining);

        return true;
    }
```

The functions that use withdraw() treat it differently, always assuming that the whole amount is successfully retrieved.

I.e. UserManager's balance in AssetManager will be reduced less than user's balance in UserManager as UserManager's unstake() always removes the full `amount` from `staker.stakedAmount` and `totalStaked`:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol#L691-L705

```solidity
    function unstake(uint96 amount) external whenNotPaused nonReentrant {
        Staker storage staker = stakers[msg.sender];

        // Stakers can only unstaked stake balance that is unlocked. Stake balance
        // becomes locked when it is used to underwrite a borrow.
        if (staker.stakedAmount - staker.locked < amount) revert InsufficientBalance();

        comptroller.withdrawRewards(msg.sender, stakingToken);

        staker.stakedAmount -= amount;
        totalStaked -= amount;

        if (!IAssetManager(assetManager).withdraw(stakingToken, msg.sender, amount)) {
            revert AssetManagerWithdrawFailed();
        }
```

UToken's borrow() also always assumes that full `amount` is retrieved to the borrower, adding full `amount` to `accountBorrows` entry:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/market/UToken.sol#L512-L555

```solidity
    function borrow(address to, uint256 amount) external override onlyMember(msg.sender) whenNotPaused nonReentrant {
        IAssetManager assetManagerContract = IAssetManager(assetManager);
        if (amount < minBorrow) revert AmountLessMinBorrow();
        if (amount > getRemainingDebtCeiling()) revert AmountExceedGlobalMax();

        ...

        uint256 accountBorrowsNew = borrowedAmount + amount + fee;
        uint256 totalBorrowsNew = totalBorrows + amount + fee;

        // Update internal balances
        accountBorrows[msg.sender].principal += amount + fee;

        ...

        // Withdraw the borrowed amount of tokens from the assetManager and send them to the borrower
        if (!assetManagerContract.withdraw(underlying, to, amount)) revert WithdrawFailed();

        // Call update locked on the userManager to lock this borrowers stakers. This function
        // will revert if the account does not have enough vouchers to cover the borrow amount. ie
        // the borrower is trying to borrow more than is able to be underwritten
        IUserManager(userManager).updateLocked(msg.sender, uint96(amount + fee), true);

        emit LogBorrow(msg.sender, to, amount, fee);
    }
```

UToken's redeem() similarly always burns full `uTokenAmount` corresponding to `underlyingAmount` requested from AssetManager:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/market/UToken.sol#L707-L740

```solidity
    function redeem(uint256 amountIn, uint256 amountOut) external override whenNotPaused nonReentrant {
        if (!accrueInterest()) revert AccrueInterestFailed();
        if (amountIn != 0 && amountOut != 0) revert AmountZero();

        uint256 exchangeRate = exchangeRateStored();

        // Amount of the uToken to burn
        uint256 uTokenAmount;

        // Amount of the underlying token to redeem
        uint256 underlyingAmount;

        if (amountIn > 0) {
            // We calculate the exchange rate and the amount of underlying to be redeemed:
            // uTokenAmount = amountIn
            // underlyingAmount = amountIn x exchangeRateCurrent
            uTokenAmount = amountIn;
            underlyingAmount = (amountIn * exchangeRate) / WAD;
        } else {
            // We get the current exchange rate and calculate the amount to be redeemed:
            // uTokenAmount = amountOut / exchangeRate
            // underlyingAmount = amountOut
            uTokenAmount = (amountOut * WAD) / exchangeRate;
            underlyingAmount = amountOut;
        }

        totalRedeemable -= underlyingAmount;
        _burn(msg.sender, uTokenAmount);

        IAssetManager assetManagerContract = IAssetManager(assetManager);
        if (!assetManagerContract.withdraw(underlying, msg.sender, underlyingAmount)) revert WithdrawFailed();

        emit LogRedeem(msg.sender, amountIn, amountOut, underlyingAmount);
    }
```

Same approach is in administrative removeReserves():

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/market/UToken.sol#L770-L784

```solidity
    function removeReserves(address receiver, uint256 reduceAmount)
        external
        override
        whenNotPaused
        nonReentrant
        onlyAdmin
    {
        if (!accrueInterest()) revert AccrueInterestFailed();

        totalReserves -= reduceAmount;

        if (!IAssetManager(assetManager).withdraw(underlying, receiver, reduceAmount)) revert WithdrawFailed();

        emit LogReservesReduced(receiver, reduceAmount, totalReserves);
    }
```


## Tool used

Manual Review

## Recommendation

Consider:

* either accounting for the actual amount withdrawn, i.e. return actual retrieved amount from withdraw(), move `IAssetManager(assetManager).withdraw` before other logic and operate this amount returned by withdraw() instead of `amount` initially requested in all the accounting logics above,

* or just require in withdraw() that actual amount withdrawn be equal to the requested one.

For the second option as fee on transfer tokens aren't in the main scope for credit lines functionality UNION covers, such a requirement will mean that if the `amount` is lacking the corresponding logic needs to be run again with a smaller one. But only amount actually withdrawn be accounted for, so the discrepancies be avoided:

https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/asset/AssetManager.sol#L366-L369

```solidity
        emit LogWithdraw(token, account, amount, remaining);


+       return remaining < (dustThreshold * amount) / WAD;
-       return true;
    }
```