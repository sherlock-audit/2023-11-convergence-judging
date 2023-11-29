Spare Leather Puma

medium

# If the multiple calls to `writeStakingRewards` cross a week's end, it will result in unfair distribution of rewards

## Summary
If the multiple calls to `writeStakingRewards` cross a week's end, it will result in unfair distribution of rewards

## Vulnerability Detail
The first call to `writeStakingRewards` calls `checkpoints` which makes sure all gauges are checkpointed up to the current week. However, there rises a issue if after `_checkpoints` the week end is crossed. This would allow for not up-to-date values of the gauges to be used. If the values are already added to the `totalWeightLocked`, its value will be inflated (as the gauge weights can only decrease in the time as votes are locked and time passes). 
```solidity
    function _setTotalWeight() internal {
        ICvgControlTower _cvgControlTower = cvgControlTower;
        IGaugeController _gaugeController = _cvgControlTower.gaugeController();
        uint128 _cursor = cursor;
        uint128 _totalGaugeNumber = uint128(gauges.length);

        /// @dev compute the theoric end of the chunk
        uint128 _maxEnd = _cursor + cvgRewardsConfig.maxLoopSetTotalWeight;
        /// @dev compute the real end of the chunk regarding the length of staking contracts
        uint128 _endChunk = _maxEnd < _totalGaugeNumber ? _maxEnd : _totalGaugeNumber;

        /// @dev if last chunk of the total weighted locked processs
        if (_endChunk == _totalGaugeNumber) {
            /// @dev reset the cursor to 0 for _distributeRewards
            cursor = 0;
            /// @dev set the step as DISTRIBUTE for reward distribution
            state = State.DISTRIBUTE;
        } else {
            /// @dev setup the cursor at the index start for the next chunk
            cursor = _endChunk;
        }

        totalWeightLocked += _gaugeController.get_gauge_weight_sum(_getGaugeChunk(_cursor, _endChunk));

        /// @dev emit the event only at the last chunk
        if (_endChunk == _totalGaugeNumber) {
            emit SetTotalWeight(_cvgControlTower.cvgCycle(), totalWeightLocked);
        }
    }
```

Then if any gauges have manually been checkpointed before the subsequent call to `_distributeCvgRewards` , it would mean that the sum of all weights of the gauges will be less than `totalWeightLocked`, meaning there will be underdistribution of rewards. If no gauges have been manually checkpointed, it would simply mean unfair distribution of rewards (as the values are not up-to-date).
```solidity
    function _distributeCvgRewards() internal {
        ICvgControlTower _cvgControlTower = cvgControlTower;
        IGaugeController gaugeController = _cvgControlTower.gaugeController();

        uint256 _cvgCycle = _cvgControlTower.cvgCycle();

        /// @dev number of gauge in GaugeController
        uint128 _totalGaugeNumber = uint128(gauges.length);
        uint128 _cursor = cursor;

        uint256 _totalWeight = totalWeightLocked;
        /// @dev cursor of the end of the actual chunk
        uint128 cursorEnd = _cursor + cvgRewardsConfig.maxChunkDistribute;

        /// @dev if the new cursor is higher than the number of gauge, cursor become the number of gauge
        if (cursorEnd > _totalGaugeNumber) {
            cursorEnd = _totalGaugeNumber;
        }

        /// @dev reset the cursor if the distribution has been done
        if (cursorEnd == _totalGaugeNumber) {
            cursor = 0;

            /// @dev reset the total weight of the gauge
            totalWeightLocked = 0;

            /// @dev update the states to the control_tower sync
            state = State.CONTROL_TOWER_SYNC;
        }
        /// @dev update the global cursor in order to be taken into account on next chunk
        else {
            cursor = cursorEnd;
        }

        uint256 stakingInflation = stakingInflationAtCycle(_cvgCycle);
        uint256 cvgDistributed;
        InflationInfo[] memory inflationInfos = new InflationInfo[](cursorEnd - _cursor);
        address[] memory addresses = _getGaugeChunk(_cursor, cursorEnd);
        /// @dev fetch weight of gauge relative to the cursor
        uint256[] memory gaugeWeights = gaugeController.get_gauge_weights(addresses);
        for (uint256 i; i < gaugeWeights.length; ) {
            /// @dev compute the amount of CVG to distribute in the gauge
            cvgDistributed = (stakingInflation * gaugeWeights[i]) / _totalWeight;

            /// @dev Write the amount of CVG to distribute in the staking contract
            ICvgAssetStaking(addresses[i]).processStakersRewards(cvgDistributed);

            inflationInfos[i] = InflationInfo({
                gauge: addresses[i],
                cvgDistributed: cvgDistributed,
                gaugeWeight: gaugeWeights[i]
            });

            unchecked {
                ++i;
            }
        }

        emit EventChunkWriteStakingRewards(_cvgCycle, _totalWeight, inflationInfos);
    }
```


Note: since the requirement on calling `checkpoint` is that at least 7 days have passed since the last distribution, it would mean that the delta of the checkpoint and the end of the week will gradually decrease every week, up until we once have a distribution crossing over a week's end. The issue above is bound to happen given long-enough timeframe., 

## Impact
Unfair distribution of rewards. Possible permanent loss of rewards.

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L279C1-L338C6

## Tool used

Manual Review

## Recommendation
Add time constraints to `writeStakingRewards` in order to make sure it does not happen close to the end of the week 