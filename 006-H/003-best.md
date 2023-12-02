Recumbent Shamrock Marmot

medium

# LockingPositionService::increaseLockTime()  should also increase both the ysCvg and mgCvg value

## Summary

increaseLockTime() function should increase both the mgCvg and ysCvg values too. Otherwise, users who increased the lock time will get less share of tokens on reward distribution and less voting power than the people who minted a position with the same lock time and CVG amount.

## Vulnerability Detail

The value of ysCvg is calculated by the formula https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L584
The value of mgCvg is calculated by the formula https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L276


We can see that both the ysCvg and mgCvg value is dependent on `lockDuration` which means the longer the locking period the more the ysCvg and mgCvg value. And, it makes sense to have this because the people who choose to lock for a longer time have higher incentives in shares and voting power.

But, the `increaseLockTime()` function does not increment the ysCvg and mgCvg values. Here, the values of ysCvg and mgCvg should be re-calculated and updated on state variables according to the new lock duration.

## Impact
Not updating the two values will result in grievances of a user who increased the time lock because that particular person gets fewer shares on reward distribution and fewer voting power. 
Most importantly, since the `totalSuppliesTracking` is updated on the old and new cycle, his portion of a share will be claimed by other users.
Though it's not an attack by external actors, it's a critical bug in the protocol that's causing grievances for users.

## Code Snippet

Assume the current cycle is `5`

- `Alice` is minting a position with a lock duration of `55` and CVG amount of `100e18` and ysPercent `50`
- Her ysCvg and mgCvg value is `28645833333333333333`
- Now `Bob` minting a position with a lock duration of `43` and CVG amount of `100e18` and ysPercent `50`(Both same as Alice)
- His ysCvg and mgCvg value is `22395833333333333333`
- But then immediately `Bob` decided to increment the lock duration by '12', which resulted in a total of 60. (43 + 5 + 12).
- After the increment his ysCvg and mgCvg value remains the same.
- But, `Bob` now has the same CVG and lock duration as Alice but he has less ysCvg and mgCvg value.
- Because of this his share amount on claiming reward will also be reduced. And, his remaining portion of the share is spread over to other users claiming at that TDE.
(Below code is modified from `unlock-test.spec.ts`.)
```js
    it("Check mgCvg values", async () => {
        //Alice and Bob CVG token approval
        await (await cvgContract.connect(user1).approve(lockingPositionService, LOCKING_POSITIONS[0].cvgAmount)).wait();
        await (await cvgContract.connect(user2).approve(lockingPositionService, LOCKING_POSITIONS[0].cvgAmount)).wait();

        //Alice minting position for 100e18, for locking period 55 and ysPercent is 50
        const res = await (
            await lockingPositionService.connect(user1).mintPosition(55, LOCKING_POSITIONS[0].cvgAmount, 50, user1, true)
        ).wait();

        //Bob minting position for  100e18, for locking period 43 and ysPercent is 50
        const res2 = await (
            await lockingPositionService.connect(user2).mintPosition(43, LOCKING_POSITIONS[0].cvgAmount, 50, user2, true)
        ).wait();

        console.log('Before time lock increase...')
        //Alice's token Id
        console.log(await lockingPositionService.lockingPositions(1)); //lastEndCycle: 60, mgCvgAmount: 28645833333333333333
        //Bob's token id
        console.log(await lockingPositionService.lockingPositions(2)); //lastEndCycle: 48, mgCvgAmount: 22395833333333333333


        //Bob increasing his locking period
        await lockingPositionService.connect(user2).increaseLockTime(2, 12);

        console.log('After time lock increase...');
        await increaseCvgCycle(contractUsers, 43);
        const user1YsBalance = await lockingPositionService.balanceOfYsCvgAt(1, 36);
        const user2YsBalance = await lockingPositionService.balanceOfYsCvgAt(2, 36);

        console.log(user1YsBalance, user2YsBalance);


        const totalYsSupplyHistory = await lockingPositionService.totalSupplyYsCvgHistories(36);

        console.log('Share of user1', ((BigNumber.from(user1YsBalance).mul(ethers.parseUnits('10', 20))).div(BigNumber.from(totalYsSupplyHistory))).toString())
        // ^ 561224489795918367347
        console.log('Share of user2', ((BigNumber.from(user2YsBalance).mul(ethers.parseUnits('10', 20))).div(BigNumber.from(totalYsSupplyHistory))).toString())
        // ^ 438775510204081632652

        //Alice's token Id
        console.log(await lockingPositionService.lockingPositions(1)); //lastEndCycle: 60, mgCvgAmount: 28645833333333333333
        //Bob's token id
        console.log(await lockingPositionService.lockingPositions(2)); //lastEndCycle: 60, mgCvgAmount: 22395833333333333333
});
```
- In the above code we see that Alice has more share `561224489795918367347` than Bob `438775510204081632652`. But, it was supposed to be equal and this imbalance happened because of not updating ysCvg and mgCvg on `increaseLockTime()`

## Tool used

Manual Review

## Recommendation

Re-calculate the `mgCvgAmount` and call `_ysCvgCheckpoint` on `increaseTimeLock`
```solidity
         newMgCvgAmount = (amountVote * (newEndCycle - oldEndCycle)) /(MAX_LOCK * MAX_PERCENT)
         lockingPosition.mgCvgAmount += newMgCvgAmount;
         _ysCvgCheckpoint(durationAdd, 
              (lockingPosition.cvgLocked * lockingPosition.ysPercentage) / MAX_PERCENTAGE,
              actualCycle,
              newEndCycle
         );
```
