Raspy Corduroy Raven

medium

# `LockingPositionService.mintPosition()` problems with duplicate `tokenIds` from `LockingPositionManager` may cause users to lose their funds

## Summary

The current implementation and usage of the `LockingPositionManager` contract inside `LockingPositionService` may cause users to lose their funds under certain conditions when they are locking a new position by calling `LockingPositionService.mintPosition()`. This may happen due to duplicate `tokenIds` if the `LockingPositionManager` is changed in the `CvgControlTower`, causing the `LockingPositionService` to overwrite existing locking positions.

## Vulnerability Detail

When `LockingPositionService.mintPosition()` is called, the `tokenId` for the new position is determined by calling `_lockingPositionManager.nextId();` (line 257 LockingPositionService.sol). It's important to note that `_lockingPositionManager` is not a storage variable, but instead it is dynamically fetched by calling `CvgControlTower.lockingPositionManager()`. That means that whenever the state var `lockingPositionManager` inside the `CvgControlTower` is changed by a call to `CvgControlTower.setLockingPositionManager()`, the address for the `LockingPositionManager` will as well be different and `_lockingPositionManager.nextId();` (line 257 LockingPositionService.sol) will also be different and may return a `tokenId` that is already in use by another user.

Example:

1. Alice is minting a new locking position, calling `LockingPositionService.mintPosition()` and locking an `amount` of 10000 CVG into this new position. The tokenId of her position will be 1 (see line 45, 60 LockingPositionManager.sol)
1. If another user would then mint another locking position, they would receive `tokenId` with the value 2 from the `LockingPositionManager.nextId()`, since the `tokenId` is always incremented by 1 (line 60 LockingPositionManager.sol)
1. However Governance sets a new `LockingPositionManager` by calling `CvgControlTower.setLockingPositionManager()`
1. Then Bob is minting a new locking position, calling `LockingPositionService.mintPosition()` with `ysPercentage` value of `MAX_PERCENTAGE`, and locking an `amount` of 10 CVG into this new position. The `tokenId` that is fetched from `LockingPositionManager.nextId()` will have a value of 1 again which is the same `tokenId` value from Alice's locking position, since `LockingPositionManager` was changed. Alice's locking position will be overwritten (line 293 LockingPositionService.sol) with the new lockingPosition from Bob, thus Alice will lose 10000 CVG because her position and her amount (`totalCvgLocked`) got overwritten by Bob's position.

The issue is that even if Governance would try to help Alice by setting the old `LockingPositionManager`, Alice's 10000 CVG would still be lost, because her amount was overwritten by Bob's `lockingPosition` on line 293 in LockingPositionService.sol. So Alice would end up with Bob's amount which is only 10 CVG, so Alice would have lost 9990 CVG even if Governance would have tried to help her in this example.

Another issue is that the `lockingExtension` from Bob's locked position would be pushed into the existing `lockExtensions[tokenId]` which is not empty since it already contains an entry from Alice's locking extension before. As a consequence, when mgCvg balance is evaluated for Bob, it will use Alices and Bobs locking extension, which means Bob's balance of mgCvg will be higher than it should be.

## Impact

Users may lose their funds if `LockingPositionManager` is being updated as illustrated in the example above. Also Governance's attempt to help by setting back to the old `LockingPositionManager` won't solve this issue as shown above.

## Code Snippet

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L231-L312

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionManager.sol#L39-L47

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionManager.sol#L59-L61

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/CvgControlTower.sol#L329-L331

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L247-L249

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L276-L283

## Tool used

Manual Review

## Recommendation

Consider adjusting the `LockingPositionService` contract to work similar to the `SdtStakingPositionService` contract:

- When it's initialized, the `SdtStakingPositionService` is fetching the `sdtStakingPositionManager` from `CvgControlTower` and assigns it to its storage variable `sdtStakingPositionManager` (line 247, 249 SdtStakingPositionService.sol). Then it uses the stored `sdtStakingPositionManager` inside its `deposit` function (line 283 SdtStakingPositionService.sol). Thus avoiding similar issues as were described in this report.

In the same way the `LockingPositionService` can be adjusted so that inside it's `initialze` function the `CvgControlTower.lockingPositionManager()` is assigned to a state variable which can then be used in the contract:

```solidity
// LockingPositionService

114    function initialize(ICvgControlTower _cvgControlTower) external initializer {
115        cvgControlTower = _cvgControlTower;
116        _transferOwnership(msg.sender);
117        ICvg _cvg = _cvgControlTower.cvgToken();
118        require(address(_cvg) != address(0), "CVG_ZERO");
119        cvg = _cvg;
+ 120      ILockingPositionManager _lockingPositionManager = _cvgControlTower.lockingPositionManager();
+ 121      require(address(_lockingPositionManager) != address(0), "LOCKING_POSITION_MANAGER_ZERO");
+ 122      lockingPositionManager = _lockingPositionManager; // @audit: assign to state var  
```