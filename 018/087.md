Low Bamboo Rhino

medium

# Missing CVG Rewards Processing Status Check

## Summary

The `claimCvgRewards()` function in the `SdtStakingPositionService` contract does not include a check for the `_cycleInfo[lastClaimedCycle].isCvgProcessed` flag. This omission can  claiming rewards before they have been processed.

## Vulnerability Detail

The `claimCvgRewards()` function allows users to claim CVG rewards for a specific staking position. However, it does not check whether the CVG rewards for the corresponding cycle have been processed `(_cycleInfo[lastClaimedCycle].isCvgProcessed == true)`.



## Impact

Premature Claims: In the absence of the check, users can claim CVG rewards before they have been processed, leading to incorrect reward distribution and potential economic losses.


## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L338-L386

```solidity
function claimCvgRewards(uint256 tokenId) external checkCompliance(tokenId) {
    uint128 actualCycle = stakingCycle;

    uint128 lastClaimedCvg = _lastClaims[tokenId].lastClaimedCvg;
    uint128 lastClaimedSdt = _lastClaims[tokenId].lastClaimedSdt;
    uint128 lastClaimedCycle = lastClaimedCvg < lastClaimedSdt ? lastClaimedSdt : lastClaimedCvg;
    uint256 lengthHistory = _stakingHistoryByToken[tokenId].length;

    if (lastClaimedCycle == 0) {
        lastClaimedCycle = uint128(_stakingHistoryByToken[tokenId][0]);
    }

    require(actualCycle > lastClaimedCycle, "ALL_CVG_CLAIMED_FOR_NOW");

    uint256 _totalAmount;
    for (; lastClaimedCycle < actualCycle; ) {
        uint256 tokenStaked = _stakedAmountEligibleAtCycle(lastClaimedCycle, tokenId, lengthHistory);
        uint256 claimableAmount;

        if (tokenStaked != 0) {
            if (lastClaimedCvg <= lastClaimedCycle && _cycleInfo[lastClaimedCycle].isCvgProcessed) {
                claimableAmount = (tokenStaked * _cycleInfo[lastClaimedCycle].cvgRewardsAmount) / _cycleInfo[lastClaimedCycle].totalStaked;
                _totalAmount += claimableAmount;
            }
        }

        unchecked {
            ++lastClaimedCycle;
        }
    }

    require(_totalAmount > 0, "NO_CVG_TO_CLAIM");

    _lastClaims[tokenId].lastClaimedCvg = actualCycle;

    cvg.mintStaking(msg.sender, _totalAmount);

    emit ClaimCvgMultiple(tokenId, msg.sender);
}
```
## Tool used

Manual Review

## Recommendation


It is recommended to add a check for `_cycleInfo[lastClaimedCycle].isCvgProcessed == true` in the claimCvgRewards function.


```diff
function claimCvgRewards(uint256 tokenId) external checkCompliance(tokenId) {
    uint128 actualCycle = stakingCycle;

    uint128 lastClaimedCvg = _lastClaims[tokenId].lastClaimedCvg;
    uint128 lastClaimedSdt = _lastClaims[tokenId].lastClaimedSdt;
    uint128 lastClaimedCycle = lastClaimedCvg < lastClaimedSdt ? lastClaimedSdt : lastClaimedCvg;
    uint256 lengthHistory = _stakingHistoryByToken[tokenId].length;

    if (lastClaimedCycle == 0) {
        lastClaimedCycle = uint128(_stakingHistoryByToken[tokenId][0]);
    }

    require(actualCycle > lastClaimedCycle, "ALL_CVG_CLAIMED_FOR_NOW");

    uint256 _totalAmount;
    for (; lastClaimedCycle < actualCycle; ) {
        uint256 tokenStaked = _stakedAmountEligibleAtCycle(lastClaimedCycle, tokenId, lengthHistory);
        uint256 claimableAmount;

        if (tokenStaked != 0) {
+            if (lastClaimedCvg <= lastClaimedCycle && _cycleInfo[lastClaimedCycle].isCvgProcessed) {
                claimableAmount = (tokenStaked * _cycleInfo[lastClaimedCycle].cvgRewardsAmount) / _cycleInfo[lastClaimedCycle].totalStaked;
                _totalAmount += claimableAmount;
            }
        }

        unchecked {
            ++lastClaimedCycle;
        }
    }

    require(_totalAmount > 0, "NO_CVG_TO_CLAIM");

    _lastClaims[tokenId].lastClaimedCvg = actualCycle;

    cvg.mintStaking(msg.sender, _totalAmount);

    emit ClaimCvgMultiple(tokenId, msg.sender);
}
```