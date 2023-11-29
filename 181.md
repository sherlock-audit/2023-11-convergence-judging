Innocent Scarlet Quail

medium

# SdtRewardReceiver#setPoolCvgSdtAndApprove fails to clear past approvals which can leave dangerous hanging approvals

## Summary

When setting a new pool to be used for swapping, the contract doesn't remove the SDT approval to the old pool. This leaves a dangerous hanging approval. If a previously approved pool is compromised then that hanging approval could be used to drain the contract.

## Vulnerability Detail

[SdtRewardReceiver.sol#L253-L257](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtRewardReceiver.sol#L253-L257)

    function setPoolCvgSdtAndApprove(ICrvPoolPlain _poolCvgSDT, uint256 amount) external onlyOwner {
        poolCvgSDT = _poolCvgSDT;
        sdt.approve(address(_poolCvgSDT), amount);
        sdt.approve(address(cvgSdt), amount);
    }

We see above that when setting a new pool, the approval is made to the new pool but the old approval is never cleared. This leaves a hanging SDT approval to that old pool. In the event that that pool is compromised it would still be able to drain the contract of all SDT, even if the pool is removed.

## Impact

Dangerous hanging approvals that could potentially be exploited to steal SDT

## Code Snippet

[SdtRewardReceiver.sol#L253-L257](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtRewardReceiver.sol#L253-L257)

## Tool used

Manual Review

## Recommendation

Clear approval to old pool before overwriting it:
    
        function setPoolCvgSdtAndApprove(ICrvPoolPlain _poolCvgSDT, uint256 amount) external onlyOwner {
    ++      sdt.approve(address(poolCvgSDT), 0);
            poolCvgSDT = _poolCvgSDT;
            sdt.approve(address(_poolCvgSDT), amount);
            sdt.approve(address(cvgSdt), amount);
        }