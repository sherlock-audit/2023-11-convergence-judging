Spare Leather Puma

medium

# Killing a gauge will result in a user's votes being unusable for up to 10 days

## Summary
A user may have his votes unusable for up to 10 days, upon a gauge's removal

## Vulnerability Detail
Within the GaugeController, users vote for the gauge of their choice. They allocate a power of their voting power which is based on their voting escrow. 
```solidity
@internal
def vote_for_gauge_weights(tokenId: uint256, _gauge_addr: address, _user_weight: uint256):
    """
    @notice Assign/update voting power to a gauge to add to its weight, and drag more/less inflation onto it.
    @dev For a killed gauges on.
    @param _gauge_addr Gauge which `msg.sender` votes for
    @param _user_weight Weight for a gauge in bps (units of 0.01%). Minimal is 0.01%. Ignored if 0
    """
    assert not self.isLock, "VOTE_LOCKED"
    assert not self.killed_gauges[_gauge_addr] or _user_weight == 0 , "KILLED_GAUGE"
    assert self.vote_activated[_gauge_addr], "VOTE_GAUGE_PAUSED"

    lockingManager: LockingPositionManager = self.control_tower.lockingPositionManager()
    lockingDelegate: LockingPositionDelegate = self.control_tower.lockingPositionDelegate()

    # Check if the sender is the owner or a delegatee for the token.
    assert (lockingManager.ownerOf(tokenId) == msg.sender or lockingDelegate.delegatedVeCvg(tokenId) == msg.sender), "TOKEN_NOT_OWNED"
    # Check whether the token is time-locked: a token can be time-locked by its owner to protect a potential buyer from a malicious front run.
    assert (lockingManager.unlockingTimestampPerToken(tokenId) < block.timestamp), "TOKEN_TIMELOCKED"
    escrow: VotingPowerEscrow = self.control_tower.votingPowerEscrow()
    slope: uint256 = convert(escrow.get_last_nft_slope(tokenId), uint256)
    lock_end: uint256 = escrow.locked__end(tokenId)
    _n_gauges: int128 = self.n_gauges
    next_time: uint256 = (block.timestamp + WEEK) / WEEK * WEEK
    assert lock_end > next_time, "Your token lock expires too soon"
    assert (_user_weight >= 0) and (_user_weight <= 10000), "You used all your voting power"
    assert block.timestamp >= self.last_nft_vote[tokenId][_gauge_addr] + WEIGHT_VOTE_DELAY, "Cannot vote so often"  // @audit - important line 
```
In order to not spam-change their votes, users have a WEIGHT_VOTE_DELAY of 10 days. Meaning they cannot change their vote towards a gauge for 10 days.

However, there's an admin function, `kill_gauge`, allowing the admins to kill a gauge. Upon killing a gauge users can remove their votes from said gauge. However, in order to do so, they still have to go through the 10-day timelock, thus meaning their votes are locked and unused/unusable for up to 10 days.
```vyper
    assert block.timestamp >= self.last_nft_vote[tokenId][_gauge_addr] + WEIGHT_VOTE_DELAY, "Cannot vote so often"
``` 

## Impact
Users voting power may be unusable for up to 10 days

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/GaugeController.vy#L640

## Tool used

Manual Review

## Recommendation
Change the check on #L640 to the following
```vyper
    assert self.killed_gauges[_gauge_addr] or block.timestamp >= self.last_nft_vote[tokenId][_gauge_addr] + WEIGHT_VOTE_DELAY, "Cannot vote so often"
```