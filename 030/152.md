Powerful Chrome Orca

medium

# `claimMultipleStaking` function will cause asset loss when sdtRewardCount=0

## Summary

`claimMultipleStaking` function will cause asset loss when sdtRewardCount=0

## Vulnerability Detail

In function `claimMultipleStaking` from SdtRewardReceiver.sol, when parameter `sdtRewardCount` is equal to 0, it will not enter the `for (uint256 totalRewardIndex; totalRewardIndex < sdtRewardCount; ) { ... }` loop, thereby bypassing the check `require(totalRewardIndex != sdtRewardCount - 1, "REWARD_COUNT_TOO_SMALL");`, resulting in the `_sdtRewards` not being correctly counted. Eventually, funds were lost due to not doing a proper check on `sdtRewardCount`. It should be noted that although there is a comment `This parameter must be configured through the front-end.` for the parameter `sdtRewardCount`, considering that the attribute of the function is external, this risk still exists.

```solidity
function claimMultipleStaking(
        ISdtStakingPositionManager.ClaimSdtStakingContract[] calldata claimContracts,
        bool isConvert,
        bool isMint,
        uint256 sdtRewardCount
    ) external {
        // ...
        /// @dev Array merging & accumulating rewards coming from diferent claims.
        ICommonStruct.TokenAmount[] memory _totalSdtClaimable = new ICommonStruct.TokenAmount[](sdtRewardCount);

        // .......
        for (uint256 totalRewardIndex; totalRewardIndex < sdtRewardCount; ) {

            // ....

            require(totalRewardIndex != sdtRewardCount - 1, "REWARD_COUNT_TOO_SMALL");

            unchecked {
               ++totalRewardIndex;
            }
        }
        // ....
```

## Impact

loss of funds

## Code Snippet

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtRewardReceiver.sol#L119

## Tool used

Manual Review

## Recommendation

Add a check on the parameter `sdtRewardCount` to ensure that it cannot be equal to 0

```solidity
function claimMultipleStaking(
        ISdtStakingPositionManager.ClaimSdtStakingContract[] calldata claimContracts,
        bool isConvert,
        bool isMint,
        uint256 sdtRewardCount
    ) external {
        require(sdtRewardCount != 0, "ZERO_REWARD_COUNT"); // <----- Add

        /// @dev Checks for all positions input in data : Token ownership & verify positions are linked to the right staking service & verify timelocking
        sdtStakingPositionManager.checkMultipleClaimCompliance(claimContracts, msg.sender);

        /// @dev Accumulates amounts of CVG coming from all claims.
        uint256 _totalCvgClaimable;

        /// @dev Array merging & accumulating rewards coming from diferent claims.
        ICommonStruct.TokenAmount[] memory _totalSdtClaimable = new ICommonStruct.TokenAmount[](sdtRewardCount);
}
```