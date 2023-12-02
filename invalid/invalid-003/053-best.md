Perfect Coffee Sawfish

high

# getClaimableCyclesAndAmounts() getter function will return wrong ClaimableCyclesAndAmounts[] because of rounding issues

## Summary
getClaimableCyclesAndAmounts() function is used to get an array of token and reward associated to the Staking position sorted by cycleId. However this function will return wrong values in certain cases causing a difference between real  rewards and what is actually returned.

## Vulnerability Detail
[getClaimableCyclesAndAmounts()](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L997) function will cause a loss of precision when calculating the  cvgAmount. This [code](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L1027) is where the issue happens.

For example:
- amountStaked = 157
- _cycleInfo[lastClaimedSdt].cvgRewardsAmount= 100
- totalStaked= 1000
- cvgAmount will be (157*100)/1000 // it will return 15 instead of 15.7 because of solidity truncation

## Impact
Function will return a value different than the real value.

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L1027

## Tool used

Manual Review,
VsCode

## Recommendation
Use a multiplier for making operations that can lead to rounding down issues
