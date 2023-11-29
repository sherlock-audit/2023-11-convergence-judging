Main Shadow Hare

high

# Users can't receive rewards in the actual `cvgCycle` due to unexpected error

## Summary
Users are unable to receive rewards in the actual `cvgCycle` due to the `_lockingPositionService.totalSupplyYsCvgHistories(cycleClaimed)` returns `0` for the actual `cvgCycle`. This means users cannot claim rewards for the Token Distribution Event if the actual `cvgCycle` matchs with their last cycle of the token ownership or delegation.

## Vulnerability Detail
According to the [docs](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/technical-docs/lock/YsDistributor.md#validation-1) the `YsDistributor.claimRewards` function is designed to claim rewards associated with a particular token ID that represents a locking position NFT. It checks the ownership of the token is the owner or the delegatee and that the TDE is in the past or current cycle, meaning the rewards are available to be claimed. So users should be able to claim rewards for the actual `cvgCycle`. The share of rewards is computed by doing the ratio between the ysCvgBalance of the token and the totalSupply of ysCvg at the specified TDE.
```solidity
        uint256 share = (_lockingPositionService.balanceOfYsCvgAt(tokenId, cycleClaimed) * 10 ** 20) /
            _lockingPositionService.totalSupplyYsCvgHistories(cycleClaimed);
```
But the `_lockingPositionService.totalSupplyYsCvgHistories(cycleClaimed)` returns 0 for the actual `cvgCycle` because last known total supply which the `LockingPositionDelegate.totalSupplyYsCvgHistories` mapping contains is for `cvgControlTower.cvgCycle() - 1` cycle.
https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L541-L563
https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L830-L836

## Impact
Users can't receive rewards for the actual `cvgCycle` therefore never receive rewards for the Token Distribution Event if the actual `cvgCycle` matches with their last cycle of the token ownership or delegation.

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Rewards/YsDistributor.sol#L185

## Tool used

Manual Review

## Recommendation
Consider using the `totalSupplyOfYsCvgAt` function to receive total supply of ysCvg at the time
```diff
 184        uint256 share = (_lockingPositionService.balanceOfYsCvgAt(tokenId, cycleClaimed) * 10 ** 20) /
-185            _lockingPositionService.totalSupplyYsCvgHistories(cycleClaimed);
+185            _lockingPositionService.totalSupplyOfYsCvgAt(cycleClaimed);
```