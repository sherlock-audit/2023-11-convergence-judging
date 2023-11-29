Tart Peanut Turtle

high

# Inaccurate Cycle Checkpointing and Staked Amount Tracking in Convergence Protocol

## Summary
The identified vulnerability in the Convergence Protocol pertains to the deposit function. Specifically, the _updateStakedDeposit function lacks adequate checks, potentially leading to inaccurate update cycle checkpoint validation. This flaw may result in incorrect reward distribution and expose the protocol to manipulations by users.
  
## Vulnerability Detail
The _updateAmountStakedDeposit function does not sufficiently verify whether a user is depositing within the same cycle, leading to potential issues with cycle checkpoints.
Example (Alice): If Alice updates her staked deposit in the same cycle, the system incorrectly pushes a new checkpoint for the next cycle, causing a miscalculation of staked amounts.
```solidity
    function deposit(uint256 tokenId, uint256 amount, address operator) external {
        /// @dev Verify if deposits are paused
        require(!depositPaused, "DEPOSIT_PAUSED");
        /// @dev Verify if the staked amount is > 0
        require(amount != 0, "DEPOSIT_LTE_0");
        /// @dev Memorize storage data
        ICvgControlTower _cvgControlTower = cvgControlTower;
        ISdtStakingPositionManager _sdtStakingPositionManager = sdtStakingPositionManager;
        uint256 _cvgStakingCycle = stakingCycle;
        /// @dev Fetches the receiver, if the caller is the SdtUtilities, we put the initiator of the tx as receiver
        address receiver = msg.sender == _cvgControlTower.sdtUtilities() ? operator : msg.sender;
        uint256 _tokenId;
        /// @dev If tokenId != 0, user deposits for an already existing position, we have so to check ownership
        if (tokenId != 0) {
            /// @dev Fetches, for the tokenId, the owner, the StakingPositionService linked to and the timestamp of unlocking
            _sdtStakingPositionManager.checkIncreaseDepositCompliance(tokenId, receiver);
            _tokenId = tokenId;
        }
        /// @dev Else, we increment the nextId to get the new tokenId
        else {
            _tokenId = _sdtStakingPositionManager.nextId();
        }

        /// @dev Update the CycleInfo & the TokenInfo for the next cycle
        _updateAmountStakedDeposit(_tokenId, amount, _cvgStakingCycle + 1);

        /// @dev transfers stakingAsset tokens from caller to the vault contract ( SdtBlackhole or CvgSdtStaking )
        stakingAsset.transferFrom(msg.sender, vault, amount);
        /// @dev Mints the NFT to the receiver only when tokenId == 0
        if (tokenId == 0) {
            _sdtStakingPositionManager.mint(receiver);
        }

        emit Deposit(_tokenId, receiver, _cvgStakingCycle, amount);
    }
```
The code keeps track of cycles where actions occurred and checks them for reference. The `_stakingHistoryByToken` array is used to record these checkpoints, and the amount staked for each cycle is reported based on the last action cycle. This helps manage the distribution of rewards and pending amounts accurately across cycles, but the code fails whenever a user updates the staked deposit after the _lastActionCycle is initially 0.
```solidity
 function _updateAmountStakedDeposit(uint256 tokenId, uint256 amount, uint256 nextCycle) internal {
        /// @dev Get the amount already staked on this position and adds it the new deposited amount
        uint256 _newTokenStakedAmount = tokenTotalStaked(tokenId) + amount;

        /// @dev updates the amount staked for this tokenId for the next cvgCycle
        _tokenInfoByCycle[nextCycle][tokenId].amountStaked = _newTokenStakedAmount;

        /**
         * @dev Increments the pending amount with the deposited amount.
         *      The pending amount is the staked amount still in accumulation mode.
         *      Is always removed from the witdhraw before the amountStaked.
         */
        _tokenInfoByCycle[nextCycle][tokenId].pendingStaked += amount;

        /// @dev increments the total amount staked on the Staking Contract for the nextCycle
        _cycleInfo[nextCycle].totalStaked += amount;

        uint256 cycleLength = _stakingHistoryByToken[tokenId].length;

        /// @dev If it's the mint of the position
        if (cycleLength == 0) {
            _stakingHistoryByToken[tokenId].push(nextCycle);
        }
        /// @dev Else it's not the mint of the position
        else {
            /// @dev fetches the _lastActionCycle where an action has been performed
            uint256 _lastActionCycle = _stakingHistoryByToken[tokenId][cycleLength - 1];

            /// @dev if this _lastActionCycle is less than the next cycle => it's the first deposit done on this cycle
            if (_lastActionCycle < nextCycle) {
                uint256 currentCycle = nextCycle - 1;
                /// @dev if this _lastActionCycle is less than the current cycle =>
                ///      No deposits occurred on the last cycle & no withdraw on this cycle
                if (_lastActionCycle < currentCycle) {
                    /// @dev we have so to checkpoint the current cycle
                    _stakingHistoryByToken[tokenId].push(currentCycle);
                    /// @dev and to report the amountStaked of the lastActionCycle to the currentCycle
                    _tokenInfoByCycle[currentCycle][tokenId].amountStaked = _tokenInfoByCycle[_lastActionCycle][tokenId]
                        .amountStaked;
                }
                /// @dev checkpoint the next cycle
                _stakingHistoryByToken[tokenId].push(nextCycle);
            }
        }
    }
 ``` 
 
 ## POC
 ```solidity
 // Assume tokenId = 1

// Alice stakes 100 stakingAsset in Cycle N
// _stakingHistoryByToken[1] is initially empty
// _lastActionCycle is initially 0

uint256 _lastActionCycle = _stakingHistoryByToken[1][_stakingHistoryByToken[1].length - 1];

// Assuming nextCycle is N+1 (we don't know the exact value in this example)

if (_lastActionCycle < nextCycle) {
    uint256 currentCycle = nextCycle - 1;

    // This condition is true since _lastActionCycle is initially 0
    if (_lastActionCycle < currentCycle) {
        // Record the current cycle as a checkpoint (currentCycle = N)
        _stakingHistoryByToken[1].push(currentCycle);

        // Report the amountStaked of the lastActionCycle (0) to the currentCycle (N)
        _tokenInfoByCycle[N][1].amountStaked = _tokenInfoByCycle[0][1].amountStaked;
    }

    // Checkpoint the next cycle (nextCycle = N+1)
    _stakingHistoryByToken[1].push(nextCycle);
}

// Now, let's say someone comes and updates their staked deposit in the same cycle (N+1)

// Alice updates her staked deposit to 150 stakingAsset in Cycle N+1
// _lastActionCycle is now N+1 (last checkpoint)

// The same logic will be applied again:

_lastActionCycle = _stakingHistoryByToken[1][_stakingHistoryByToken[1].length - 1];

if (_lastActionCycle < nextCycle) {
    uint256 currentCycle = nextCycle - 1;

    // This condition is now false since _lastActionCycle is N+1 (last checkpoint)
    // So, it doesn't enter this block, and nextCycle is not pushed again
}

// The updated staked deposit will be reflected in the currentCycle (N+1)
_tokenInfoByCycle[N+1][1].amountStaked = updatedStakedAmount; // Updated staked amount
```
Since the _lastActionCycle is N+1 and it checks `if (_lastActionCycle < currentCycle)` which is false, the  currentCycle which is still `N+1` but is not pushed into the  `_stakingHistoryByToken[1]` array but it updates the  `_tokenInfoByCycle[N+1][1].amountStaked = _tokenInfoByCycle[N+1][1].amountStaked;` which leads to wrongly `_updateAmountStakedDeposit()` deposit.

This is likely a High risk. Exploitation of this could result in financial losses for users, as rewards might be miscalculated or distributed incorrectly.

## Impact
Malicious actors could manipulate the deposit process, impacting the integrity of the Convergence Protocol

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L706C7-L723C65

## Tool used
Manual Review

## Recommendation
Implement more robust checks for cycle checkpoints, ensuring accurate tracking of staked amounts during deposits.

