Precise Snowy Eagle

high

# In the case that the total supply of CVG tokens is insufficient, the last user to claim the rewards will suffer a loss

## Summary
In the case that the total supply of CVG tokens is insufficient, the last user to claim the rewards will suffer a loss
## Vulnerability Detail
1.https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L370
`claimCvgRewards()` method in the SdtStakingPositionService.sol is used to Claim CVG rewards for a Staking Position。

2.The _totalAmount in the `claimCvgRewards()` method is the cumulative reward for each cycle and is passed as an argument to the `mintStaking()` .

3.https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Token/Cvg.sol#L62
`mintStaking() `method in the Cvg.sol   is used to mint cvg to user
The problem is that in the `mintStaking() `, when  mintedStaking +_totalAmount > MAX_STAKING the function will revert,
so users  will lose rewards



For simplicity,
assume Total supply of Staking rewards is 190 
Go to cycle 1  (CYCLE_1)
userA deposit 100  (tokenid=1)
userB deposit 100  (tokenid=2)
Totalstaked = 200  (CYCLE_1+1)

Go to cycle 3  (CYCLE_3)
cvgRewardsAmount =200
userA claimCvgRewards `_totalAmountA` = 100 * 200/200=100
userB claimCvgRewards  `_totalAmountB`= 100 * 200/200=100

at this point , mintedStaking +_totalAmountB (100+100)>190,  `mintStaking() `method revert，so userB cannot claim the remaining reward.
_totalAmount may be cumulative rewards, userB may lose several cycles of rewards




## Impact
 The last user to claim the rewards will suffer a loss

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L370

https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Token/Cvg.sol#L62
## Tool used

Manual Review

## Recommendation
Add a judgment to the  `mintStaking() `method
```solidity

amount = newMintedStaking <= MAX_STAKING  ? amount : MAX_STAKING - mintedStaking;

```
