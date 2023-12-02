Tart Peanut Turtle

medium

# Implicit Assumption of currentCycle in Checkpointing Logic

## Summary
The code implicitly assumes the value of currentCycle as nextCycle - 1 in the checkpointing logic, potentially leading to unexpected behavior and incorrect updating of staked amounts.

## Vulnerability Detail
```solidity
  function withdraw(uint256 tokenId, uint256 amount) external checkCompliance(tokenId) {
        require(amount != 0, "WITHDRAW_LTE_0");

        uint256 _cvgStakingCycle = stakingCycle;

        /// @dev Update the CycleInfo & the TokenInfo for the current & next cycle
        _updateAmountStakedWithdraw(tokenId, amount, _cvgStakingCycle);

        /// @dev Calls in low level the function withdrawing funds from the vault to the user
        _callWithSignature(amount);

        emit Withdraw(tokenId, msg.sender, _cvgStakingCycle, amount);
    }
```
The absence of an explicit assignment for currentCycle before its use in the checkpointing process can result in misinterpretations, particularly if the assumption about its value does not hold true. This oversight may lead to incorrect checkpointing and staked amount updates, impacting the accuracy of the staking mechanism.
```solidity
function _updateAmountStakedWithdraw(uint256 tokenId, uint256 amount, uint256 currentCycle) internal {
        uint256 nextCycle = currentCycle + 1;
        /// @dev get pending staked amount not already eligible for rewards
        uint256 nextCyclePending = _tokenInfoByCycle[nextCycle][tokenId].pendingStaked;
        /// @dev Get amount already staked on the token when the last operation occurred
        uint256 _tokenTotalStaked = tokenTotalStaked(tokenId);

        /// @dev Verify that the withdrawn amount is lower than the total staked amount
        require(amount <= _tokenTotalStaked, "WITHDRAW_EXCEEDS_STAKED_AMOUNT");
        uint256 _newTokenStakedAmount = _tokenTotalStaked - amount;

        /// @dev update last amountStaked for current cycle
        uint256 _lastActionCycle = _stakingHistoryByToken[tokenId][_stakingHistoryByToken[tokenId].length - 1];

        /// @dev if this _lastActionCycle is less than the current cycle =>
        ///      No deposits occurred on the last cycle & no withdraw on this cycle
        if (_lastActionCycle < currentCycle) {
            /// @dev we have so to checkpoint the current cycle
            _stakingHistoryByToken[tokenId].push(currentCycle);
            /// @dev and to report the amountStaked of the lastActionCycle to the currentCycle
            _tokenInfoByCycle[currentCycle][tokenId].amountStaked = _tokenInfoByCycle[_lastActionCycle][tokenId]
                .amountStaked;
        }

        /// @dev updates the amount staked for this position for the next cycle
        _tokenInfoByCycle[nextCycle][tokenId].amountStaked = _newTokenStakedAmount;

        /// @dev Fully removes the amount from the totalStaked of next cycle.
        ///      This withdrawn amount is not anymore eligible to the distribution of the next cycle.
        _cycleInfo[nextCycle].totalStaked -= amount;

        /// @dev If there is some token deposited on this cycle ( pending token )
        ///      We first must to remove them before the tokens that are already accumulating rewards
        if (nextCyclePending != 0) {
            /// @dev If the amount to withdraw is lower or equal to the pending amount
            if (nextCyclePending >= amount) {
                /// @dev we decrement this pending amount
                _tokenInfoByCycle[nextCycle][tokenId].pendingStaked -= amount;
            }
            /// @dev Else, the amount to withdraw is greater than the pending
            else {
                /// @dev Computes the amount to remove from the staked amount eligible to rewards
                uint256 _amount = amount - nextCyclePending;

                /// @dev Fully removes the pending amount for next cycle
                delete _tokenInfoByCycle[nextCycle][tokenId].pendingStaked;

                /// @dev Removes the adjusted amount to the staked total amount eligible to rewards
                _cycleInfo[currentCycle].totalStaked -= _amount;

                /// @dev Removes the adjusted amount to the staked position amount eligible to rewards
                _tokenInfoByCycle[currentCycle][tokenId].amountStaked -= _amount;
            }
        }
        /// @dev If nothing has been desposited on this cycle
        else {
            /// @dev removes the withdrawn amount to the staked total amount eligible to rewards
            _cycleInfo[currentCycle].totalStaked -= amount;
            /// @dev removes the withdrawn amount to the staked token amount eligible to rewards
            _tokenInfoByCycle[currentCycle][tokenId].amountStaked -= amount;
        }
    }
```
The `_lastActionCycle` is less than `currentCycle` which the `_updateAmountStakedWithdraw` assumes is true but the `currentCycle` is not gotten/updated to know if `currentCycle` is greater than the `_lastActionCycle` what it does is that it just pushes the currentCycle to the  _stakingHistoryByToken[tokenId] which shouldn't be.

Example:
Let's say Alice wants to withdraw her staked token from `StakingPositionService` she calls the `withdraw` function and the withdraw function calls the `_updateAmountStakedWithdraw`. it passes some checks if there are then it updates the token amount and the total amount for the next and the current cycle, the question is does it actually get the currentCycle before updating it?

No, it doesn't and the currentCycle checkpoint will not be updated. and  `_tokenInfoByCycle[currentCycle][tokenId].amountStaked = _tokenInfoByCycle[_lastActionCycle][tokenId]` will not be updated and there will no be information about the currentCycle but the  `_tokenInfoByCycle[nextCycle][tokenId].amountStaked = _newTokenStakedAmount;` updates which the function is not intended to be done that way.
## Impact
The vulnerability has the potential to compromise the integrity of staking records, introducing errors in the tracking of staked amounts for users. and this will affect the accuracy of reward distribution and the overall functioning of the staking mechanism. 
## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L752C8-L762C1
## Tool used
Manual Review

## Recommendation
```solidity
     if (_lastActionCycle < currentCycle) {
+     uint256 currentCycle = nextCycle - 1;
            /// @dev we have so to checkpoint the current cycle
            _stakingHistoryByToken[tokenId].push(currentCycle);
            /// @dev and to report the amountStaked of the lastActionCycle to the currentCycle
            _tokenInfoByCycle[currentCycle][tokenId].amountStaked = _tokenInfoByCycle[_lastActionCycle][tokenId]
                .amountStaked;
        }
        _tokenInfoByCycle[nextCycle][tokenId].amountStaked = - _newTokenStakedAmount;
```