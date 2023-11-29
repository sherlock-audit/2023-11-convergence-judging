Proper Lemonade Sealion

medium

# cvgRewards may be incorrectly calculated due to possible changes in gagueWeights and totalWeight

## Summary
cvgRewards may be incorrectly calculated due to possible changes in gagueWeights and totalWeight

## Vulnerability Detail
The `totalWeight` is cached before the individual guage weights are captured to give out the rewards. Hence any change in gaugeWeight's between these can cause the rewards to be sent out incorrectly. The voting method to change the weights is prevented but there are other ways that can alter the weight albeit at low likelihood.

1. The `_setTotalWeight` and `_distributeCvgRewards` functions are called in different blocks there is a WEEK occuring b/w these blocks. In this case if the `checkpoint_gauge()` function is called on the gaugeController, the weight sums of the gauges will be lower than the previously computed `totalWeight` leading to less cvg rewards being sent out.
2. The `_setTotalWeight` and `_distributeCvgRewards` functions are called in different transactions there is a gauge is added / killed in b/w these blocks. This will also cause the gauge weight sum to be different from the totalWeight previosuly cached casuing less / more cvg inflation for the cycle. 

## Impact
cvg inlation may be different than expected if timing is not handled correctly and the user's may receive lower rewards

## Code Snippet
cached `totalWeightLocked` is used
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L289

## Tool used

Manual Review

## Recommendation
Be aware of the timing issues mentioned above and schedule actions accordingly