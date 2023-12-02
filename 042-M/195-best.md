Skinny Violet Python

high

# Protocol doesn't account for the fact that bribe rewards can be provided in SDT instead of sdTKN

## Summary

Right now, it seems that the protocol assumes that all bribe rewards will be provided in sdTKN when in fact there are cases where bribe rewards will be provided in SDT instead. This breaks large parts of the protocol. 

## Vulnerability Detail

From the Stake DAO docs (https://stakedao.gitbook.io/stakedaohq/platform/liquid-lockers/vote-incentives):

`
Vote incentives are distributed in SDT unless the peg protection mode is activated (sdTKN/TKN peg below 0.99) in which case vote incentives are distributed in sdTKN.
`

Clearly it is not an option to set the bribe assets to only be sdTKN, as we will lose out on SDT rewards when peg protection mode is not activated. 

However, including `SDT` among the bribe assets breaks how the `SdtBlackHole` works. The `SdtBlackHole` allows a staking service to set which bribe assets should be sent to it's buffer pair when `pullSdStakingBribes` in `SdtBlackHole` is called (of course the origin of this would be `processSdtRewards` in `SdtStakingPositionService`, when we try to gather gauge and bribe rewards). Of course, in the current model, these bribe assets must be disjoint; if multiple staking services have the same bribe asset, then the black hole cannot determine which staking service the bribe asset should be sent to (and consequently it will just be sent to the first one that calls `processSdtRewards` / `pullSdStakingBribes`). This is because `pullSdStakingBribes` looks like this:

```solidity
        /// @dev iterates over all bribes token retrieved
        for (uint256 i; i < bribeTokens.length; ) {
            /// @dev get the balance in the bribe token
            IERC20 bribeToken = bribeTokens[i];
            uint256 balance = bribeToken.balanceOf(address(this));

            if (balance != 0) {
                /// @dev send rewards to claimer
                uint256 claimerRewards = (balance * _processorRewardsPercentage) / 100_000;
                if (claimerRewards > 0) {
                    bribeToken.transfer(_processor, claimerRewards);
                    balance -= claimerRewards;
                }

                /// @dev send the balance of the bribe token minus claimer rewards to the buffer
                bribeToken.transfer(sdtRewardReceiver, balance);

                _bribeTokensAmounts[i] = ICommonStruct.TokenAmount({token: bribeTokens[i], amount: balance});
            }
            unchecked {
                ++i;
            }
        }
```

Notice the `uint256 balance = bribeToken.balanceOf(address(this));`

Let's say that we end up setting SDT as a possible bribe asset on all staking services with `setBribeTokens`. Here's an attack that exploits this:

1. Attacker first finds a staking service that doesn't have much staked, and quickly stakes to take up almost 100% of the staking share in that staking service
2. Attacker manipulates sdTKN/TKN peg to be above 0.99 for the sdTKN of another staking service, so that bribe rewards for this staking service are distributed in SDT
3. After bribe rewards for the other service are distributed, attacker calls `processSdtRewards` from the staking service where he has majority staking share. He will end up taking the bribe reward SDT of the other staking service. 

## Impact

Protocol just doesn't distribute bribe rewards correctly between staking services, leading to loss of bribe rewards for many users. Attacker can also potentially steal SDT rewards. 

## Code Snippet

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBlackHole.sol#L96-L134

## Tool used

Manual Review

## Recommendation

You can have a special case for SDT, where after fetching from the `MultiMerkleStash`, you can tell the `SdtBlackHole` how much `SDT` it should distribute to a specific staking service. 
