Digital Peanut Sidewinder

medium

# Use safeTransfer/safeTransferFrom instead of transfer/transferFrom

## Summary

## Vulnerability Detail
The ERC20 token standard includes transfer and transferFrom functions that are used to move tokens between accounts. However, these functions do not require a return value according to the original ERC20 standard. This can lead to discrepancies in token transfer implementations, where some tokens return a boolean value and others do not return anything. If a contract assumes that these functions will revert on failure without checking the return value, it may incorrectly assume that a transfer was successful when it was not.
## Impact
It is good to add a require() statement that checks the return value of token transfers or to use something like OpenZeppelin’s safeTransfer/safeTransferFrom unless one is sure the given token reverts in case of a failure. Failure to do so will cause silent failures of transfers and affect token accounting in contract
## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/CvgSdtBuffer.sol#L136
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/CvgSdtBuffer.sol#L142
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBlackHole.sol#L86
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBlackHole.sol#L119
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBlackHole.sol#L124


## Tool used

Manual Review

## Recommendation
Consider implementing the use of safeTransfer and safeTransferFrom functions from OpenZeppelin's SafeERC20 library.