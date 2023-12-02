Innocent Scarlet Quail

high

# Tokens that are both bribes and StakeDao gauge rewards will cause loss of funds

## Summary

When SdtStakingPositionService is pulling rewards and bribes from buffer, the buffer will return a list of tokens and amounts owed. This list is used to set the rewards eligible for distribution. Since this list is never check for duplicate tokens, a shared bribe and reward token would cause the token to show up twice in the list. The issue it that _sdtRewardsByCycle is set and not incremented which will cause the second occurrence of the token to overwrite the first and break accounting. The amount of token received from the gauge reward that is overwritten will be lost forever.

## Vulnerability Detail

In [L559](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L559) of SdtStakingPositionService it receives a list of tokens and amount from the buffer.

[SdtBuffer.sol#L90-L168](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBuffer.sol#L90-L168)

        ICommonStruct.TokenAmount[] memory bribeTokens = _sdtBlackHole.pullSdStakingBribes(
            processor,
            _processorRewardsPercentage
        );

        uint256 rewardAmount = _gaugeAsset.reward_count();

        ICommonStruct.TokenAmount[] memory tokenAmounts = new ICommonStruct.TokenAmount[](
            rewardAmount + bribeTokens.length
        );

        uint256 counter;
        address _processor = processor;
        for (uint256 j; j < rewardAmount; ) {
            IERC20 token = _gaugeAsset.reward_tokens(j);
            uint256 balance = token.balanceOf(address(this));
            if (balance != 0) {
                uint256 fullBalance = balance;

                ...

                token.transfer(sdtRewardsReceiver, balance);

              **@audit token and amount added from reward_tokens pulled directly from gauge**

                tokenAmounts[counter++] = ICommonStruct.TokenAmount({token: token, amount: balance});
            }

            ...

        }

        for (uint256 j; j < bribeTokens.length; ) {
            IERC20 token = bribeTokens[j].token;
            uint256 amount = bribeTokens[j].amount;

          **@audit token and amount added directly with no check for duplicate token**

            if (amount != 0) {
                tokenAmounts[counter++] = ICommonStruct.TokenAmount({token: token, amount: amount});

            ...

        }

SdtBuffer#pullRewards returns a list of tokens that is a concatenated array of all bribe and reward tokens. There is not controls in place to remove duplicates from this list of tokens. This means that tokens that are both bribes and rewards will be duplicated in the list.

[SdtStakingPositionService.sol#L561-L577](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L561-L577)

        for (uint256 i; i < _rewardAssets.length; ) {
            IERC20 _token = _rewardAssets[i].token;
            uint256 erc20Id = _tokenToId[_token];
            if (erc20Id == 0) {
                uint256 _numberOfSdtRewards = ++numberOfSdtRewards;
                _tokenToId[_token] = _numberOfSdtRewards;
                erc20Id = _numberOfSdtRewards;
            }

          **@audit overwrites and doesn't increment causing duplicates to be lost**            

            _sdtRewardsByCycle[_cvgStakingCycle][erc20Id] = ICommonStruct.TokenAmount({
                token: _token,
                amount: _rewardAssets[i].amount
            });
            unchecked {
                ++i;
            }
        }

When storing this list of rewards, it overwrites _sdtRewardsByCycle with the values from the returned array. This is where the problem arises because duplicates will cause the second entry to overwrite the first entry. Since the first instance is overwritten, all funds in the first occurrence will be lost permanently.

## Impact

Tokens that are both bribes and rewards will be cause tokens to be lost forever 

## Code Snippet

[SdtStakingPositionService.sol#L550-L582](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L550-L582)

## Tool used

Manual Review

## Recommendation

Either sdtBuffer or SdtStakingPositionService should be updated to combine duplicate token entries and prevent overwriting.