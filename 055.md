Happy Rouge Cheetah

medium

# SdtFeeCollector.setUpFeesRepartition should send current rewards to the previous recipients

## Summary
SdtFeeCollector.setUpFeesRepartition function changes recipient for the next `withdrawSdt` call, but it should send current rewards to the previous recipients first.
## Vulnerability Detail
User can stake their CvgSdt tokens into `SdtStakingPositionService` contract. For doing so, Convergence protocol pays them additional SDT tokens for each cycle. This SDT rewards are just some percentage of SDT rewards of all gauge assets. When gauge receives SDT token as rewards, then some percentage of it [is sent to the `SdtFeeCollector` contract](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBuffer.sol#L115-L122). And `SdtFeeCollector` is created in such way, that part of all collected SDT [is sent to the `_cvgSdtBuffer`](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtFeeCollector.sol#L60). So when `withdrawSdt` is called, then all SDT is sent out to the recipients, [according to the percentage](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtFeeCollector.sol#L111) and 1 of recipients is `_cvgSdtBuffer`.

Using `SdtFeeCollector.setUpFeesRepartition` function [it's possible to change recipients and/or their shares](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtFeeCollector.sol#L80-L101). The problem is that in case of this call, the function should send current SDT amount to the current recipients and only then change recipients. Otherwise, CvgSdt stakers can receive incorrect rewards amount for a cycle, for example.
## Impact
Previous recipients receive wrong share of SDT tokens
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Call `withdrawSdt` first and then change recipients and their shares.