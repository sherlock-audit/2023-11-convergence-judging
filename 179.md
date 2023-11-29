Innocent Scarlet Quail

high

# Bribe collection from sdtBlackHole is first-mover takes all which causes loss for multiple gauges that share the same bribe token

## Summary

When collecting bribes from sdtBlackHole, the entire balance of the bribe token is sent to the sdtRewardReceiver. This means that shared bribe tokens will credit the entire balance of the bribe token to the first buffer that claims regardless of how much was earned by each underlying token.

## Vulnerability Detail

When sdtBuffer is processing rewards, it pulls bribes from sdtBlackHole via the pullSdStakingBribes function in [L90](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBuffer.sol#L90). 

[SdtBlackHole.sol#L96-L134](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBlackHole.sol#L96-L134)

    function pullSdStakingBribes(
        address _processor,
        uint256 _processorRewardsPercentage
    ) external returns (ICommonStruct.TokenAmount[] memory) {

        ...

        for (uint256 i; i < bribeTokens.length; ) {
            IERC20 bribeToken = bribeTokens[i];

          **@audit this sends the entire balance of the token**

            uint256 balance = bribeToken.balanceOf(address(this));

            if (balance != 0) {
                uint256 claimerRewards = (balance * _processorRewardsPercentage) / 100_000;
                if (claimerRewards > 0) {
                    bribeToken.transfer(_processor, claimerRewards);
                    balance -= claimerRewards;
                }

                bribeToken.transfer(sdtRewardReceiver, balance);

                _bribeTokensAmounts[i] = ICommonStruct.TokenAmount({token: bribeTokens[i], amount: balance});
            }
            unchecked {
                ++i;
            }
        }
        return _bribeTokensAmounts;
    }

When sending the bribe tokens, sdtBlackHole uses the entire balance for sending and crediting the tokens. This creates an issue for bribe tokens that are shared (FXS, CRV, USDC, WETH, etc.). When multiple gauges are earning the same tokens from bribes then the first gauge that calls after the bribes are deposited will receive all of them, which doesn't properly distribute them to the gauges that actually earned them.

## Impact

Shared bribe tokens will be first-move takes all which steals rewards from other gauges

## Code Snippet

[SdtBlackHole.sol#L96-L134](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBlackHole.sol#L96-L134)

## Tool used

Manual Review

## Recommendation

Bribes should be sent directly to the buffer instead of to the black hole. Alternatively this could be fixed by adding sub accounts to the blackhole or by making a separate black hole for each asset.