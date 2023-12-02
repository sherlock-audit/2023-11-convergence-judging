Proper Lemonade Sealion

medium

# Division difference can result in a revert when claiming treasury yield and excess rewards to some users

## Summary
Different ordering of calculations are used to compute `ysTotal` in different situations. This causes the totalShares tracked to be less than the claimable amount of shares  

## Vulnerability Detail
`ysTotal` is calculated differently when adding to `totalSuppliesTracking` and when computing `balanceOfYsCvgAt`.
When adding to `totalSuppliesTracking`, the calculation of `ysTotal` is as follows:

```solidity
        uint256 cvgLockAmount = (amount * ysPercentage) / MAX_PERCENTAGE;
        uint256 ysTotal = (lockDuration * cvgLockAmount) / MAX_LOCK;
```

In `balanceOfYsCvgAt`, `ysTotal` is calculated as follows

```solidity
        uint256 ysTotal = (((endCycle - startCycle) * amount * ysPercentage) / MAX_PERCENTAGE) / MAX_LOCK;
```

This difference allows the `balanceOfYsCvgAt` to be greater than what is added to `totalSuppliesTracking`

### POC
```solidity
  startCycle 357
  endCycle 420
  lockDuration 63
  amount 2
  ysPercentage 80
```

Calculation in `totalSuppliesTracking` gives:
```solidity
        uint256 cvgLockAmount = (2 * 80) / 100; == 1
        uint256 ysTotal = (63 * 1) / 96; == 0
```
Calculation in `balanceOfYsCvgAt` gives:
```solidity
        uint256 ysTotal = ((63 * 2 * 80) / 100) / 96; == 10080 / 100 / 96 == 1
```

### Example Scenario
Alice, Bob and Jake locks cvg for 1 TDE and obtains rounded up `balanceOfYsCvgAt`. A user who is aware of this issue can exploit this issue further by using `increaseLockAmount` with small amount values by which the total difference difference b/w the user's calculated `balanceOfYsCvgAt` and the accounted amount in `totalSuppliesTracking` can be increased. Bob and Jake claims the reward at the end of reward cycle. When Alice attempts to claim rewards, it reverts since there is not enough reward to be sent.

## Impact
This breaks the shares accounting of the treasury rewards. Some user's will get more than the actual intended rewards while the last withdrawals will result in a revert

## Code Snippet
`totalSuppliesTracking` calculation

In `mintPosition`
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L261-L263

In `increaseLockAmount`
https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L339-L345

In `increaseLockTimeAndAmount`
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L465-L470

`_ysCvgCheckpoint`
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L577-L584

`balanceOfYsCvgAt` calculation
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L673-L675

## Tool used

Manual Review

## Recommendation
Perform the same calculation in both places

```diff
+++                     uint256 _ysTotal = (_extension.endCycle - _extension.cycleId)* ((_extension.cvgLocked * _lockingPosition.ysPercentage) / MAX_PERCENTAGE) / MAX_LOCK;
---     uint256 ysTotal = (((endCycle - startCycle) * amount * ysPercentage) / MAX_PERCENTAGE) / MAX_LOCK;
```