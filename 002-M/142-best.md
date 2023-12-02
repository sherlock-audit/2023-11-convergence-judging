Atomic Velvet Nuthatch

medium

# Delegation Limitation in Voting Power Management

## Summary

MgCVG Voting power delegation system is constrained by 2 hard limits, first on the number of tokens delegated to one user (`maxTokenIdsDelegated = 25`) and second on the number of delegatees for one token ( `maxMgDelegatees = 5`). Once this limit is reached for a token, the token owner cannot modify the delegation percentage to an existing delegated user. This inflexibility can prevent efficient and dynamic management of delegated voting power.

## Vulnerability Detail
Observe these lines : 
```solidity
function delegateMgCvg(uint256 _tokenId, address _to, uint96 _percentage) external onlyTokenOwner(_tokenId) {
    require(_percentage <= 100, "INVALID_PERCENTAGE");

    uint256 _delegateesLength = delegatedMgCvg[_tokenId].length;
    require(_delegateesLength < maxMgDelegatees, "TOO_MUCH_DELEGATEES");

    uint256 tokenIdsDelegated = mgCvgDelegatees[_to].length;
    require(tokenIdsDelegated < maxTokenIdsDelegated, "TOO_MUCH_MG_TOKEN_ID_DELEGATED");
```
if either `maxMgDelegatees` or `maxTokenIdsDelegated` are reached,  delegation is no longer possible.
The problem is the fact that this function can be either used to delegate or to update percentage of delegation or also to remove a delegation but in cases where we already delegated to a maximum of users (`maxMgDelegatees`) OR the user to who we delegated has reached the maximum number of tokens that can be delegated to him/her (`maxTokenIdsDelegated`), an update or a removal of delegation is no longer possible.

6 scenarios are possible : 
1. `maxTokenIdsDelegated` is set to 5, Alice is the third to delegate her voting power to Bob and choose to delegate 10% to him. Bob gets 2 other people delegating their tokens to him, Alice wants to increase the power delegated to Bob to 50% but she cannot due to Bob reaching `maxTokenIdsDelegated`
2. `maxTokenIdsDelegated` is set to 25, Alice is the 10th to delegate her voting power to Bob and choose to delegate 10%, DAO decrease `maxTokenIdsDelegated` to 3, Alice wants to increase the power delegated to Bob to 50%, but she cannot due to this
3. `maxTokenIdsDelegated` is set to 5, Alice is the third to delegate her voting power to Bob and choose to delegate 90%. Bob gets 2 other people delegating their tokens to him, Alice wants to only remove the power delegated to Bob using this function, but she cannot due to this
4. `maxMgDelegatees` is set to 3, Alice delegates her voting power to Bob,Charly and Donald by 20% each, Alice reaches `maxMgDelegatees` and she cannot update her voting power for any of Bob,Charly or Donald
5. `maxMgDelegatees` is set to 5, Alice delegates her voting power to Bob,Charly and Donald by 20% each,DAO decreases`maxMgDelegatees` to 3. Alice cannot update or remove her voting power delegated to any of Bob,Charly and Donald
6. `maxMgDelegatees` is set to 3, Alice delegates her voting power to Bob,Charly and Donald by 20% each, Alice wants to only remove her delegation to Bob but she reached `maxMgDelegatees` so she cannot only remove her delegation to Bob

A function is provided to remove all user to who we delegated but this function cannot be used as a solution to this problem due to 2 things : 
- It's clearly not intended to do an update of voting power percentage by first removing all delegation we did because `delegateMgCvg()` is clearly defined to allow to delegate OR to remove one delegation OR to update percentage of delegation but in some cases it's impossible which is not acceptable
- if Alice wants to update it's percentage delegated to Bob , she would have to remove all her delegatees and would take the risk that someone is faster than her and delegate to Bob before her, making Bob reaches `maxTokenIdsDelegated` and would render impossible for Alice to re-delegate to Bob

### POC
You can add it to test/ut/delegation/balance-delegation.spec.ts : 
```Typescript
it("maxTokenIdsDelegated is reached => Cannot update percentage of delegate", async function () {
        (await lockingPositionDelegate.maxTokenIdsDelegated()).should.be.equal(25);
        await lockingPositionDelegate.connect(treasuryDao).setMaxTokenIdsDelegated(3);
        (await lockingPositionDelegate.maxTokenIdsDelegated()).should.be.equal(3);

        await lockingPositionDelegate.connect(user1).delegateMgCvg(1, user10, 20);
        await lockingPositionDelegate.connect(user2).delegateMgCvg(2, user10, 30);
        await lockingPositionDelegate.connect(user3).delegateMgCvg(3, user10, 30);
        
        const txFail = lockingPositionDelegate.connect(user1).delegateMgCvg(1, user10, 40);
        await expect(txFail).to.be.revertedWith("TOO_MUCH_MG_TOKEN_ID_DELEGATED");
    });
    it("maxTokenIdsDelegated IS DECREASED => PERCENTAGE UPDATE IS NO LONGER POSSIBLE", async function () {
        await lockingPositionDelegate.connect(treasuryDao).setMaxTokenIdsDelegated(25);
        (await lockingPositionDelegate.maxTokenIdsDelegated()).should.be.equal(25);

        await lockingPositionDelegate.connect(user1).delegateMgCvg(1, user10, 20);
        await lockingPositionDelegate.connect(user2).delegateMgCvg(2, user10, 30);
        await lockingPositionDelegate.connect(user3).delegateMgCvg(3, user10, 30);

        await lockingPositionDelegate.connect(treasuryDao).setMaxTokenIdsDelegated(3);
        (await lockingPositionDelegate.maxTokenIdsDelegated()).should.be.equal(3);        

        const txFail = lockingPositionDelegate.connect(user1).delegateMgCvg(1, user10, 40);
        await expect(txFail).to.be.revertedWith("TOO_MUCH_MG_TOKEN_ID_DELEGATED");
        await lockingPositionDelegate.connect(treasuryDao).setMaxTokenIdsDelegated(25);
        (await lockingPositionDelegate.maxTokenIdsDelegated()).should.be.equal(25);
    });
    it("maxMgDelegatees : TRY TO UPDATE PERCENTAGE DELEGATED TO A USER IF WE ALREADY REACH maxMgDelegatees", async function () {
        await lockingPositionDelegate.connect(treasuryDao).setMaxMgDelegatees(3);
        (await lockingPositionDelegate.maxMgDelegatees()).should.be.equal(3);

        await lockingPositionDelegate.connect(user1).delegateMgCvg(1, user10, 20);
        await lockingPositionDelegate.connect(user1).delegateMgCvg(1, user2, 30);
        await lockingPositionDelegate.connect(user1).delegateMgCvg(1, user3, 30);

        const txFail = lockingPositionDelegate.connect(user1).delegateMgCvg(1, user10, 40);
        await expect(txFail).to.be.revertedWith("TOO_MUCH_DELEGATEES");
    });
    it("maxMgDelegatees : maxMgDelegatees IS DECREASED => PERCENTAGE UPDATE IS NO LONGER POSSIBLE", async function () {
        await lockingPositionDelegate.connect(treasuryDao).setMaxMgDelegatees(5);
        (await lockingPositionDelegate.maxMgDelegatees()).should.be.equal(5);

        await lockingPositionDelegate.connect(user1).delegateMgCvg(1, user10, 20);
        await lockingPositionDelegate.connect(user1).delegateMgCvg(1, user2, 30);
        await lockingPositionDelegate.connect(user1).delegateMgCvg(1, user3, 10);

        await lockingPositionDelegate.connect(treasuryDao).setMaxMgDelegatees(2);
        (await lockingPositionDelegate.maxMgDelegatees()).should.be.equal(2);

        const txFail2 = lockingPositionDelegate.connect(user1).delegateMgCvg(1, user2, 50);
        await expect(txFail2).to.be.revertedWith("TOO_MUCH_DELEGATEES");
    });
```
## Impact
In some cases it is impossible to update percentage delegated or to remove only one delegated percentage then forcing users to remove all their voting power delegatations, taking the risk that someone is faster then them to delegate to their old delegated users and reach threshold for delegation, making impossible for them to re-delegate

## Code Snippet

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionDelegate.sol#L278

## Tool used

Manual Review

## Recommendation

Separate functions for new delegations and updates : Implement logic that differentiates between adding a new delegatee and updating an existing delegation to allow updates to existing delegations even if the maximum number of delegatees is reached