Skinny Violet Python

medium

# Claiming rewards might take more gas than Ethereum block gas limit

## Summary

There is a specific case where claiming rewards might cost the user more than 30,000,000 gas (the Ethereum block gas limit), meaning they might never be able to claim rewards. 

## Vulnerability Detail

Here is a snippet from `_claimCvgSdtRewards`:

```solidity
        for (; lastClaimedSdt < actualCycle; ) {
            /// @dev Retrieve the amount staked at the iterated cycle for this Staking position.
            uint256 tokenStaked = _stakedAmountEligibleAtCycle(lastClaimedSdt, tokenId, lengthHistory);
```

First, let's start with `_stakedAmountEligibleAtCycle`:

```solidity
    function _stakedAmountEligibleAtCycle(
        uint256 cycleId,
        uint256 tokenId,
        uint256 lengthHistory
    ) internal view returns (uint256) {
        uint256 i = lengthHistory - 1;
        uint256 historyCycle = _stakingHistoryByToken[tokenId][i];
        /// @dev Finds the cycle of the last first action performed before the {_cycleId}
        while (historyCycle > cycleId) {
            historyCycle = _stakingHistoryByToken[tokenId][i];
            unchecked {
                --i;
            }
        }

        return _tokenInfoByCycle[historyCycle][tokenId].amountStaked;
    }
```

We first iterate through all cycles since the last time that we claimed (or just the first time we staked), and then inside `_stakedAmountEligibleAtCycle` have to again iterate through all the cycles in `_stakingHistoryByToken`, which is just all the staking cycles in which we deposited or withdrew. Let's say that we haven't claimed rewards for an extremely long, but still reasonable time (say 5 - 10 years, which translates to 260 - 520 cycles). We've also been consistently depositing/withdrawing every week during this timeframe, but again not claiming. This becomes extremely expensive in that case, and there's also a SLOAD inside the while loop. 

There is also this inside `_claimCvgSdtRewards`:

```solidity
                    for (uint256 erc20Id; erc20Id < maxLengthRewards; ) {
                        /// @dev Get the ERC20 and the amount distributed on the iterated cycle.
                        ICommonStruct.TokenAmount memory rewardAsset = _sdtRewardsByCycle[lastClaimedSdt][erc20Id + 1];
```

For each cycle since the last time we claimed, we must not only iterate through all possible reward tokens, but also make a separate storage access for each one. This can also get expensive very fast, especially if we haven't claimed in a long time and there are lots of reward tokens. 

This can get even worse when you consider that in `SdtRewardReceiver` we have a `claimMultipleStaking` which should be able to claim for multiple staking contracts and multiple token ids. 

## Impact

Under specific circumstances, user might not be able to claim rewards since it would take more than 30,000,000 gas 

## Code Snippet

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L429-L515

## Tool used

Manual Review

## Recommendation
No need to have two loops (one outer loop over the cycle and then an inner loop to determine the staking amount before the cycle); you can just use a two pointers approach instead. 
