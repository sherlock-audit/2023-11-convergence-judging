Spare Leather Puma

high

# `increaseLockAndTime` does not correctly calculate `_newVotingPower`.

## Summary
`increaseLockAndTime` does not correctly calculate `_newVotingPower`. 

## Vulnerability Detail
User's `mgCvg` voting power is based on the amount they've escrowed and the lock time. Users can call `increaseLockAndTime` to both increase the amount locked and the time. The function is expected to properly increase the user's `mgCvg` balance, though this is nto the case 
```solidity
        if (lockingPosition.ysPercentage != MAX_PERCENTAGE) {
            /** @dev Update voting power through veCVG contract, link voting power to the nft tokenId. */
            uint256 amountVote = amount * (MAX_PERCENTAGE - lockingPosition.ysPercentage);
            _newVotingPower = (amountVote * (newEndCycle - actualCycle - 1)) / (MAX_LOCK * MAX_PERCENTAGE);
            lockingPosition.mgCvgAmount += _newVotingPower;

            _cvgControlTower.votingPowerEscrow().increase_unlock_time_and_amount(
                tokenId,
                block.timestamp + ((newEndCycle - actualCycle) * 7 days),
                amountVote / MAX_PERCENTAGE
            );
        }
```
As we can see the `mgCvg` amount is increased only by the newly locked amount multiplied by the new lock duration. However, it does not take in consideration the increase of the lock of the previously staked amount. 
This results in loss of voting power for the users who invoke `increaseLockAndTime` 

## Impact
Loss of Voting power

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L475C1-L486C10

## Tool used

Manual Review

## Recommendation
Take into consideration the previously staked tokens. 
