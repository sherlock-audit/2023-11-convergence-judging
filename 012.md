Spare Leather Puma

high

# Users can game the governance voting by delegating back-and-forth

## Summary
With the current delegation structure, governance voting can be gamed. 

## Vulnerability Detail
A key part of governance voting is that either snapshot values should be used or after a vote, the votes should be locked. However, with the current implementation, there are no snapshots of users` mgCvg voting power.  Considering, there are no pauses to the delegations, this means that at any voting time, the user can delegate power from one of his wallets to their 2nd one, vote, re-delegate all power back to their first one and vote again, artificially increasing their voting power. Given enough wallets, the user can get basically infinite voting power. 

Example scenario: 
1. User A has X voting power.
2. User A votes once
3. User A delegates their power to their other wallet
4. User A votes from their other wallet
5. Repeat.

## Impact
Users can get pretty much infinite voting power 

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionDelegate.sol#L278C1-L279C1

## Tool used

Manual Review

## Recommendation
Add a lock to `delegateMgCvg` so users cannot re-delegate during active votes. Or consider adding snapshot values to the contract. 