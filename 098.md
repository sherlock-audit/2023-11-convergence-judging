Howling Coffee Bird

medium

# The `tokenURI()` and `stakingInfo()` methods does not check if the NFT has been minted

## Summary
By invoking the `SdtStakingPositionManager.tokenURI()` and  `SdtStakingPositionService.stakingInfo()` methods for a maliciously provided NFT id, the returned data may deceive potential users, as the method will return data for a non-existent NFT id that appears to be genuine. 

## Vulnerability Detail
The [SdtStakingPositionManager.tokenURI()](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionManager.sol#L175-L184) and [SdtStakingPositionService.stakingInfo()](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L823-L840) methods lack any requirements stating that the provided NFT id must be created. 
```solidity

function stakingInfo(uint256 tokenId) public view returns (ISdtStakingPositionService.StakingInfo memory) {
        uint256 pending = _tokenInfoByCycle[stakingCycle + 1][tokenId].pendingStaked;

        (uint256 _cvgClaimable, ICommonStruct.TokenAmount[] memory _sdtRewardsClaimable) = getAllClaimableAmounts(
            tokenId
        );

        return (
            ISdtStakingPositionService.StakingInfo({
                tokenId: tokenId,
                symbol: symbol,
                pending: pending,
                totalStaked: tokenTotalStaked(tokenId) - pending,
                cvgClaimable: _cvgClaimable,
                sdtClaimable: _sdtRewardsClaimable
            })
        );
    }
```

We can also see that in the standard implementation by [OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/cf86fd9962701396457e50ab0d6cc78aa29a5ebc/contracts/token/ERC721/ERC721.sol#L94), this check is present:
[Throws if _tokenId is not a valid NFT](https://eips.ethereum.org/EIPS/eip-721)
This issue is similar to the one reported in https://github.com/code-423n4/2023-04-caviar-findings/issues/44.

## Impact
By invoking the `SdtStakingPositionManager.tokenURI()` and  `SdtStakingPositionService.stakingInfo()` methods for a maliciously provided NFT id, the returned data may deceive potential users, as the method will return data for a non-existent NFT id that appears to be genuine. This can lead to a poor user experience or financial loss for users.
Violation of the ERC721-Metadata part standard.

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L823-L840

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionManager.sol#L175-L184
## Tool used

Manual Review

## Recommendation
Throw an error if the NFT id is invalid.
