Spare Leather Puma

medium

# Certain functions should not be usable when `GaugeController` is locked.

## Summary
Possible unfair over/under distribution of rewards

## Vulnerability Detail
When `writeStakingRewards` is invoked for the first time it calls `_checkpoints` which sets the lock in the GaugeController to true. What this does is it doesn't allow for any new vote changes. The idea behind it is that until the rewards are fully distributed there are no changes in the gauges' weights so the distribution of rewards is correct. 
However, there are multiple unrestricted functions which can alter the outcome of the rewards and result in not only unfair distribution, but also to many overdistributed or underdistributed rewards.
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

If any of `change_gauge_weight` `change_type_weight` or  is called after the `totalWeightLocked` is calculated, it will result in incorrect distribution of rewards. When `_distributeCvgRewards` is called, some gauges may not have the same value that has been used to calculate the `totalWeightLocked` and this may result in distribution too many or too little rewards. It also gives an unfair advantage/disadvantage to the different gauges. 
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


## Impact
Unfair distribution of rewards. Over/underdistributing rewards.

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L244C1-L272C6

## Tool used

Manual Review

## Recommendation
Add a lock to `change_gauge_weight` `change_type_weight` 
