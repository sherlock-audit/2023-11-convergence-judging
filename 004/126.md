Sticky Chambray Bobcat

high

# LockingPositionDelegate::manageOwnedAndDelegated unchecked duplicate tokenId allow metaGovernance manipulation

## Summary

A malicious user can multiply his share of meta governance delegation for a tokenId by adding that token multiple times when calling `manageOwnedAndDelegated`

## Vulnerability Detail

Without checks to prevent the addition of duplicate token IDs, a user can artificially inflate their voting power and their metaGovernance delegations. 

A malicious user can add the same tokenId multiple times, and thus multiply his own share of meta governance delegation with regards to that tokenId.

Scenario:

1. Bob delegates a part of metaGovernance to Mallory - he allocates 10% to her and 90% to Alice.
2. Mallory calls `manageOwnedAndDelegated` and adds the same `tokenId` 10 times, each time allocating 10% of the voting power to herself.
3. Mallory now has 100% of the voting power for the `tokenId`, fetched by calling `mgCvgVotingPowerPerAddress`, harming Bob and Alice metaGovernance voting power.

## Impact

The lack of duplicate checks can be exploited by a malicious user to manipulate the metaGovernance system, allowing her to gain illegitimate voting power (up to 100%) on a delegated tokenId, harming the delegator and the other delegations of the same `tokenId`.

## Code Snippet

```solidity
    function manageOwnedAndDelegated(OwnedAndDelegated calldata _ownedAndDelegatedTokens) external {
        ...

❌      for (uint256 i; i < _ownedAndDelegatedTokens.owneds.length;) { //@audit no duplicate check
        ...
        }

❌      for (uint256 i; i < _ownedAndDelegatedTokens.mgDelegateds.length;) { //@audit no duplicate check
        ...
        }

❌      for (uint256 i; i < _ownedAndDelegatedTokens.veDelegateds.length;) { //@audit no duplicate check
        ...
        }
    }

```

```solidity
function mgCvgVotingPowerPerAddress(address _user) public view returns (uint256) {
        uint256 _totalMetaGovernance;
        ...
        /** @dev Sum voting power from delegated (allowed) tokenIds to _user. */
        for (uint256 i; i < tokenIdsDelegateds.length; ) {
            uint256 _tokenId = tokenIdsDelegateds[i];
            (uint256 _toPercentage, , uint256 _toIndex) = _lockingPositionDelegate.getMgDelegateeInfoPerTokenAndAddress(
                _tokenId,
                _user
            );
            /** @dev Check if is really delegated, if not mg voting power for this tokenId is 0. */
            if (_toIndex < 999) {
                uint256 _tokenBalance = balanceOfMgCvg(_tokenId);
                _totalMetaGovernance += (_tokenBalance * _toPercentage) / MAX_PERCENTAGE;
            }

            unchecked {
                ++i;
            }
        }
        ...
❌    return _totalMetaGovernance; //@audit total voting power for the tokenID which will be inflated by adding the same tokenID multiple times
    }
```

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionDelegate.sol#L330

### PoC

Add in balance-delegation.spec.ts:

```ts
    it("Manage tokenIds for user10 with dupes", async () => {
        let tokenIds = {owneds: [], mgDelegateds: [1, 1], veDelegateds: []};
        await lockingPositionDelegate.connect(user10).manageOwnedAndDelegated(tokenIds);
    });

    it("Checks mgCVG balances of user10 (delegatee)", async () => {
        const tokenBalance = await lockingPositionService.balanceOfMgCvg(1);

        // USER 10
        const delegatedPercentage = 70n;

        //@audit: The voting power is multiplied by 2 due to the duplicate
        const exploit_multiplier = 2n;
        const expectedVotingPower = (exploit_multiplier * tokenBalance * delegatedPercentage) / 100n;
        const votingPower = await lockingPositionService.mgCvgVotingPowerPerAddress(user10);

        // take solidity rounding down into account
        expect(votingPower).to.be.approximately(expectedVotingPower, 1);
    });

```
## Tool used

## Recommendation

Ensuring the array of token IDs is sorted and contains no duplicates. This can be achieved by verifying that each tokenId in the array is strictly greater than the previous one, it ensures uniqueness without additional data structures.