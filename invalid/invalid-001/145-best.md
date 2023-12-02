Atomic Velvet Nuthatch

medium

# Reentrancy Risk in YsDistributor during rewards claiming

## Summary
As said in the readme of the project : 
```Solidity
YsDistributor receives rewards from the treasury. The list of ERC20 can vary.
  Curve (CRV)
  Convex (CVX)
  StakeDao (SDT)
  Frax-Share (FXS)
  Prisma (PRISMA)
  (…)
  USDC
  USDT
  DAI
```
The list of ERC20 can vary and there is no restrictions using tokens not listed here in the future as rewards.
The `claimRewards` function in YSDistributor contract, which processes reward claims for locking position NFTs, show a risk of reentrancy. This risk arises from the absence of the Check-Effects-Interaction (CEI) pattern, particularly concerning when considering potential future integration with custom ERC20 tokens that enable reentrant calls

## Vulnerability Detail
`claimRewards` allows users to claim their rewards associated with a specific Token ID (`tokenId`) and Time-Dependent Event (`tdeId`). It calculates the user's share of rewards and then calls `_claimTokenRewards` to distribute them.
In `_claimTokenRewards`, the contract iterates over an array of token addresses and transfers the calculated reward amounts to the receiver. This transfer occurs without the protection of the CEI pattern or a reentrancy guard.
```solidity
function _claimTokenRewards(uint256 tokenId, uint256 tdeId, uint256 share, address receiver) internal {
    ...
    for (uint256 i; i < tokens.length; ) {
        IERC20 _token = IERC20(tokens[i]);
        uint256 _amountUser = _calculateUserRewardAmount(tdeId, _token, share);
        _token.safeTransfer(receiver, _amountUser);
        unchecked { ++i;}
    }
    ...
}
```
As you can see below the state of the function is updated after tokens have been distributed : 
```solidity
    function claimRewards(uint256 tokenId, uint256 tdeId, address receiver, address operator) external {
        // CODE 
        .....
        // CODE

        /// @dev Claim according token rewards.
        _claimTokenRewards(tokenId, tdeId, share, receiver);

        /// @dev Mark the TDE id for this token as claimed on the Storage.
        rewardsClaimedForToken[tokenId][tdeId] = true;
```
Currently, the function primarily interacts with standard ERC20 tokens. However, if the protocol integrates custom ERC20 tokens with hooks that can invoke external contracts before transferring funds, it could expose the function to reentrancy attacks by reentering the function with a TDE not claimed and drain the contract from all rewards accumulated.

## Impact

No ERC777 but the risk is still here if in the future Convergence is using a custom ERC20 that call user before sending funds so the impact would be critical with possible ERC777 interaction , here I think the risk is medium due to the fact that a malicious actors could exploit reentrancy vulnerabilities to drain funds in certain cases

## Code Snippet

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/YsDistributor.sol#L191

## Tool used

Manual Review

## Recommendation

Update the state before distributing rewards

```solidity
function claimRewards(uint256 tokenId, uint256 tdeId, address receiver, address operator) external {
    // CODE 
    .....
    // CODE
   
    /// @dev Mark the TDE id for this token as claimed on the Storage.
+   rewardsClaimedForToken[tokenId][tdeId] = true;

    /// @dev Claim according token rewards.
+   _claimTokenRewards(tokenId, tdeId, share, receiver);
```