Jeiwan

high

# Increased reward token inflation due to double counting of `totalFrozen`

## Summary
Increased reward token inflation due to double counting of `totalFrozen`
## Vulnerability Detail
In Union Protocol, [stakers receive reward in UNION token](https://docs.union.finance/protocol-overview/plain-english-overview#stakers-earning-union-from-comptroller). The token is emitted at a certain rate set by the governance and is sent to the [Comptroller](https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/token/Comptroller.sol#L20) contract. Stakers can call the [withdrawRewards](https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/token/Comptroller.sol#L231) function of Comptroller to get their share of UNION. Staker's share is determined based on:
1. [individual staker's activity](https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/token/Comptroller.sol#L240-L245);
1. [global state of a UserManager contract](https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/token/Comptroller.sol#L247-L248).

The latter includes: the total amount of staked tokens and the total amount of frozen tokens. As you can see, the [_getUserManagerState](https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/token/Comptroller.sol#L306) function subtracts `totalFrozen` from `totalStaked` to get the effective staked amount:
```solidity
function _getUserManagerState(IUserManager userManager) internal view returns (UserManagerState memory) {
    UserManagerState memory userManagerState;

    userManagerState.totalFrozen = userManager.totalFrozen();
    userManagerState.totalStaked = userManager.totalStaked() - userManagerState.totalFrozen;
    if (userManagerState.totalStaked < 1e18) {
        userManagerState.totalStaked = 1e18;
    }

    return userManagerState;
}
```

Later in the [withdrawRewards](https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/token/Comptroller.sol#L231) function, `totalFrozen` is subtracted [once again](https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/token/Comptroller.sol#L260) before updating the global inflation index:
```solidity
// update the global states
uint256 totalStaked_ = userManagerState.totalStaked - userManagerState.totalFrozen;
gInflationIndex = _getInflationIndexNew(totalStaked_, block.number - gLastUpdatedBlock);
```
## Impact
Since `totalFrozen` is subtracted twice, the effective amount of staked tokens will be lower than in reality. As a result, [inflation per block](https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/token/Comptroller.sol#L480) will be higher (`effectiveTotalStaked` will be further away from `halfDecayPoint`) and the [division by effectiveAmount](https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/token/Comptroller.sol#L520) will result in a bigger reward per effective staked amount.
## Code Snippet
See Vulnerability Detail.
## Tool used
Manual Review
## Recommendation
Short term, subtract `totalFrozen` amount only once. Long term, notice that subtracting total frozen amount of tokens at least once always increases the inflation index because the total staked amounts becomes lower (and further away from the decay point)–this means that stakers might be incentivized to borrow from themselves (via intermediary accounts) and overdue debts to reduce the effective staked amount. 
