Sticky Chambray Bobcat

high

# Division by Zero in CvgRewards::_distributeCvgRewards leads to locked funds

## Summary

The bug occurs when `CvgRewards::_setTotalWeight` sets `totalWeightLocked` to zero, leading to a division by zero error in 
`CvgRewards::_distributeCvgRewards`, and blocking cycle increments. The blocking results in all Cvg locked to be unlockable permanently.

## Vulnerability Detail

The function `_distributeCvgRewards` of `CvgRewards.sol` is designed to calculate and distribute CVG rewards among staking contracts. It calculates the `cvgDistributed` for each gauge based on its weight and the total staking inflation. However, if the `totalWeightLocked` remains at zero (due to some gauges that are available but no user has voted for any gauge), the code attempts to divide by zero.

The DoS of `_distributeCvgRewards` will prevent cycle from advancing to the next state `State.CONTROL_TOWER_SYNC`, thus forever locking the users’ locked CVG tokens.

## Impact

Loss of users’ CVG tokens due to DoS of `_distributeCvgRewards` blocking the state.

## Code Snippet

```solidity
    function _setTotalWeight() internal {
        ...
❌      totalWeightLocked += _gaugeController.get_gauge_weight_sum(_getGaugeChunk(_cursor, _endChunk)); //@audit `totalWeightLocked` can be set to 0 if no gauge has received any vote
        ...
    }

```

```solidity
    function _distributeCvgRewards() internal {
        ...
        uint256 _totalWeight = totalWeightLocked;
        ...
        for (uint256 i; i < gaugeWeights.length; ) {
            /// @dev compute the amount of CVG to distribute in the gauge
❌          cvgDistributed = (stakingInflation * gaugeWeights[i]) / _totalWeight; //@audit will revert if `_totalWeight` is zero
        ...

```

```solidity
/**
* @notice Unlock CVG tokens under the NFT Locking Position : Burn the NFT, Transfer back the CVG to the user.  Rewards from YsDistributor must be claimed before or they will be lost.    * @dev The locking time must be over
* @param tokenId to burn
*/
function burnPosition(uint256 tokenId) external {
...
❌      require(_cvgControlTower.cvgCycle() > lastEndCycle, "LOCKED"); //@audit funds are locked if current `cycle <= lastEndCycle`
...
    }
```

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L321

## Tool used

## Recommendation
If the _totalWeight is zero, just consider the cvg rewards to be zero for that cycle, and continue with other logic:

```diff
-cvgDistributed = (stakingInflation * gaugeWeights[i]) / _totalWeight; 
+cvgDistributed = _totalWeight == 0 ? 0 : (stakingInflation * gaugeWeights[i]) / _totalWeight;
```