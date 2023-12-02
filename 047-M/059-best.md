Happy Rouge Cheetah

medium

# LockingPositionService.mintPosition creates 1 week longer lock than user requested

## Summary
LockingPositionService.mintPosition creates 1 week longer lock than user requested.
## Vulnerability Detail
`LockingPositionService.mintPosition` function allows Cvg holder to stake his tokens to the LockingPositionService in order to be able to participate is Dao and/or receive treasury rewards.

User provides `lockDuration` param to the function, which is amount of cycles that user has agreed to lock his tokens for. Using that param `endLockCycle` [is calculated](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L250), which is the cycle, [when lock is going to be expired](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L286).  

In case if `ysPercentage` is not 100%, that means that user would like to receive some voting power from part of Cvg tokens.
In this case [veCvg lock is created for user](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L270-L274). The problem is that `_unlock_time` for it is provided as `block.timestamp + (lockDuration + 1) * 7 days`, which is 1 week bigger than staker has requested.
## Impact
Lock is created for 1 week longer than user has requested.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Do not increase duration:
```solidity
_cvgControlTower.votingPowerEscrow().create_lock(
                tokenId,
                amountVote / MAX_PERCENTAGE,
                block.timestamp + lockDuration * 7 days
            );
```