Active Obsidian Mule

high

# `lockExtension` not update/pushed for `increaseLockTime`, results in missed ysCvg and locked yield

## Summary

LockingPositionService.increaseLockTime fails to update or push a new lockExtension, causing ysCvg cycle balance retrieval logic to fail, leading to user not receiving rewards and reward tokens being permanently locked.

## Vulnerability Detail

For all lock position updates in LockingPositionService, a lockExtension should be pushed to the `lockingExtensions[tokenId]` array. These lock extensions are vital for tracking the ysCvg balance of the position, as retrieved by `balanceOfYsCvgAt`. 

`balanceOfYsCvgAt` is used by the YsDistributor to determine the share of rewards which a given tokenId should receive at a given cycle. Since balance is not correctly tracked due to missing lockExtension, users will not receive rewards for added duration from `increaseLockTime`.

Furthermore, since `increaseLockTime` updates the `totalSuppliesTracking` accordingly to the duration being added, the sum of all ysCvg balances will not equal the total supply for these cycles. Since there's no other way to withdraw deposited rewards from YsDistributor, this inconsistency will lead to reward tokens being permanently locked in the contract.

## Impact

Loss of rewards for users using `increaseLockTime` and reward tokens being permanently locked in the `YsDistributor` contract.

## Code Snippet

`balanceOfYsCvgAt` retrieves ysCvg balance at a given cycle by iterating over lock extensions

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L656
```solidity
/**
 * @notice Fetch the balance of ysCVG (treasury share)  for a specified tokenId and at a specified cycle, can be in the future.
 * @param _tokenId id of the token
 * @param _cycleId id of the cycle
 */
function balanceOfYsCvgAt(uint256 _tokenId, uint256 _cycleId) public view returns (uint256) {
    require(_cycleId != 0, "NOT_EXISTING_CYCLE");

    LockingPosition memory _lockingPosition = lockingPositions[_tokenId];
    LockingExtension[] memory _extensions = lockExtensions[_tokenId];
    uint256 _ysCvgBalance;

    /** @dev If the requested cycle is before or after the lock , there is no balance. */
    if (_lockingPosition.startCycle >= _cycleId || _cycleId > _lockingPosition.lastEndCycle) {
        return 0;
    }
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
}
```

## Tool used

Manual Review

## Recommendation

Push a lockExtension for the given duration that the lock is increased by.