Active Obsidian Mule

high

# `balanceOfYsCvgAt` incorrectly retrieves the sum of all past cycles balances rather than just the balance at the given cycle

## Summary

`balanceOfYsCvgAt` incorrectly sums the balance of all past lock extensions rather than simply summing the balance of all extensions which overlap at the given `_cycleId`. Results in much more rewards being received by positions with many extensions, and can be manipulated to steal all rewards.

Note: The same vulnerability applies to `balanceOfMgCvgAt` too, and as such, this issue and its recommended mitigation also applies to that function.

## Vulnerability Detail

Given a `_cycleId`, `balanceOfYsCvgAt` should retrieve what the balance would be at that cycleId. Problem is it doesn't consider the extension's end cycle, effectively retrieving the balance for each extension as if the cycle never ended. This causes all past extensions' ysCvg balances to be summed regardless of whether they are active at the given cycle or not. 

As a result, since the total supply logic doesn't make this mistake, an attacker can create a bunch of minimum duration extensions in row such that the total ysCvg throughout each of them adds up to be ~100% of the total supply for the given cycle. Since YsDistributor distributes rewards according to relative balances, this allows the attacker to receive all the rewards for the cycle.

## Impact

Attacker can steal all rewards for cycles even while only having a very small fraction of locked cvg/duration at any given time.

## Code Snippet

`balanceOfYsCvgAt` fails to consider the end cycle of lock extensions.

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L668
```solidity
/** @dev We go through the extensions to compute the balance of ysCvg at the cycleId */
for (uint256 i; i < _extensions.length; ) {
    /** @dev Don't take into account the extensions if in the future. */
    // @audit don't we have to enforce that the _cycleId is < endCycle?
    if (_extensions[i].cycleId < _cycleId) {
        LockingExtension memory _extension = _extensions[i];
        uint256 _firstTdeCycle = TDE_DURATION * (_extension.cycleId / TDE_DURATION + 1);
        uint256 _ysTotal = (((_extension.endCycle - _extension.cycleId) *
            _extension.cvgLocked *
            _lockingPosition.ysPercentage) / MAX_PERCENTAGE) / MAX_LOCK;
        uint256 _ysPartial = ((_firstTdeCycle - _extension.cycleId) * _ysTotal) / TDE_DURATION;
        /** @dev For locks that last less than 1 TDE. */
        // @audit off by one as when equal to TDE duration we should get full amount
        if (_extension.endCycle - _extension.cycleId <= TDE_DURATION) {
            _ysCvgBalance += _ysPartial;
        } else {
            _ysCvgBalance += _cycleId <= _firstTdeCycle ? _ysPartial : _ysTotal;
        }
    }
    ++i;
}
return _ysCvgBalance;
```

## Tool used

Manual Review

## Recommendation

Only retrieve the ysCvgBalance for extensions whose endCycle >= `_cycleId`. e.g. instead of:

```solidity
if (_extensions[i].cycleId < _cycleId) {
```

use:

```solidity
if (_extensions[i].cycleId < _cycleId && _extensions[i].endCycle >= _cycleId) {
```