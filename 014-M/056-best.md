Happy Rouge Cheetah

medium

# CvgSdtBuffer.pullRewards should be called after all gauges has processed rewards already

## Summary
As SDT rewards of CvgSdtBuffer depends on other's gauges SDT rewards(as percentage of them), that means that in case if CvgSdtBuffer.pullRewards will be called before all other gauges SDT rewards for the cycle will be received, then stakes of CvgSdtBuffer will receive smaller rewards amount.
## Vulnerability Detail
User can stake their CvgSdt tokens into `SdtStakingPositionService` contract. For doing so, Convergence protocol pays them additional SDT tokens for each cycle(not only this, there are other rewards as well). These SDT rewards are just some percentage of SDT rewards of all other gauges. When gauge receives SDT token as reward, then some percentage of it [is sent to the `SdtFeeCollector` contract](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBuffer.sol#L115-L122). And `SdtFeeCollector` is created in such way, that part of all collected SDT [is sent to the `_cvgSdtBuffer`](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtFeeCollector.sol#L60). So when `withdrawSdt` is called, then all SDT is sent out to the recipients, [according to the percentage](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtFeeCollector.sol#L111) and one of recipients is `_cvgSdtBuffer`.

In order to get rewards from StakeDao someone needs to call `SdtStakingPositionService.processSdtRewards`. This function [can be called only once per cycle](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L556)(after cycle has finished) and [it calls `buffer.pullRewards`](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L559), which will claim all pending rewards.

This function can be called for both staked gauges and CvgStd. The difference is that for gauge this function will [then call `_gaugeAsset.claim_rewards`](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBuffer.sol#L87), which will claim all rewards from StakeDao for this gauge. And for CvgSdt it will claim rewards in other way and one of steps will be [to claim SDT from `SdtFeeCollector`](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/CvgSdtBuffer.sol#L89).

Because `CvgSdtBuffer` SDT balance depends on other gauges in the system(they should already claim rewards) it means that `SdtStakingPositionService.processSdtRewards` should be called for CvgSdt only after it was called for all other gauges. Otherwise, it means that not whole amount of SDT were sent to the `CvgSdtBuffer` and as result smaller amount of SDT is distributed to the stakers for the cycle. As `processSdtRewards` can be called by anyone, it means that such situation will likely to occur: maliciously or not.
## Impact
CvgSdt stakers will receive smaller amount of SDT rewards for the cycle.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Each gauge can register if it have already claimed rewards for the cycle to the tower contract. Then only when all gauges have claimed rewards, then `SdtStakingPositionService` of CvgSdt is allowed to claim rewards.