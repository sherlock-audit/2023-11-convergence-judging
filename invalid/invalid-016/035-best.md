Joyous Fuzzy Cobra

medium

# Tokens with approval race protection or not returning a `bool` on `approve` might break token approvals.

## Summary
Certain tokens on might revert on approval and cause unexpected behaviour.

## Vulnerability Detail
Certain tokens, including [USDT](https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7), [KNC](https://etherscan.io/token/0xdd974d5c2e2928dea5f71b9825b8b646686bd200#code)  have an approval race protection mechanism in place, requiring the allowance to be set to either zero upon any update.
When the owner calls the `approveTokens` function with these kind of tokens in the array, the transactions revert and owner will not be able to approve the tokens. 
Also, some (USDT for instance) do not return a bool on approve call. Those tokens are incompatible with the protocol because Solidity will check the return data size, which will be zero and will lead to a revert.
Finally,  USDT approve will always revert due to the IERC20 interface mismatch.

## Impact
Token approval will be blocked and a host of other unexpected behaviours.
## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/utils/SdtUtilities.sol#L216
```SdtUtilities.sol
    function approveTokens(TokenSpender[] calldata _tokenSpenders) external onlyOwner {
        for (uint256 i; i < _tokenSpenders.length; ) {
            IERC20(_tokenSpenders[i].token).approve(_tokenSpenders[i].spender, _tokenSpenders[i].amount);
            unchecked {
                ++i;
            }
        }
    }
```
## Tool used

Manual Review

## Recommendation
Approve to zero first, use forceApprove from SafeERC20, or safeIncreaseAllowance.