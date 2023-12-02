Innocent Scarlet Quail

high

# Tokens minted during non-TDE cycles vote count is unfairly reduced and can't get max vote boost causing significant loss of yield

## Summary

When locking tokens, the amount of time locked changes the voting power received from locking a token. This incentivizes users to lock for the max duration to get the most possible votes/rewards. This is problematic because locks are required to end on TDE cycles. The result is that users that stake on non-TDE cycles can never lock the max amount. These users are therefore disadvantaged and suffer loss of yield. If they wait to stake then they lose on yield while waiting and if they stake right away they will lose partial yield due to not being able to reach the max lock.

## Vulnerability Detail

[LockingPositionService.sol#L231-L254](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L231-L254)

    function mintPosition(
        uint96 lockDuration,
        uint256 amount,
        uint64 ysPercentage,
        address receiver,
        bool isAddToManagedTokens
    ) external onlyWalletOrWhiteListedContract {
        require(amount > 0, "LTE");
        require(ysPercentage <= MAX_PERCENTAGE, "YS_%_OVER_100");
        require(ysPercentage % RANGE_PERCENTAGE == 0, "YS_%_10_MULTIPLE");
        require(lockDuration <= MAX_LOCK, "MAX_LOCK_96_CYCLES");

        ICvgControlTower _cvgControlTower = cvgControlTower;

        uint96 actualCycle = uint96(_cvgControlTower.cvgCycle());
        uint96 endLockCycle = actualCycle + lockDuration;

     **@audit locking duration is unfairly limited**
        require(endLockCycle % TDE_DURATION == 0, "END_MUST_BE_TDE_MULTIPLE");

When creating a lock it is required that the final cycle of the lock must be a TDE multiple. The fundamental problem with this is that veCVG/ysCVG received is as follows ([L584](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L584)):

    (lockDuration * cvgLockAmount) / MAX_LOCK

It utilizes MAX_LOCK which is a constant 96 cycles. Meanwhile users are limited so that their end cycle is has to be a TDE multiple. This creates yield loss. Take the following example. The current cycle is 10. Any created lock MUST end on a TDE multiple and can't go over 96 weeks. The longest possible time the user can lock is only 86 cycles. If they attempted to lock for 96 then the final week would be 106 which isn't a multiple of 12. This means the user only receives a max:

    86 / 96 * cvgLockAmount = 89.6%

This 10.4% penalty is now applied to the user unfairly for the entire duration of their lock. This causes a definite and significant loss of yield to the user.

## Impact

Tokens minted during non-TDE cycles suffer unfair loss of yield.

## Code Snippet

[LockingPositionService.sol#L231-L312](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L231-L312)

## Tool used

Manual Review

## Recommendation

I see two solutions to this:
    
    1) Tokens should be calculated on the real max (i.e. using 86 instead of 96 in the above scenario
    2) User should be allowed to lock for longer than 96 to the nearest even TDE (i.e. a 98 cycle lock in the above scenario)