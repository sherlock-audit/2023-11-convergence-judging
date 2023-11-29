Precise Plum Shark

high

# Potential collision risk due to using `abi.encodePacked()` with multiple dynamic arguments in `tokenURI()` function.

## Summary
The code snippet is a function that returns the token URI for a given token ID. The function uses `abi.encodePacked()` with multiple dynamic arguments, which can cause a collision risk. A collision occurs when two different inputs produce the same output. This can lead to unexpected behavior and security issues.

## Vulnerability Detail
The function `tokenURI()` calls `abi.encodePacked()` with two dynamic arguments: `localBaseURI` and `Strings.toString(tokenId)`. According to the Solidity documentation, `abi.encodePacked()` does not pad dynamic types to 32 bytes. This means that the length of the output depends on the length of the inputs. For example, `abi.encodePacked("a", "bc")` produces the same output as `abi.encodePacked("ab", "c")`. This can cause a collision if two different token IDs have the same string representation after concatenating with the base URI.

## Impact
A collision can have serious consequences for the functionality and security of the contract. For example, if two different token IDs have the same token URI, then they will point to the same metadata and image. This can confuse the users and affect the value of the tokens. Moreover, a collision can also allow an attacker to exploit the contract logic and bypass some checks or validations.

## Code Snippet
Source Link:- https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionManager.sol#L88-L99

## Tool used
- Slither
- Manual Review

## Recommendation
To avoid the collision risk, do not use more than one dynamic type in `abi.encodePacked()`. Use `abi.encode()` instead, which pads dynamic types to 32 bytes and ensures a unique output for each input. Alternatively, you can also use a separator between the dynamic arguments, such as a slash (/) or a hash (#), to prevent them from merging. For example, you can change the code snippet to:
```solidity
function tokenURI(uint256 tokenId) public view virtual override returns (string memory) {
        _requireMinted(tokenId);

        ILockingLogo _logo = cvgControlTower.lockingLogo();
        if (address(_logo) == address(0)) {
            string memory localBaseURI = _baseURI();
            return
                bytes(localBaseURI).length > 0 ? string(abi.encode(localBaseURI, "/", Strings.toString(tokenId))) : "";
        }

        return _logo._tokenURI(logoInfo(tokenId));
    }

```