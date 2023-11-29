Bubbly Ruby Rattlesnake

medium

# Users will permanently lose rewards, if they burn their lock position before claiming rewards

## Summary
Users who lock their positions for rewards may lose these rewards, if they close (burn) their position before rewards are claimed. After a position is closed (burnt), the `tokenId` related to it will no longer point to a valid owner address, which is a requirement to be able to claim rewards.

## Vulnerability Detail
Users Lock their positions and accrue rewards related to it. After users want to retrieve their `cvg` back and close position (when the lock time is over), users can do this using function [burnPosition](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L511) in `LockingPositionService.sol`. The position is represented by an ERC721-like contract, which `burn` and `ownerOf` is inherited from pure ERC721. 

In order to claim rewards, users should use function [claimRewards](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/YsDistributor.sol#L155) in `YsDistributor.sol`. 

If by a mistake users burn their position, but there are still pending rewards, they will not be able to retrieve them afterwards. This is because the `claimRewards` function retrieves a position by `tokenId`, which after burning, does not point to a valid owner. 

The rewards claim can be called directly by user, or from `CvgUtilities.sol`.
In first case, it will revert:

```solidity

        if (msg.sender != _cvgControlTower.cvgUtilities()) {
            operator = msg.sender;
        }
        require(
            operator == _lockingPositionManager.ownerOf(tokenId) ||
                operator == lockingPositionDelegate.delegatedYsCvg(tokenId),
            "NOT_OWNED_OR_DELEGATEE"
        );

```

Because operator will be msg.sender, and ownerOf(tokenId) will be zero after burn.

In second case,

```solidity

            for (uint256 j; j < claimTdes[i].tdeIds.length; ) {
                ysDistributor.claimRewards(
                    tokenId,
                    claimTdes[i].tdeIds[j],
                    msg.sender,
                    msg.sender
                );

```

It will be the same, since `msg.sender` will be in fact the caller, and the same issue will happen. It can only by avoided, if an user had a delegate before, which probably is not a common case. So for an user who acts just without a delegate, burning a position will lock their related rewards forever.




## Impact
Some users will lose their unclaimed rewards if they burn their locking positions too early.

## Code Snippet

```solidity

    function claimRewards(uint256 tokenId, uint256 tdeId, address receiver, address operator) external {
        ICvgControlTower _cvgControlTower = cvgControlTower;
        ILockingPositionService _lockingPositionService = lockingPositionService;
        ILockingPositionManager _lockingPositionManager = lockingPositionManager;
        require(_lockingPositionManager.unlockingTimestampPerToken(tokenId) < block.timestamp, "TOKEN_TIMELOCKED");

        if (msg.sender != _cvgControlTower.cvgUtilities()) {
            operator = msg.sender;
        }
        require(
            operator == _lockingPositionManager.ownerOf(tokenId) ||
                operator == lockingPositionDelegate.delegatedYsCvg(tokenId),
            "NOT_OWNED_OR_DELEGATEE"
        );//@audit for burned token it will not work anymore

```

## Tool used

Manual Review

## Recommendation
Force claim rewards on a burnt position or cache the last owner in a separate variable e.g. in `struct LockingPosition`, other than that claim should be possible since the mappings are not cleared on burn.
