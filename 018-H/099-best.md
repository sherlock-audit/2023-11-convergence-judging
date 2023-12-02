Big Cloth Gorilla

high

# Bribe assets are never transferred to the staking in the ```pullRewards``` function of the ```SdtBuffer``` contract.

## Summary

In the [```pullRewards```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBuffer.sol#L74-L171) function of the [```SdtBuffer```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBuffer.sol) contract, gauge rewards are transferred to the staking but not bribe assets although they should.

## Vulnerability Detail

In the [```pullRewards```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBuffer.sol#L74-L171) function of the [```SdtBuffer```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBuffer.sol) contract, we call the [```pullSdStakingBribes```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBlackHole.sol#L96-L134) function of the [```SdtBlackHole```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBlackHole.sol) contract

```solidity
ICommonStruct.TokenAmount[] memory bribeTokens = _sdtBlackHole.pullSdStakingBribes(
            processor,
            _processorRewardsPercentage
 );
```

to receive bribe rewards from the [```SdtBlackHole```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBlackHole.sol) contract. This is done via the 

```solidity
bribeToken.transfer(sdtRewardReceiver, balance);
```

statement while iterating through bribe tokens in the [```pullSdStakingBribes```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBlackHole.sol#L96-L134) function.

We then iterate through gauge rewards and transfer them to the staking, and proceed to the transfer of ```stdFees``` to the ```sdtFeeCollector```.

```solidity
uint256 counter;
address _processor = processor;
for (uint256 j; j < rewardAmount; ) {
            /// @dev Retrieve the reward asset on the gauge contract
            IERC20 token = _gaugeAsset.reward_tokens(j);
            /// @dev Fetches the balance in this reward asset
            uint256 balance = token.balanceOf(address(this));
            /// @dev distributes if the balance is different from 0
            if (balance != 0) {
                uint256 fullBalance = balance;
                /// @dev Some fees are taken in SDT
                if (token == _sdt) {
                    ISdtFeeCollector _feeCollector = cvgControlTower.sdtFeeCollector();
                    /// @dev Fetches the % of fees to take & compute the amount compare to the actual balance
                    uint256 sdtFees = (_feeCollector.rootFees() * balance) / 100_000;
                    balance -= sdtFees;
                    /// @dev Transfers SDT fees to the FeeCollector
                    token.transfer(address(_feeCollector), sdtFees);
                }

                /// @dev send rewards to claimer
                uint256 claimerRewards = (fullBalance * _processorRewardsPercentage) / DENOMINATOR;
                if (claimerRewards > 0) {
                    token.transfer(_processor, claimerRewards);
                    balance -= claimerRewards;
                }

                /// @dev transfers the balance (or the balance - fees for SDT) minus claimer rewards (if any) to the staking contract
                token.transfer(sdtRewardsReceiver, balance);
                /// @dev Pushes in the TokenAmount array
                tokenAmounts[counter++] = ICommonStruct.TokenAmount({token: token, amount: balance});
            }
            /// @dev else reduces the length of the array to not return some useless 0 TokenAmount structs
            else {
                // solhint-disable-next-line no-inline-assembly
                assembly {
                    mstore(tokenAmounts, sub(mload(tokenAmounts), 1))
                }
            }
 ```
 
 
 We then need to iterate through bribe assets and transfer them to the staking as well, as stated in the comments ```        /// @dev Iterates through bribes assets, transfers them to the staking and push in the TokenReward amount```.
 
 This intended behavior is also made clear in the ```technical-docs``` and the diagram of the [```pullRewards```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBuffer.sol#L74-L171) function as per below
 
 ```mermaid
sequenceDiagram
    SdtStaking->>+SdtBuffer: pullRewads()
    note over SdtBuffer: Check Only corresponding SdtStaking
    SdtBuffer->>Gauge: claim_rewards(sdtBlackHole), receives rewards from the gauge
    SdtBuffer->>SdtBlackhole: pullSdStakingBribes(), receives bribes from the SdtBlackhole
    SdtBuffer-->>+Gauge: reward_count(), fetches number of tokens from StakeDao gauge
    SdtBuffer-->>SdtBuffer: Initialize [tokenAmountStruct] with max length

    loop reward_count gauge rewards
        SdtBuffer-->>+Gauge: reward_token(i), fetches the iterated reward token
        SdtBuffer-->>+ERC20: balanceOf(buffer), fetches balanceOf the token
        alt if balance > 0
            alt if reward token is SDT
                SdtBuffer-->>FeeCollector: Get the fee % in SDT to take
                SdtBuffer-->>SdtBuffer: Compute the amount of fees and adjust balance
                SdtBuffer->>ERC20: transfer(feeCollector, feeAmount)
            end
            SdtBuffer->>ERC20: transfer(SdtStaking, balance)
            SdtBuffer-->>SdtBuffer: Push in the [tokenAmountStruct]
        else if balance == 0
            SdtBuffer-->>SdtBuffer: Adjust the array size by removing last element
        end
    end

    loop bribe rewards
        SdtBuffer-->>+ERC20: balanceOf(buffer), fetches balanceOf the token
        alt if balance > 0
            SdtBuffer->>ERC20: transfer(SdtStaking, balance)
            SdtBuffer-->>SdtBuffer: Push in the [tokenAmountStruct]
        else if balance == 0
            SdtBuffer-->>SdtBuffer: Adjust the array size by removing last element
        end
    end

    SdtBuffer -->> SdtStaking : returns([tokenAmountStruct])
```
 
 However, we only push the TokenReward amount WITHOUT transferring the assets to the staking contract, meaning these assets are effectively stuck in the [```SdtBuffer```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBuffer.sol) contract.

## Impact

We mark this vulnerability as HIGH as this means bribe assets will be stuck in the [```SdtBuffer```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBuffer.sol) contract hence not claimable by the user, which defeats the purpose of having a bribe mechanism for this contract.

## Code Snippet

Below is the code of the [```pullRewards```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBuffer.sol#L74-L171) function.

```solidity
function pullRewards(address processor) external returns (ICommonStruct.TokenAmount[] memory) {
        /// @dev Prepare data from storage
        ISdtBlackHole _sdtBlackHole = cvgControlTower.sdtBlackHole();
        address sdtRewardsReceiver = cvgControlTower.sdtRewardReceiver();

        ISdAssetGauge _gaugeAsset = gaugeAsset;
        IERC20 _sdt = sdt;
        uint256 _processorRewardsPercentage = processorRewardsPercentage;

        /// @dev callable only by the linked staking contract
        require(msg.sender == sdtStaking, "ONLY_STAKING");

        /// @dev claim & receives rewards from the gauge
        _gaugeAsset.claim_rewards(address(_sdtBlackHole));

        /// @dev receives bribes rewards from the SdtBlackHole and fetches the array of all bribe assets linked to this buffer in the SdtBlackHole
        ICommonStruct.TokenAmount[] memory bribeTokens = _sdtBlackHole.pullSdStakingBribes(
            processor,
            _processorRewardsPercentage
        );

        /// @dev Fetches the amount of rewards contained in the StakeDao gauge
        uint256 rewardAmount = _gaugeAsset.reward_count();

        /// @dev Instantiate the returned array with the maximum potential length
        ICommonStruct.TokenAmount[] memory tokenAmounts = new ICommonStruct.TokenAmount[](
            rewardAmount + bribeTokens.length
        );

        /// @dev counter used for the actual index of the array, we need it as we remove 0 amounts from our returned array
        uint256 counter;
        address _processor = processor;
        for (uint256 j; j < rewardAmount; ) {
            /// @dev Retrieve the reward asset on the gauge contract
            IERC20 token = _gaugeAsset.reward_tokens(j);
            /// @dev Fetches the balance in this reward asset
            uint256 balance = token.balanceOf(address(this));
            /// @dev distributes if the balance is different from 0
            if (balance != 0) {
                uint256 fullBalance = balance;
                /// @dev Some fees are taken in SDT
                if (token == _sdt) {
                    ISdtFeeCollector _feeCollector = cvgControlTower.sdtFeeCollector();
                    /// @dev Fetches the % of fees to take & compute the amount compare to the actual balance
                    uint256 sdtFees = (_feeCollector.rootFees() * balance) / 100_000;
                    balance -= sdtFees;
                    /// @dev Transfers SDT fees to the FeeCollector
                    token.transfer(address(_feeCollector), sdtFees);
                }

                /// @dev send rewards to claimer
                uint256 claimerRewards = (fullBalance * _processorRewardsPercentage) / DENOMINATOR;
                if (claimerRewards > 0) {
                    token.transfer(_processor, claimerRewards);
                    balance -= claimerRewards;
                }

                /// @dev transfers the balance (or the balance - fees for SDT) minus claimer rewards (if any) to the staking contract
                token.transfer(sdtRewardsReceiver, balance);
                /// @dev Pushes in the TokenAmount array
                tokenAmounts[counter++] = ICommonStruct.TokenAmount({token: token, amount: balance});
            }
            /// @dev else reduces the length of the array to not return some useless 0 TokenAmount structs
            else {
                // solhint-disable-next-line no-inline-assembly
                assembly {
                    mstore(tokenAmounts, sub(mload(tokenAmounts), 1))
                }
            }

            unchecked {
                ++j;
            }
        }

        /// @dev Iterates through bribes assets, transfers them to the staking and push in the TokenReward amount
        for (uint256 j; j < bribeTokens.length; ) {
            IERC20 token = bribeTokens[j].token;
            uint256 amount = bribeTokens[j].amount;
            /// @dev Fetches the bribe token balance
            if (amount != 0) {
                /// @dev Pushes in the TokenAmount array
                tokenAmounts[counter++] = ICommonStruct.TokenAmount({token: token, amount: amount});
            }
            /// @dev else reduces the length of the array to not return some useless 0 TokenAmount structs
            else {
                // solhint-disable-next-line no-inline-assembly
                assembly {
                    mstore(tokenAmounts, sub(mload(tokenAmounts), 1))
                }
            }
            unchecked {
                ++j;
            }
        }

        return tokenAmounts;
 }
```

and the [```pullSdStakingBribes```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBlackHole.sol#L96-L134) function of the [```SdtBlackHole```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBlackHole.sol) contract

```solidity
 function pullSdStakingBribes(
        address _processor,
        uint256 _processorRewardsPercentage
    ) external returns (ICommonStruct.TokenAmount[] memory) {
        /// @dev verifies that the caller is a Buffer
        require(isBuffer[msg.sender], "NOT_A_BUFFER");

        /// @dev fetches bribes token of the calling buffer
        IERC20[] memory bribeTokens = bribeTokensLinkedToBuffer[msg.sender];
        ICommonStruct.TokenAmount[] memory _bribeTokensAmounts = new ICommonStruct.TokenAmount[](bribeTokens.length);

        address sdtRewardReceiver = cvgControlTower.sdtRewardReceiver();

        /// @dev iterates over all bribes token retrieved
        for (uint256 i; i < bribeTokens.length; ) {
            /// @dev get the balance in the bribe token
            IERC20 bribeToken = bribeTokens[i];
            uint256 balance = bribeToken.balanceOf(address(this));

            if (balance != 0) {
                /// @dev send rewards to claimer
                uint256 claimerRewards = (balance * _processorRewardsPercentage) / 100_000;
                if (claimerRewards > 0) {
                    bribeToken.transfer(_processor, claimerRewards);
                    balance -= claimerRewards;
                }

                /// @dev send the balance of the bribe token minus claimer rewards to the buffer
                bribeToken.transfer(sdtRewardReceiver, balance);

                _bribeTokensAmounts[i] = ICommonStruct.TokenAmount({token: bribeTokens[i], amount: balance});
            }
            unchecked {
                ++i;
            }
        }

        return _bribeTokensAmounts;
 }
 ```

## Tool used

Manual Review / Visual Studio

## Recommendation

The fix is relatively straightforward and we just need to transfer the bribe assets to the staking when iterating over them in the [```pullRewards```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBuffer.sol#L74-L171) function, which would then look like the below (note that we added the statement ```token.transfer(sdtRewardsReceiver, amount);```)

```solidity
function pullRewards(address processor) external returns (ICommonStruct.TokenAmount[] memory) {
        /// @dev Prepare data from storage
        ISdtBlackHole _sdtBlackHole = cvgControlTower.sdtBlackHole();
        address sdtRewardsReceiver = cvgControlTower.sdtRewardReceiver();

        ISdAssetGauge _gaugeAsset = gaugeAsset;
        IERC20 _sdt = sdt;
        uint256 _processorRewardsPercentage = processorRewardsPercentage;

        /// @dev callable only by the linked staking contract
        require(msg.sender == sdtStaking, "ONLY_STAKING");

        /// @dev claim & receives rewards from the gauge
        _gaugeAsset.claim_rewards(address(_sdtBlackHole));

        /// @dev receives bribes rewards from the SdtBlackHole and fetches the array of all bribe assets linked to this buffer in the SdtBlackHole
        ICommonStruct.TokenAmount[] memory bribeTokens = _sdtBlackHole.pullSdStakingBribes(
            processor,
            _processorRewardsPercentage
        );

        /// @dev Fetches the amount of rewards contained in the StakeDao gauge
        uint256 rewardAmount = _gaugeAsset.reward_count();

        /// @dev Instantiate the returned array with the maximum potential length
        ICommonStruct.TokenAmount[] memory tokenAmounts = new ICommonStruct.TokenAmount[](
            rewardAmount + bribeTokens.length
        );

        /// @dev counter used for the actual index of the array, we need it as we remove 0 amounts from our returned array
        uint256 counter;
        address _processor = processor;
        for (uint256 j; j < rewardAmount; ) {
            /// @dev Retrieve the reward asset on the gauge contract
            IERC20 token = _gaugeAsset.reward_tokens(j);
            /// @dev Fetches the balance in this reward asset
            uint256 balance = token.balanceOf(address(this));
            /// @dev distributes if the balance is different from 0
            if (balance != 0) {
                uint256 fullBalance = balance;
                /// @dev Some fees are taken in SDT
                if (token == _sdt) {
                    ISdtFeeCollector _feeCollector = cvgControlTower.sdtFeeCollector();
                    /// @dev Fetches the % of fees to take & compute the amount compare to the actual balance
                    uint256 sdtFees = (_feeCollector.rootFees() * balance) / 100_000;
                    balance -= sdtFees;
                    /// @dev Transfers SDT fees to the FeeCollector
                    token.transfer(address(_feeCollector), sdtFees);
                }

                /// @dev send rewards to claimer
                uint256 claimerRewards = (fullBalance * _processorRewardsPercentage) / DENOMINATOR;
                if (claimerRewards > 0) {
                    token.transfer(_processor, claimerRewards);
                    balance -= claimerRewards;
                }

                /// @dev transfers the balance (or the balance - fees for SDT) minus claimer rewards (if any) to the staking contract
                token.transfer(sdtRewardsReceiver, balance);
                /// @dev Pushes in the TokenAmount array
                tokenAmounts[counter++] = ICommonStruct.TokenAmount({token: token, amount: balance});
            }
            /// @dev else reduces the length of the array to not return some useless 0 TokenAmount structs
            else {
                // solhint-disable-next-line no-inline-assembly
                assembly {
                    mstore(tokenAmounts, sub(mload(tokenAmounts), 1))
                }
            }

            unchecked {
                ++j;
            }
        }

        /// @dev Iterates through bribes assets, transfers them to the staking and push in the TokenReward amount
        for (uint256 j; j < bribeTokens.length; ) {
            IERC20 token = bribeTokens[j].token;
            uint256 amount = bribeTokens[j].amount;
            /// @dev Fetches the bribe token balance
            if (amount != 0) {
                 token.transfer(sdtRewardsReceiver, amount);
                            
                /// @dev Pushes in the TokenAmount array
                tokenAmounts[counter++] = ICommonStruct.TokenAmount({token: token, amount: amount});
            }
            /// @dev else reduces the length of the array to not return some useless 0 TokenAmount structs
            else {
                // solhint-disable-next-line no-inline-assembly
                assembly {
                    mstore(tokenAmounts, sub(mload(tokenAmounts), 1))
                }
            }
            unchecked {
                ++j;
            }
        }

        return tokenAmounts;
 }
```
