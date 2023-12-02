Innocent Scarlet Quail

high

# LockPositionDelegate doesn't clear delegates on transfer which can be used to honeypot buyers

## Summary

When transferring locked positions, the delegates are not cleared from the previous owner. This can be used to honeypot buyers. A malicious user would list their token which a large TDE claim available. They can then frontrun the legitimate user with a call setting themselves as the ysCVG delegate. Then after they can claim the TDE at the expense of the new owner who paid extra for the token since it was entitled to a TDE claim.

## Vulnerability Detail

[LockingPositionDelegate.sol#L237-L240](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionDelegate.sol#L237-L240)

    function delegateYsCvg(uint256 _tokenId, address _to) external onlyTokenOwner(_tokenId) {
        delegatedYsCvg[_tokenId] = _to;
        emit DelegateShare(_tokenId, _to);
    }

delegateYsCvg allows the owner of the token to set the receiver of yield sharing. It is also the only function that sets the delegatedYsCvg mapping. LockingPositionManager uses the standard implementation of transfers which never clears out this delegation when the token is transferred. As a result of this, all delegation will persist across transfers. This enables the honeypot scenario described above. 

## Impact

Users can be honeypotted by malicious token sellers

## Code Snippet

[SdtStakingPositionManager.sol#L21](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionManager.sol#L21)

## Tool used

Manual Review

## Recommendation

Overwrite the afterTokenTransfer method to forcefully clear delegation when a token is transferred