Spare Leather Puma

high

# `increaseLockTime` wrongfully calculates ysBalance

## Summary
`increaseLockTime` wrongfully calculates ysBalance 

## Vulnerability Detail
After putting `cvg` in an escrow, users receive `ys` balance. The ys balance is based on two things - the amount locked and the lock duration 
```solidity
        if (lockingPosition.ysPercentage != 0) {
            _ysCvgCheckpoint(
                lockingPosition.lastEndCycle - actualCycle,
                (amount * lockingPosition.ysPercentage) / MAX_PERCENTAGE,
                actualCycle,
                lockingPosition.lastEndCycle
            );
        }
```
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
After the users have already created their escrow, they can increase the lock's duration by calling `increaseLockTime`. However, let's look at what happens with the `ys` balance when the `increaseLockTime` is called: 
```solidity
        if (lockingPosition.ysPercentage != 0) {
            /** @dev Retrieve the balance registered at the cycle where the ysBalance is supposed to drop. */
            uint256 _ysToReport = balanceOfYsCvgAt(tokenId, oldEndCycle - 1);
            /** @dev Add this value to the tracking on the oldEndCycle. */
            totalSuppliesTracking[oldEndCycle].ysToAdd += _ysToReport;
            /** @dev Report this value in the newEndCycle in the Sub part. */
            totalSuppliesTracking[newEndCycle].ysToSub += _ysToReport;
        }
```
As we can see, new value is not calculated. The only thing that happens changing the cycle when the ys totalSupply will decrease. This corrupts the `totalSupplyOfYsCvg` 
This gives an unfair advantage to people who have already staked for a long time: and also puts people who significantly increase their lock at a disadvantage

Consider the following 2 scenarios 

#### Scenario 1
1. User has locked their tokens for a very short time (2 weeks)
2. User increases their lock time to max - 96 weeks
3. Despite the user having locked their tokens for 96 weeks, their ys balance is still based on only on the initial 2 weeks and is significantly smaller than what it is supposed to be 

#### Scenario 2
1. User has locked their tokens for max lock time - 96 weeks 
2. 95 weeks pass. User has 1 week left on their lock.
3. The user decides to increase their lock time by just 1 week.
4. Despite the user locking for only 1 additional week and having a lock which will last only 2 weeks, they have ys balance based on their 96 weeks lock.

## Impact
Corrupted global accounting 

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L407C1-L414C10

## Tool used

Manual Review

## Recommendation
Fix is non-trivial. The new balance must carefully be calculated and accounted for. 