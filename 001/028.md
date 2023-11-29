Spare Leather Puma

high

# Ys rewards should not be claimable when `cvgCycle() == cycleClaimed`

## Summary
Users may lose out on rewards if they call `claimRewards` when `cvgCycle() == cycleClaimed`

## Vulnerability Detail
Based on the amount of cvg tokens the users have locked and the duration they've locked them for, the users are allocated ys balance. Based on that balance they can claim rewards via the `claimRewards` function within `YsDistributor`. The current requirement to claim the rewards for a cycle is the following: 
```solidity
        uint256 cycleClaimed = tdeId * TDE_DURATION;

        /// @dev Cannot claim a TDE not available yet.
        require(_cvgControlTower.cvgCycle() >= cycleClaimed, "NOT_AVAILABLE");
```

However, this logic is faulty as it allows for claiming rewards when `cvgCycle == cycleClaimed`, which is faulty as rewards may not yet be finalized. 
If a user calls it they will claim the rewards for the said cycle. However, if we look at the `depositMultipleToken` function above, we will see that the next call (happening within the same cycle) will distribute rewards towards this same cycle. Any users who have already called `claimRewards` will be locked out of these rewards and the rewards will be lost/ stuck within the contract forever.
```solidity
    function depositMultipleToken(TokenAmount[] calldata deposits) external onlyTreasuryBonds {
        uint256 _actualCycle = cvgControlTower.cvgCycle();
        uint256 _actualTDE = _actualCycle % TDE_DURATION == 0 // @audit - if cvgCycle == cycleClaimed, then _actualCycle % TDE_DURATION == 0
            ? _actualCycle / TDE_DURATION  // @audit - this value will be used 
            : (_actualCycle / TDE_DURATION) + 1;

        address[] memory _tokens = depositedTokenAddressForTde[_actualTDE];
        uint256 tokensLength = _tokens.length;

        for (uint256 i; i < deposits.length; ) {
            IERC20 _token = deposits[i].token;
            uint256 _amount = deposits[i].amount;

            depositedTokenAmountForTde[_actualTDE][_token] += _amount;
```

## Impact
Loss of rewards

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/YsDistributor.sol#L101C1-L105C49
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/YsDistributor.sol#L176

## Tool used

Manual Review

## Recommendation
Change the `>=` in the require statement to `>`
 ```solidity
-        require(_cvgControlTower.cvgCycle() >= cycleClaimed, "NOT_AVAILABLE");

+        require(_cvgControlTower.cvgCycle() > cycleClaimed, "NOT_AVAILABLE");
```