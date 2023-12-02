Rhythmic Felt Lynx

medium

# Risk of Permanently Lost Rewards due to Claiming Before Treasury Deposit

## Summary
When Treasury Bond `depositMultipleToken()` in current TDE and users try to `claimRewards()` for current TDE, users may lose their rightful rewards.

## Vulnerability Detail
The mechanism responsible for TreasuryBonds depositing rewards into `YsDistributor.sol` through the `depositMultipleToken()` function calculates the TDE for reward distribution based on the current cvgCycle. Each cvgCycle spans a week, and `TDE_DURATION` equals 12 weeks or 12 cvgCycles. This calculation of `_actualTDE` results in two potential outcomes: the current TDE or the subsequent one.
```solidity
        uint256 _actualCycle = cvgControlTower.cvgCycle();
        uint256 _actualTDE = _actualCycle % TDE_DURATION == 0
            ? _actualCycle / TDE_DURATION
            : (_actualCycle / TDE_DURATION) + 1;
```
The issue arises when users use `claimRewards()` for the current TDE before the execution of `depositMultipleToken()` within the same block. If, coincidentally, the TDE calculation lands on the current TDE and `claimRewards()` is executed first, users risk permanently losing their rewards. Once a specific TDE id is claimed, users are unable to make further claims, effectively trapping their rewards within the `YsDistributor.sol` contract.
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
        );

      ->uint256 cycleClaimed = tdeId * TDE_DURATION;

        require(_lockingPositionService.lockingPositions(tokenId).lastEndCycle >= cycleClaimed, "LOCK_OVER");

      ->require(_cvgControlTower.cvgCycle() >= cycleClaimed, "NOT_AVAILABLE");

      ->require(!rewardsClaimedForToken[tokenId][tdeId], "ALREADY_CLAIMED");

         and the totalSupply of ysCvg at the specified TDE. */
        uint256 share = (_lockingPositionService.balanceOfYsCvgAt(tokenId, cycleClaimed) * 10 ** 20) /
            _lockingPositionService.totalSupplyYsCvgHistories(cycleClaimed);

        require(share != 0, "NO_SHARES");

        _claimTokenRewards(tokenId, tdeId, share, receiver);

        rewardsClaimedForToken[tokenId][tdeId] = true;
    }
```
## Impact
The rewards are essentially locked in the contract if `claimRewards()` is executed before `depositMultipleToken()` in the same block, resulting in users being unable to claim their rightful rewards.

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/YsDistributor.sol#L101
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/YsDistributor.sol#L155
## Tool used

Manual Review

## Recommendation
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
        );

        uint256 cycleClaimed = tdeId * TDE_DURATION;

        require(_lockingPositionService.lockingPositions(tokenId).lastEndCycle >= cycleClaimed, "LOCK_OVER");

-       require(_cvgControlTower.cvgCycle() >= cycleClaimed, "NOT_AVAILABLE");
+       require(_cvgControlTower.cvgCycle() > cycleClaimed, "NOT_AVAILABLE");

        require(!rewardsClaimedForToken[tokenId][tdeId], "ALREADY_CLAIMED");

         and the totalSupply of ysCvg at the specified TDE. */
        uint256 share = (_lockingPositionService.balanceOfYsCvgAt(tokenId, cycleClaimed) * 10 ** 20) /
            _lockingPositionService.totalSupplyYsCvgHistories(cycleClaimed);

        require(share != 0, "NO_SHARES");

        _claimTokenRewards(tokenId, tdeId, share, receiver);

        rewardsClaimedForToken[tokenId][tdeId] = true;
    }
```
