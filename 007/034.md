Big Cloth Gorilla

high

# The ```pullRewards``` function insufficiently checks ERC20 transfers, leading to potential loss of funds / illegitimate rewards.

## Summary

Success of transfers of rewards is not checked in the [```pullRewards```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/CvgSdtBuffer.sol#L76-L174) function of the [```CvgSdtBuffer```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/CvgSdtBuffer.sol#L76-L174) contract.
That means these transfers can silently fail, leading to ```_cvgSdtStaking``` potentially not being able to pull rewards.

## Vulnerability Detail

The ```transfer``` and ```transferFrom``` functions of ```ERC20``` tokens return a boolean value (```true``` is transfer is successful, ```false``` if it is not), and will not revert in case of unsuccessful transfers.

By definition, 

```solidity
IERC20 _sdt = sdt;
IERC20 _cvgSdt = cvgSdt;
IERC20 _sdFrax3Crv = sdFrax3Crv;
```

meaning these 3 tokens are ```ERC20``` so the above statement applies for them.

This means that these statements will not revert in case of unsuccessful transfers

```solidity
_sdt.transfer(_processor, processorRewards);
```

```solidity
_sdt.transfer(sdtRewardReceiver, sdtAmount);
```

```solidity
_sdFrax3Crv.transferFrom(veSdtMultisig, _processor, processorRewards);
```

```solidity
_sdFrax3Crv.transferFrom(veSdtMultisig, sdtRewardReceiver, sdFrax3CrvAmount);
```

```solidity
 _cvgSdt.transfer(_processor, processorRewards);
```

and 

```solidity
_cvgSdt.transfer(sdtRewardReceiver, cvgSdtAmount);
```

This means that any (or all) of them can silently fail without the function reverting, meaning the user who initiated the function could receive a percentage of rewards without the rewards actually being pulled.

The ```sdtRewardAssets``` return value in that case will also be incorrect.

## Impact

We mark the impact of this vulnerability as HIGH because it could lead to rewards not being received as they should and potential errors in contract accountancy.

It could also lead to rewards being successfully transferred to the ```_processor``` who initiated it without the reward tokens being successfully transferred to the Staking contract.

## Code Snippet

Below is the [```pullRewards```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/CvgSdtBuffer.sol#L76-L174) function definition

```solidity
function pullRewards(address processor) external returns (ICommonStruct.TokenAmount[] memory) {
        ICvgControlTower _cvgControlTower = cvgControlTower;
        ISdtStakingPositionService _cvgSdtStaking = _cvgControlTower.cvgSdtStaking();
        address sdtRewardReceiver = cvgControlTower.sdtRewardReceiver();

        address veSdtMultisig = _cvgControlTower.veSdtMultisig();
        IERC20 _sdt = sdt;
        IERC20 _cvgSdt = cvgSdt;
        IERC20 _sdFrax3Crv = sdFrax3Crv;

        require(msg.sender == address(_cvgSdtStaking), "NOT_CVG_SDT_STAKING");

        /// @dev disperse sdt fees
        _cvgControlTower.sdtFeeCollector().withdrawSdt();

        /// @dev claim sdFrax3CrvReward from feedistributor on behalf of the multisig
        feeDistributor.claim(veSdtMultisig);

        /// @dev Fetches balance of itself in SDT
        uint256 sdtAmount = _sdt.balanceOf(address(this));

        /// @dev Fetches balance of itself in CvgSdt
        uint256 cvgSdtAmount = _cvgSdt.balanceOf(address(this));

        /// @dev Fetches balance of veSdtMultisig in sdFrax3Crv
        uint256 sdFrax3CrvAmount = _sdFrax3Crv.balanceOf(veSdtMultisig);

        /// @dev TokenAmount array struct returned
        ICommonStruct.TokenAmount[] memory sdtRewardAssets = new ICommonStruct.TokenAmount[](3);
        uint256 counter;

        uint256 _processorRewardsPercentage = processorRewardsPercentage;
        address _processor = processor;

        /// @dev distributes if the balance is different from 0
        if (sdtAmount != 0) {
            /// @dev send rewards to claimer
            uint256 processorRewards = sdtAmount * _processorRewardsPercentage / DENOMINATOR;
            if (processorRewards != 0) {
                _sdt.transfer(_processor, processorRewards);
                sdtAmount -= processorRewards;
            }

            sdtRewardAssets[counter++] = ICommonStruct.TokenAmount({token: _sdt, amount: sdtAmount});
            ///@dev transfers all Sdt to the CvgSdtStaking
            _sdt.transfer(sdtRewardReceiver, sdtAmount);
        }
        /// @dev else reduces the length of the array to not return some useless 0 TokenAmount structs
        else {
            // solhint-disable-next-line no-inline-assembly
            assembly {
                mstore(sdtRewardAssets, sub(mload(sdtRewardAssets), 1))
            }
        }

        /// @dev distributes if the balance is different from 0
        if (sdFrax3CrvAmount != 0) {
            /// @dev send rewards to claimer
            uint256 processorRewards = sdFrax3CrvAmount * _processorRewardsPercentage / DENOMINATOR;
            if (processorRewards != 0) {
                _sdFrax3Crv.transferFrom(veSdtMultisig, _processor, processorRewards);
                sdFrax3CrvAmount -= processorRewards;
            }

            sdtRewardAssets[counter++] = ICommonStruct.TokenAmount({token: _sdFrax3Crv, amount: sdFrax3CrvAmount});
            ///@dev transfers from all tokens detained by veSdtMultisig
            _sdFrax3Crv.transferFrom(veSdtMultisig, sdtRewardReceiver, sdFrax3CrvAmount);
        }
        /// @dev else reduces the length of the array to not return some useless 0 TokenAmount structs
        else {
            // solhint-disable-next-line no-inline-assembly
            assembly {
                mstore(sdtRewardAssets, sub(mload(sdtRewardAssets), 1))
            }
        }

        /// @dev distributes if the balance is different from 0
        if (cvgSdtAmount != 0) {
            /// @dev send rewards to claimer
            uint256 processorRewards = cvgSdtAmount * _processorRewardsPercentage / DENOMINATOR;
            if (processorRewards != 0) {
                _cvgSdt.transfer(_processor, processorRewards);
                cvgSdtAmount -= processorRewards;
            }

            sdtRewardAssets[counter++] = ICommonStruct.TokenAmount({token: _cvgSdt, amount: cvgSdtAmount});
            ///@dev transfers all CvgSdt to the CvgSdtStaking
            _cvgSdt.transfer(sdtRewardReceiver, cvgSdtAmount);
        }
        /// @dev else reduces the length of the array to not return some useless 0 TokenAmount structs
        else {
            // solhint-disable-next-line no-inline-assembly
            assembly {
                mstore(sdtRewardAssets, sub(mload(sdtRewardAssets), 1))
            }
        }

        return sdtRewardAssets;
    }
```

## Tool used

Manual Review

## Recommendation

The fix for this vulnerability is relatively straightforward and we just need to ```require``` all these transfers to return ```true``` before moving forward with the rest of the logic.

Fixed version of the [```pullRewards```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/CvgSdtBuffer.sol#L76-L174) function would then look like

```solidity
function pullRewards(address processor) external returns (ICommonStruct.TokenAmount[] memory) {
        ICvgControlTower _cvgControlTower = cvgControlTower;
        ISdtStakingPositionService _cvgSdtStaking = _cvgControlTower.cvgSdtStaking();
        address sdtRewardReceiver = cvgControlTower.sdtRewardReceiver();

        address veSdtMultisig = _cvgControlTower.veSdtMultisig();
        IERC20 _sdt = sdt;
        IERC20 _cvgSdt = cvgSdt;
        IERC20 _sdFrax3Crv = sdFrax3Crv;

        require(msg.sender == address(_cvgSdtStaking), "NOT_CVG_SDT_STAKING");

        /// @dev disperse sdt fees
        _cvgControlTower.sdtFeeCollector().withdrawSdt();

        /// @dev claim sdFrax3CrvReward from feedistributor on behalf of the multisig
        feeDistributor.claim(veSdtMultisig);

        /// @dev Fetches balance of itself in SDT
        uint256 sdtAmount = _sdt.balanceOf(address(this));

        /// @dev Fetches balance of itself in CvgSdt
        uint256 cvgSdtAmount = _cvgSdt.balanceOf(address(this));

        /// @dev Fetches balance of veSdtMultisig in sdFrax3Crv
        uint256 sdFrax3CrvAmount = _sdFrax3Crv.balanceOf(veSdtMultisig);

        /// @dev TokenAmount array struct returned
        ICommonStruct.TokenAmount[] memory sdtRewardAssets = new ICommonStruct.TokenAmount[](3);
        uint256 counter;

        uint256 _processorRewardsPercentage = processorRewardsPercentage;
        address _processor = processor;

        /// @dev distributes if the balance is different from 0
        if (sdtAmount != 0) {
            /// @dev send rewards to claimer
            uint256 processorRewards = sdtAmount * _processorRewardsPercentage / DENOMINATOR;
            if (processorRewards != 0) {
                (bool success, ) = _sdt.transfer(_processor, processorRewards);
                
                require(success, "transfer failed");
                sdtAmount -= processorRewards;
            }

            sdtRewardAssets[counter++] = ICommonStruct.TokenAmount({token: _sdt, amount: sdtAmount});
            ///@dev transfers all Sdt to the CvgSdtStaking
            (bool success, ) = _sdt.transfer(sdtRewardReceiver, sdtAmount);
             require(success, "transfer failed");
        }
        /// @dev else reduces the length of the array to not return some useless 0 TokenAmount structs
        else {
            // solhint-disable-next-line no-inline-assembly
            assembly {
                mstore(sdtRewardAssets, sub(mload(sdtRewardAssets), 1))
            }
        }

        /// @dev distributes if the balance is different from 0
        if (sdFrax3CrvAmount != 0) {
            /// @dev send rewards to claimer
            uint256 processorRewards = sdFrax3CrvAmount * _processorRewardsPercentage / DENOMINATOR;
            if (processorRewards != 0) {
                (bool success, ) = _sdFrax3Crv.transferFrom(veSdtMultisig, _processor, processorRewards);
                require(success, "transfer failed");

                sdFrax3CrvAmount -= processorRewards;
            }

            sdtRewardAssets[counter++] = ICommonStruct.TokenAmount({token: _sdFrax3Crv, amount: sdFrax3CrvAmount});
            ///@dev transfers from all tokens detained by veSdtMultisig
            (bool success, ) = _sdFrax3Crv.transferFrom(veSdtMultisig, sdtRewardReceiver, sdFrax3CrvAmount);
             require(success, "transfer failed");

        }
        /// @dev else reduces the length of the array to not return some useless 0 TokenAmount structs
        else {
            // solhint-disable-next-line no-inline-assembly
            assembly {
                mstore(sdtRewardAssets, sub(mload(sdtRewardAssets), 1))
            }
        }

        /// @dev distributes if the balance is different from 0
        if (cvgSdtAmount != 0) {
            /// @dev send rewards to claimer
            uint256 processorRewards = cvgSdtAmount * _processorRewardsPercentage / DENOMINATOR;
            if (processorRewards != 0) {
                (bool success, ) = _cvgSdt.transfer(_processor, processorRewards);
                require(success, "transfer failed");

                cvgSdtAmount -= processorRewards;
            }

            sdtRewardAssets[counter++] = ICommonStruct.TokenAmount({token: _cvgSdt, amount: cvgSdtAmount});
            ///@dev transfers all CvgSdt to the CvgSdtStaking
            (bool success, ) = _cvgSdt.transfer(sdtRewardReceiver, cvgSdtAmount);
            require(success, "transfer failed");

        }
        /// @dev else reduces the length of the array to not return some useless 0 TokenAmount structs
        else {
            // solhint-disable-next-line no-inline-assembly
            assembly {
                mstore(sdtRewardAssets, sub(mload(sdtRewardAssets), 1))
            }
        }

        return sdtRewardAssets;
}
```
