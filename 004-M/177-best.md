Innocent Scarlet Quail

medium

# Users who frequently increase lock balance will DOS themselves over time

## Summary

Each time a user adds tokens to their lock it pushes a new extension to the token. These extensions are looped through whenever ysCVG is calculated. For tokens that are frequently increased in balance, such as an active user or a CVX-like integration, this extension list will become too long and will trigger an OOP error when attempting to claim TDE rewards. This permanently breaks claiming which causes huge loss of yield for the token holder.

## Vulnerability Detail

[LockingPositionService.sol#L360-L371](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L360-L371)

        LockingExtension memory lockingExtension = LockingExtension({
            cycleId: actualCycle,
            endCycle: lockingPosition.lastEndCycle,
            cvgLocked: amount,
            mgCvgAdded: _newVotingPower
        });

        /** @dev Add a lock extension linked to the Amount Extension. */
        lockExtensions[tokenId].push(lockingExtension);

        /** @dev Transfer CVG from user wallet to here. */
        cvg.transferFrom(msg.sender, address(this), amount);

Each time CVG is added to the lock, an extension is appended to it. This array is looped through in it's entirety whenever ysCVG is calculated:

[LockingPositionService.sol#L668-L685](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L668-L685)

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

If the extensions array is too long then an OOG error will occur. When claiming, ysDistributor makes a call to this method in L184. As a result if this method is failing then it is impossible to claim TDE rewards for the token.

It can also be abused maliciously. Since tokens can be sold a malicious user could DOS their token on purpose then sell it as a honey pot. The value of the TDE claim would appear to be claimable and would therefore increase the value of the token. After the user buys the token they cannot claim the TDE.

## Impact

TDE claims can be DOS'd and affected tokens can be sold as honey pots

## Code Snippet

[LockingPositionService.sol#L439-L505
](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L439-L505)

## Tool used

Manual Review

## Recommendation

When an amount is added to a cycle that already has an extension, modify the existing extension instead of pushing a new one. By making that change it is nearly impossible for this to occur over any reasonable amount of time.