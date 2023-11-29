Attractive Pineapple Halibut

medium

# When cvgCycle is incremented shortly after or at thursday midnight of TDE there is a possibility to mint big amount of veCVG and burn them shortly after

## Summary
When cvgCycle is incremented shortly after or right at thursday midnight of TDE there is a possibility to mint big amount of veCVG and burn them shortly after. Unlock time of veCVG is alsways set as the next thursday midnight of mint. When we mint veCVG in LockingPositionService.mintPosition with durationAdd == 0 shortly before the thursday midnight we can generate a big amount of veCVG and burn them when cvgCycle in incremented. 

## Vulnerability Detail

The unlock time of minted veCVG is always set as the next thursday midnight in `create_lock` in `veCVG.py`

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/veCVG.vy#L328

We are able to call `LockingPositionService.mintPosition` function with `lockDuration == 0` if the current `cvgCycle % TDE_DURATION == 0`, although we won't mint any ysCVG or mgCVG we will be able to mint `amount` of veCVG with such parameters. 

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L252

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L270-L273

If `cvgCycle` is incremented shortly after or right at the thursday midnight we will be able to burn previously minted veCVG right at the moment of the increment. At the midnight the `unlock_time` of veCVG will be valid for burn and after the increment the current cycle will be more than `endLockCycle`, thus allowing to `burnPosition`.

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/veCVG.vy#L404

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L518


## Impact

With such possibility we could perform a flash loan attack to generate a big amounts of veCVG in a short time. Having a big amount of veCVG could be used to maliciously affect protocol voting.

## Code Snippet

```solidity
mintPosition(0, <big amount of CVG from a flashloan>, 100, msg.sender, true) // shortly before the thursday midnight of TDE
<doing something malicious with the obtrained veCVG>
burnPosition(tokenId) // right after the increment of cvgCycle
```

## Tool used

Manual Review

## Recommendation

To disallow minting of veCVG without appopriate locking time we could add checks for `lastEndCycle > actualCycle` as in `LockingPositionService.increaseLockAmount` in other locking methods: `increaseLockTime` and `increaseLockTimeAndAmount`. Similary we should add check for `lockDuration > 0` in `LockingPositionService.mintPosition`.
