Innocent Scarlet Quail

medium

# cvgControlTower and veCVG lock timing will be different and lead to yield loss scenarios

## Summary

When creating a locked CVG position, there are two more or less independent locks that are created. The first is in lockingPositionService and the other is in veCVG. LockingPositionService operates on cycles (which are not finite length) while veCVG always rounds down to the absolute nearest week. The disparity between these two accounting mechanism leads to conflicting scenario that the lock on LockingPositionService can be expired while the lock on veCVG isn't (and vice versa). Additionally tokens with expired locks on LockingPositionService cannot be extended. The result is that the token is expired but can't be withdrawn. The result of this is that the expired token must wait to be unstaked and then restaked, cause loss of user yield and voting power while the token is DOS'd.

## Vulnerability Detail

Cycles operate using block.timestamp when setting lastUpdateTime on the new cycle in [L345](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L345). It also requires that at least 7 days has passed since this update to roll the cycle forward in [L205](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L205). The result is that the cycle can never be exactly 7 days long and the start/end of the cycle will constantly fluctuate. 

Meanwhile when veCVG is calculating the unlock time it uses the week rounded down as shown in [L328](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/veCVG.vy#L328). 

We can demonstrate with an example:

Assume the first CVG cycle is started at block.timestamp == 1,000,000. This means our first cycle ends at 1,604,800. A user deposits for a single cycle at 1,400,000. A lock is created for cycle 2 which will unlock at 2,209,600. 

The lock on veCVG does not match this though. Instead it's calculation will yield:

    (1,400,000 + 2 * 604,800) / 604,800 = 4

    4 * 604,800 = 2,419,200

As seen these are mismatched and the token won't be withdrawable until much after it should be due to the check in veCVG [L404](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/veCVG.vy#L404).

This DOS will prevent the expired lock from being unstaked and restaked which causes loss of yield.

The opposite issue can also occur. For each cycle that is slightly longer than expected the veCVG lock will become further and further behind the cycle lock on lockingPositionService. This can also cause a dos and yield loss because it could prevent user from extending valid locks due to the checks in [L367](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/veCVG.vy#L367) of veCVG.

An example of this:

Assume a user locks for 96 weeks (58,060,800). Over the course of that year, it takes an average of 2 hours between the end of each cycle and when the cycle is rolled over. This effectively extends our cycle time from 604,800 to 612,000 (+7200). Now after 95 cycles, the user attempts to increase their lock duration. veCVG and lockingPositionService will now be completely out of sync:

After 95 cycles the current time would be:

    612,000 * 95 = 58,140,000

Whereas veCVG lock ended:

    612,000 * 96 = 58,060,800

According to veCVG the position was unlocked at 58,060,800 and therefore increasing the lock time will revert due to [L367](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/veCVG.vy#L367)

The result is another DOS that will cause the user loss of yield. During this time the user would also be excluded from taking place in any votes since their veCVG lock is expired.

## Impact

Unlock DOS that cause loss of yield to the user

## Code Snippet

[CvgRewards.sol#L341-L349](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L341-L349)

## Tool used

Manual Review

## Recommendation

I would recommend against using block.timestamp for CVG cycles, instead using an absolute measurement like veCVG uses.