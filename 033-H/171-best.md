Curved Hotpink Dragonfly

high

# Bad users can bypass the staking time limit to obtain YsDistributor rewards

## Summary

Bad users can mint positions just before the end of cvgCycle and make profits immediately when the next cvgCycle starts, in a very short time

## Vulnerability Detail
Calculation method for obtaining reward share:
https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Rewards/YsDistributor.sol#L184-L185
```solidity
        uint256 share = (_lockingPositionService.balanceOfYsCvgAt(tokenId, cycleClaimed) * 10 ** 20) /
            _lockingPositionService.totalSupplyYsCvgHistories(cycleClaimed);

    function _calculateUserRewardAmount(uint256 _tdeId, IERC20 _token, uint256 _share) internal view returns (uint256) {
        return (depositedTokenAmountForTde[_tdeId][_token] * _share) / 10 ** 20;
    }
```
The function balanceOfYsCvgAt to calculate the reward share is as follows:
```solidity
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
            if (_extensions[i].cycleId < _cycleId) {
                LockingExtension memory _extension = _extensions[i];
                uint256 _firstTdeCycle = TDE_DURATION * (_extension.cycleId / TDE_DURATION + 1);
                uint256 _ysTotal = (((_extension.endCycle - _extension.cycleId) *
                    _extension.cvgLocked *
                    _lockingPosition.ysPercentage) / MAX_PERCENTAGE) / MAX_LOCK;
                uint256 _ysPartial = ((_firstTdeCycle - _extension.cycleId) * _ysTotal) / TDE_DURATION;
                /** @dev For locks that last less than 1 TDE. */
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

Users only need endcycle > cycleId to obtain a cycle (7 days) reward. The path for users to make profits: Before the end of a TDE cycle, for example, now cycle = 11, when cycle = 11 is about to end, the user starts a position, starting cycle 11, ending cycle 12, ysPercentage 100%. When cycle = 12, users can get benefits. The income acquisition situation is affected by the reward storage situation.

From the code below, we can see that rewards can only be deposited in advance, such as depositing when cycle = 11, or depositing when cycle = 12.
https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Rewards/YsDistributor.sol#L102-L105
```solidity
        uint256 _actualCycle = cvgControlTower.cvgCycle();
        uint256 _actualTDE = _actualCycle % TDE_DURATION == 0
            ? _actualCycle / TDE_DURATION
            : (_actualCycle / TDE_DURATION) + 1;
```

As a result, there are two types of reward deposit situations:
1. The rewards have been deposited in advance, and users can withdraw their income immediately. As for the stimulation of obtaining income in a very short time, users may obtain CVG through the loan platform to maximize their income. Other users’ profits are squeezed
2. The reward has not been deposited yet
Deposits must also be made in cycle 12, as rewards cannot be deposited for ted that have already passed. Bad users just have to wait to reap the benefits. Due to time costs, users may benefit less, depending on when the rewards are deposited.


## Impact

Bad users have the opportunity to obtain large amounts of revenue in a very short period of time, and the impact should be high

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L656-L687

https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Rewards/YsDistributor.sol#L184-L185

## Tool used

Manual Review

## Recommendation

In order to mitigate the impact, the profit share can be calculated based on the time position start time and end time.
