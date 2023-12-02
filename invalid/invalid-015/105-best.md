Precise Snowy Eagle

medium

# Use _safeMint() instead of _mint()

## Summary
OpenZeppelin recommends the usage of _safeMint() instead of _mint(). If the recipient is a contract, safeMint() checks whether they can handle ERC721 tokens.
https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/e10c70d922c30c3cd09777b868cbd9a892bb57f8/contracts/token/ERC721/ERC721Upgradeable.sol#L302
StdStakingPositionManager.sol  uses` _mint()` instead of `_safeMint()`, which can result in stuck tokens

## Vulnerability Detail
https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionManager.sol#L145
In the `deposit()` method in StdStakingPositionService.sol  calls `StdStakingPositionManager.mint()`  to mint the ERC721 token to the receiver. The problem is that if the receiver was a smart contract that does not handle ERC721 tokens, then this newly minted  token will be stuck.

## Impact
If you use `_mint()` and the contract recipient of the NFT its not prepared the NFT could be locked forever inside the contract.

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionManager.sol#L145

https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Locking/LockingPositionManager.sol#L60
## Tool used

Manual Review

## Recommendation
Use the` _safeMint` method instead of `_mint`.
In addition ，` _safeMint()`  might add a reentrancy issue, so add the nonReentrant modifier.

