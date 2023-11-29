Spare Leather Puma

high

# Removing a gauge while rewards are being distributed will result in incorrect distribution.

## Summary
Removing a gauge while rewards are being distributed will result in incorrect distribution.

## Vulnerability Detail
If a gauge is removed after its weight has been accounted in the `totalWeightLocked`, it will result in incorrect amount of rewards distributed.
There would be 2 possible scenarios for the difference in rewards distributed .
1. If all gauges have been accounted for in `totalWeightLocked`, when calling `_distributeCvgRewards` it will distribute a smaller amount of rewards than expected. 
2. If the last gauge has not yet been accounted for, it will not get accounted for at all (as it will take the id of the removed gauge). Depending on the weight the last gauge holds, it may result in both serious overdistribution or underdistribution of rewards. 
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



## Impact
Incorrect distribution of rewards

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L244C1-L272C6

## Tool used

Manual Review

## Recommendation
Do not allow for gauges to be removed if `_state != State.CHECKPOINT`
