Bouncy Cider Hippo

high

# `LockingPositionService` doesn't update `ysCVG` supply correctly on duration increase, leading to potential rewards inflation.

## Summary

The `increaseLockTimeAndAmount` function in `LockingPositionService` does not properly update ysCVG supply accounting via `_ysCvgCheckpoint` when increasing lock duration. This can allow users to claim inflated ysCVG rewards.

## Vulnerability Detail 

The `_ysCvgCheckpoint` function updates the `totalSuppliesTracking` mapping that tracks ysCVG supply changes each cycle. It is called on minting new locks and increasing lock amount, but NOT when only increasing lock duration.

So if a user increases their lock duration but not amount, `totalSuppliesTracking` will not be updated to account for the increased duration. This leads to an incorrect ysCVG total supply.

#### Root cause

The `increaseLockTimeAndAmount` function allows increasing both the lock duration and amount of CVG locked for a locking position NFT. It properly updates the voting power and total CVG locked, but does miss updating the ysCVG supply accounting via `_ysCvgCheckpoint` when increasing duration. The root cause is that `_ysCvgCheckpoint` is only called when increasing the lock amount on [lines 465-469](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L465-L470).

```solidity
_ysCvgCheckpoint(
                newEndCycle - actualCycle - 1, 
                (amount * lockingPosition.ysPercentage) / MAX_PERCENTAGE,
                actualCycle,
                newEndCycle - 1
);
```
However, it is not called when only increasing lock duration. This means the ysCVG supply accounting will be incorrect when increasing duration.

As shown, the `_ysCvgCheckpoint` function is responsible for updating the `totalSuppliesTracking` mapping which tracks the total ysCVG supply changes each cycle. 

It gets called in two places:

1. On minting a new locking position - This properly sets the initial ysCVG supply for the lock duration.

2. When increasing lock amount - This updates `totalSuppliesTracking` for the increased amount. 

The cause is that `_ysCvgCheckpoint` does NOT get called when only increasing lock duration. 

So if a user increases duration but not amount, `totalSuppliesTracking` will not be updated to account for the longer lock period.

**For example**

1. Alice mints a 12 week lock with 10% ysCVG. 
   - `_ysCvgCheckpoint` called properly, `totalSuppliesTracking` updated for 12 weeks of ysCVG.

2. 2 weeks later, Alice increases lock to 24 weeks (duration +12 weeks).
   - `_ysCvgCheckpoint` NOT called here because no amount increase. 

3. `totalSuppliesTracking` still only reflects original 12 weeks of ysCVG.
   - But Alice's lock is now 24 weeks, so she will be able to claim more ysCVG than originally accounted for.

4. When Alice claims ysCVG rewards, they will be inflated because `totalSuppliesTracking` was never updated to 24 weeks.

So by not calling `_ysCvgCheckpoint` when increasing duration, the ysCVG supply can become inflated.

## Impact

The total supply of ysCVG tokens will be incorrectly calculated when users increase their lock duration but not amount. Specifically, the `totalSuppliesTracking` mapping which accumulates ysCVG supply changes each cycle will not be updated.

This could allow users to claim more ysCVG rewards than they are entitled to if the actual ysCVG supply is lower than expected.

It could also lead to confusion around the circulating supply of ysCVG when dashboard totals do not match the on-chain data.

The risk is that a user could dramatically increase their lock duration without increasing amount, and claim excess ysCVG due to the supply discrepancy.

For example, if they extended a 12 week lock to a 2 year lock, they could claim far more ysCVG than initially minted for those first 12 weeks. 

This could throw off the total ysCVG supply and circulating supply, and lead to loss of fees or rewards for other ysCVG holders.

The impact scales with the difference between the original and extended lock durations. 

- Users can claim more ysCVG rewards than they should be entitled to based on the actual supply.

- Throws off the total ysCVG circulating supply tracked on dashboards.

- Loss of fees or rewards for other ysCVG holders due to supply discrepancy.

- The impact scales exponentially with the difference between original and extended lock duration.

## Code Snippet

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L439-L505

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L465-L470

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L94

The `_ysCvgCheckpoint` call is missing when increasing duration:

```solidity
function increaseLockTimeAndAmount(
  uint256 tokenId,
  uint256 durationAdd, // increasing duration
  uint256 amount, 
  address operator
) external {

  ...

  if (lockingPosition.ysPercentage != 0) {

    // Update ysCVG supply for increased amount
    _ysCvgCheckpoint(
      newEndCycle - actualCycle - 1,
      (amount * lockingPosition.ysPercentage) / MAX_PERCENTAGE,
      actualCycle,
      newEndCycle - 1
    );

  }

  ...

}
```

## Tool used

Manual Review

## Recommendation

Call `_ysCvgCheckpoint` when increasing duration to properly update `totalSuppliesTracking`.

```solidity
_ysCvgCheckpoint(
  durationAdd, 
  0,
  actualCycle,
  newEndCycle - 1  
);
```
This will correct the ysCVG supply accounting when increasing lock duration.
