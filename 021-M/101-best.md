Bouncy Blue Opossum

medium

# Positions that have `veCVG` amount block users from extending their lock amount/time in the last cycle

## Summary

Users can lock a certain amount of `CVG` for a certain amount of time and get an NFT in return, this NFT is associated with an amount of `veCVG`, `mgCVG`, and/or `ysCVG`. During this time users can have the ability to extend the lock time and amount at any time in that locked duration, from the docs:
>A lock may be extended in amount (amount of `CVG` locked) and in duration at any time.

However, if an NFT is associated with an amount of `veCVG` (even if it's just 10%), users are blocked from extending their position's lock time and duration in the last cycle.

## Vulnerability Detail

When a position is opened and the `ysPercentage` is not 100%, the contract interacts with the voting escrow contract (veCVG), all functions related to unlocking time in that contract are rounding down the unlock time to weeks, so when it's called from the `LockingPositionService` contract, it's being rounded down. This is causing a wrong unlock time calculation check when trying to increase the lock time/amount at the last cycle, the failed check is the following:
```solidity
assert _locked.end > block.timestamp, "Lock expired"
```
Where `_locked.end` is always less than or equal to the current time, as the unlock time is rounded down.

**What makes the current behavior even weirder is that a user can still increase lock time/amount at the first 3 days only of the last cycle, and not the last 4 days.**

**Position with `veCVG`**

```javascript
it("Actions fail on last cycle when having veCVG", async () => {
    // User 1 mints position 1
    const user1TokenId = 1;
    await cvgContract.connect(user1).approve(lockingPositionServiceContract, ethers.MaxUint256);
    await lockingPositionServiceContract.connect(user1).mintPosition(11, ethers.parseEther("1"), 50, user1, true);

    // Increase to cycle 12 + 3 days
    await increaseCvgCycle(contractUsers, 11);
    await time.increase(time.duration.days(3));

    // Verify that the position is still valid, at the last cycle
    const [, positionEndCycle] = await lockingPositionServiceContract.lockingPositions(user1TokenId);
    const currentCycle = await controlTowerContract.cvgCycle();
    expect(positionEndCycle).to.be.eq(currentCycle);

    // Try increasing lock time on the last cycle, fails
    const increaseLockTimeTx = lockingPositionServiceContract.connect(user1).increaseLockTime(user1TokenId, 12);
    await expect(increaseLockTimeTx).to.be.revertedWith("Lock expired");

    // Try increasing lock time and amount on the last cycle, fails
    const increaseLockTimeAndAmountTx = lockingPositionServiceContract
        .connect(user1)
        .increaseLockTimeAndAmount(user1TokenId, 12, ethers.parseEther("1"), user1);
    await expect(increaseLockTimeAndAmountTx).to.be.revertedWith("Lock expired");

    // Another check to verify that the lock is still valid, can't burn
    const burnTx = lockingPositionServiceContract.connect(user1).burnPosition(user1TokenId);
    await expect(burnTx).to.be.revertedWith("LOCKED");
});
```




**Position without `veCVG`**

```javascript
it("Actions succeed on last cycle when not having veCVG", async () => {
    // User 1 mints position 1
    const user1TokenId = 1;
    await cvgContract.connect(user1).approve(lockingPositionServiceContract, ethers.MaxUint256);
    await lockingPositionServiceContract.connect(user1).mintPosition(11, ethers.parseEther("1"), 100, user1, true);

    // Increase to cycle 12 + 3 days
    await increaseCvgCycle(contractUsers, 11);
    await time.increase(time.duration.days(3));

    // Verify that the position is still valid, at the last cycle
    const [, positionEndCycle] = await lockingPositionServiceContract.lockingPositions(user1TokenId);
    const currentCycle = await controlTowerContract.cvgCycle();
    expect(positionEndCycle).to.be.eq(currentCycle);

    // Both actions succeed
    await lockingPositionServiceContract.connect(user1).increaseLockTime(user1TokenId, 12);
    await lockingPositionServiceContract.connect(user1).increaseLockTimeAndAmount(user1TokenId, 12, ethers.parseEther("1"), user1);

    // Another check to verify that the lock is still valid, can't burn
    const burnTx = lockingPositionServiceContract.connect(user1).burnPosition(user1TokenId);
    await expect(burnTx).to.be.revertedWith("LOCKED");
});
```

## Impact

Preventing users from extending their lock position at the last cycle of a lock.

## Code Snippet

1. https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L270-L274
2. https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L352
3. https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L419-L422
4. https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L481-L485

## Tool used

Manual Review

## Recommendation

Handle the flooring done on veCVG by revisiting the calculations done when calling `create_lock`, `increase_unlock_time`, and `increase_unlock_time_and_amount` - so the flooring includes the whole last cycle. 
