Precise Plum Shark

medium

# Missing events in setter functions can reduce transparency and auditability in `LockingPositionDelegate` contract

## Summary
The code snippet is a contract that manages the delegation of locking positions for voting power. The contract has two setter functions that update the state variables `maxMgDelegatees` and `maxTokenIdsDelegated`. However, these functions do not emit any events when the state variables are changed. This can reduce the transparency and auditability of the contract.

## Vulnerability Detail
The contract `LockingPositionDelegate` has two setter functions: `setMaxMgDelegatees()` and `setMaxTokenIdsDelegated()`. These functions are only callable by the owner of the `lockingPositionManager` contract. The functions update the state variables `maxMgDelegatees` and `maxTokenIdsDelegated`, which are used to limit the number of `delegatees` and token IDs that can be delegated to an address. However, these functions do not emit any events when the state variables are changed. This can reduce the transparency and auditability of the contract, as it makes it harder to track and verify the changes in the contract state.

## Impact
Missing events in setter functions can have negative consequences for the functionality and security of the contract. For example, if the owner changes the state variables without emitting events, then the users and the other contracts may not be aware of the changes and may rely on outdated or incorrect information. Moreover, missing events can also make it harder to debug and test the contract, as it reduces the visibility and traceability of the contract behavior.

## Code Snippet
Source Link:- https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionDelegate.sol#L86-L99

## Tool used
- Manual Review

## Recommendation
To improve the transparency and auditability of the contract, emit events in setter functions when the state variables are changed. The events should include the old and the new values of the state variables, as well as the address of the caller.
```diff
// Define events for state variable changes
+         event MaxMgDelegateesChanged(uint256 oldMaxMgDelegatees, uint256 newMaxMgDelegatees, 
            address indexed caller);
+        event MaxTokenIdsDelegatedChanged(uint256 oldMaxTokenIdsDelegated, uint256 
           newMaxTokenIdsDelegated, address indexed caller);

function setMaxMgDelegatees(uint256 _maxMgDelegatees) external {
        require(cvgControlTower.lockingPositionManager().owner() == msg.sender, "NOT_OWNER");
        // Emit event when maxMgDelegatees is changed
+        emit MaxMgDelegateesChanged(maxMgDelegatees, _maxMgDelegatees, msg.sender);
        maxMgDelegatees = _maxMgDelegatees;
    }

    /**
    * @notice Set the maximum number of tokenIds that can be delegated to an address.
    * @dev this limit is set to avoid oog when calculating or managing the voting power, and spamming user.
    * @param _maxTokenIdsDelegated is the maximum number of tokenIds delegated for an address.
    */
    function setMaxTokenIdsDelegated(uint256 _maxTokenIdsDelegated) external {
        require(cvgControlTower.lockingPositionManager().owner() == msg.sender, "NOT_OWNER");
        // Emit event when maxTokenIdsDelegated is changed
+        emit MaxTokenIdsDelegatedChanged(maxTokenIdsDelegated, _maxTokenIdsDelegated, msg.sender);
        maxTokenIdsDelegated = _maxTokenIdsDelegated;
    }

```