Creamy Infrared Otter

high

# wrong time when increasing the locking time

## Summary
the function ((
## Vulnerability Detail
the function **increaseLockTimeAndAmount*** is used to increase the locking time and when calculating the time in line 451  it will add **1** into the last end cycle and  and on line 455 it will try to subtract the added number when he is checking  if the time duration of the lock is not more than the allowed or the max one  and it will also try to subtract it when locking into **yscvg**
` 
_ysCvgCheckpoint(
                newEndCycle - actualCycle - 1,
                (amount * lockingPosition.ysPercentage) / MAX_PERCENTAGE,
                actualCycle,
                newEndCycle - 1
            );`
but it will not subtract it in the votingEscrow on line 581 
`
 _cvgControlTower.votingPowerEscrow().increase_unlock_time_and_amount(
                tokenId,
                block.timestamp + ((newEndCycle - actualCycle) * 7 days),
                amountVote / MAX_PERCENTAGE
            );
`
which will cause to add 1 extra week to unlock
## Impact
a user can't unlock there token on the specified  date they said 
## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L481
## Tool used

Manual Review

## Recommendation
 subtract 1 when  locking in the **votingpowerescrow**