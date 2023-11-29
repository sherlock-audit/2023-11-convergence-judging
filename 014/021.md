Spare Leather Puma

medium

# `balanceOfYsCvgAt` returns wrong results when `extension[i].cycleId % TDE_DURATION == 0`

## Summary
`balanceOfYsCvgAt` returns wrong results `extension[i].cycleId % TDE_DURATION == 0`

## Vulnerability Detail
Let's first look at `_ysCvgCheckpoint`
```solidity
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
            totalSuppliesTracking[realStartCycle].ysToAdd += ysTotalAmount;  //@audit - the user is accounted for this amount towards total supply 
            totalSuppliesTracking[realEndCycle].ysToSub += ysTotalAmount;
        }
    }
```
As we can see  we have 2 scenarios - if `actualCycle % TDE_DURATION != 0` and `actualCycle % TDE_DURATION == 0`
In the 2nd scenario, the user has a non-changing balance throughout the entirety of the lock duration, unlike in the 1st scenario, where the user has a partial balance up until `nextTdeCycle`.
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
                uint256 _ysPartial = ((_firstTdeCycle - _extension.cycleId) * _ysTotal) / TDE_DURATION; // @audit - this value will be returned
                /** @dev For locks that last less than 1 TDE. */
                if (_extension.endCycle - _extension.cycleId <= TDE_DURATION) {
                    _ysCvgBalance += _ysPartial; // @audit - this value will be returned, because of the duration of the lock
                } else {
                    _ysCvgBalance += _cycleId <= _firstTdeCycle ? _ysPartial : _ysTotal;
                }
            }
            ++i;
        }
        return _ysCvgBalance;
    }
```

However, if we look in the `balanceOfYsCvgAt` this is not implemented. 
In the case where a user has staked for a `duration < TDE_DURATION` and `actualCycle % TDE_DURATION == 0` the call to `balanceOfYsCvgAt` will calculate a significantly lower value - it will return `_ysPartial`, even though the user is accounted for `_ysTotal` towards the total supply.

## Impact
User will have significantly lower balance than expected 

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L656C14-L656C30
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L577

## Tool used

Manual Review

## Recommendation
Within `balanceOfYsCvgAt` check if `_cycleId % TDE_DURATION` and adjust accordingly
