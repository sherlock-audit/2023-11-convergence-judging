Spare Leather Puma

high

# `balanceOfYsCvgAt` returns wrong value if `cycleId == _firstTdeCycle`

## Summary
`balanceOfYsCvgAt` returns wrong value if `cycleId == _firstTdeCycle`

## Vulnerability Detail
In order to understand the issue we need to first look at how `ysCvg` is checkpointed. 
```solidity
    function _ysCvgCheckpoint(
        uint256 lockDuration,
        uint256 cvgLockAmount,
        uint256 actualCycle,
        uint256 endLockCycle
    ) internal {
        /** @dev Compute the amount of ysCVG on this Locking Position proportionally with the ratio of lockDuration and MAX LOCK duration. */
        uint256 ysTotalAmount = (lockDuration * cvgLockAmount) / MAX_LOCK;
        uint256 realStartCycle = actualCycle + 1;
        uint256 realEndCycle = endLockCycle + 1;
        /** @dev If the lock is not made on a TDE cycle,   we need to compute the ratio of ysCVG  for the current partial TDE */
        if (actualCycle % TDE_DURATION != 0) {
            /** @dev Get the cycle id of next TDE to be taken into account for this LockingPosition. */
            uint256 nextTdeCycle = (actualCycle / TDE_DURATION + 1) * TDE_DURATION + 1;
            /** @dev Represent the amount of ysCvg to be taken into account on the next TDE of this LockingPosition. */
            uint256 ysNextTdeAmount = ((nextTdeCycle - realStartCycle) * ysTotalAmount) / TDE_DURATION;

            totalSuppliesTracking[realStartCycle].ysToAdd += ysNextTdeAmount;

            /** @dev When a lock is greater than a TDE_DURATION */
            if (lockDuration >= TDE_DURATION) {
                /** @dev we add the calculations for the next full TDE */
                totalSuppliesTracking[nextTdeCycle].ysToAdd += ysTotalAmount - ysNextTdeAmount;
                totalSuppliesTracking[realEndCycle].ysToSub += ysTotalAmount;
            }
            /** @dev If the lock less than TDE_DURATION. */
            else {
                /** @dev We simply remove the amount from the supply calculation at the end of the TDE */
                totalSuppliesTracking[realEndCycle].ysToSub += ysNextTdeAmount;
            }
        }
        /** @dev If the lock is performed on a TDE cycle  */
        else {
            totalSuppliesTracking[realStartCycle].ysToAdd += ysTotalAmount;
            totalSuppliesTracking[realEndCycle].ysToSub += ysTotalAmount;
        }
    }
```
Here we need to make 2 key takeaways:
1. The totalsupply at the current cycle is equal to the `totalsupply at the previous cycle + totalSuppliesTracking[currentCycle].ysToAdd - totalSuppliesTracking[currentCycle].ysToSub` 
2. If a user's lock duration is over 12 weeks (`TDE_DURATION`), ys balance starts at a significantly reduced value `((nextTdeCycle - realStartCycle) * ysTotalAmount) / TDE_DURATION;)` and increases to `ysTotalAmount` at `nextTdeCycle`

Though let's check how the user's balance is calculated in `balanceOfYsCvgAt`: 
```solidity
    function balanceOfYsCvgAt(uint256 _tokenId, uint256 _cycleId) public view returns (uint256) {
        require(_cycleId != 0, "NOT_EXISTING_CYCLE");

        LockingPosition memory _lockingPosition = lockingPositions[_tokenId];
        LockingExtension[] memory _extensions = lockExtensions[_tokenId];
        uint256 _ysCvgBalance;

        /** @dev If the requested cycle is before or after the lock , there is no balance. */
        if (_lockingPosition.startCycle >= _cycleId || _cycleId > _lockingPosition.lastEndCycle) {
            return 0;
        }
        /** @dev We go through the extensions to compute the balance of ysCvg at the cycleId */
        for (uint256 i; i < _extensions.length; ) {
            /** @dev Don't take into account the extensions if in the future. */
            if (_extensions[i].cycleId < _cycleId) {
                LockingExtension memory _extension = _extensions[i];
                uint256 _firstTdeCycle = TDE_DURATION * (_extension.cycleId / TDE_DURATION + 1);
                uint256 _ysTotal = (((_extension.endCycle - _extension.cycleId) *
                    _extension.cvgLocked *
                    _lockingPosition.ysPercentage) / MAX_PERCENTAGE) / MAX_LOCK;
                uint256 _ysPartial = ((_firstTdeCycle - _extension.cycleId) * _ysTotal) / TDE_DURATION;
                /** @dev For locks that last less than 1 TDE. */
                if (_extension.endCycle - _extension.cycleId <= TDE_DURATION) {
                    _ysCvgBalance += _ysPartial;
                } else {
                    _ysCvgBalance += _cycleId <= _firstTdeCycle ? _ysPartial : _ysTotal;  // @audit - important line 
                }
            }
            ++i;
        }
        return _ysCvgBalance;
    }
```
Let's look specifically look at the case where `_extension.endCycle - _extension.cycleId >= TDE_DURATION)` (when we reach the else statement) 
In the case where `_cycleId == firstTdeCycle`, the returned value will be `_ysPartial`, Though as we examined above, the ys balance has increased at that exact cycle. This means that in this case `balanceOfYsCvgAt` will return a significantly reduced value. 

In an example scenario where the user is the only ys staker and `_extension.endCycle - _extension.cycleId <= TDE_DURATION`, there will be a mismatch between the results from calling `balanceOfYsCvgAt` with `_firstTdeCycle` as an argument and `totalSupplyOfYsCvgAt` for the same cycle.  

## Impact
`balanceOfYsCvgAt` will return significantly reduced value any time it is called with parameter `cycleId == _firstTdeCycle` (up to 11/12 or ~91% reduced value) 

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L681

## Tool used

Manual Review

## Recommendation
Change the <= to <
```solidity
-                    _ysCvgBalance += _cycleId <= _firstTdeCycle ? _ysPartial : _ysTotal;
 
+                    _ysCvgBalance += _cycleId < _firstTdeCycle ? _ysPartial : _ysTotal;
```