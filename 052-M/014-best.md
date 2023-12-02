Spare Leather Puma

medium

# `mgCvg` balances are wrongfully calculated

## Summary
Users with significant difference in their locks may have the same `mgCVG` voting power 

## Vulnerability Detail
The `mgCvgCreated` is based on the amount a user has used for voting and their `lockDuration`. However, due to rounding down within the VotingEscrow contract, same users may get unfairly rewarded in comparison to others. 
Let's look at the code responsible for the `mgCvgCreated` amount within `mintPosition`
```solidity
        if (ysPercentage != MAX_PERCENTAGE) {
            uint256 amountVote = amount * (MAX_PERCENTAGE - ysPercentage);

            /** @dev Timestamp of the end of locking. */
            _cvgControlTower.votingPowerEscrow().create_lock(
                tokenId,
                amountVote / MAX_PERCENTAGE,
                block.timestamp + (lockDuration + 1) * 7 days
            );
            /// @dev compute the amount of mgCvg
            _mgCvgCreated = (amountVote * lockDuration) / (MAX_LOCK * MAX_PERCENTAGE);

            /// @dev Automatically add the veCVG and mgCVG in the balance taken from Snapshot.
            if (isAddToManagedTokens) {
                _cvgControlTower.lockingPositionDelegate().addTokenAtMint(tokenId, receiver);
            }
        }
```
As we know, the voting escrow contract round downs the lock time to the nearest week. However, this is not accounted for when calculating the `mgCvgCreated`. The amount is entirely based on the `lockDuration`. 
Consider the following scenario: 
If two users with equal stake both call `mintPosition` with the same `lockDuration` of (let's say) 2 weeks), but one calls it at the beginning of the week and the other one calls it at the end of the week. Because of the rounding down to the nearest week, one of the users will have locked their tokens for ~2 weeks, while the other one will have locked them for ~1 week. However, both users will receive the same amount of `mgCvg`. This is unfair for both users.

## Impact
Unfair calculation of `mgCvg`

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L266C1-L282C10

## Tool used

Manual Review

## Recommendation
Base the `mgCvg` calculated on the block.timestamp
