Atomic Velvet Nuthatch

high

# USDC Blacklisting Risk Affects several Reward Distribution process

## Summary
Convergence protocol's reward distribution mechanism is vulnerable to disruption due to the USDC blacklisting feature. If a user or a critical protocol address like the `feeCollector` or `sdtRewardsReceiver` is blacklisted by USDC, it can hinder the reward claiming process for users and potentially halt the distribution of all rewards, creating significant functionality issues

## Vulnerability Detail
According to ReadMe , several contracts should be able to receive and store USDC for rewards : 
- SdtBuffer : Accumulates and receives any ERC20 coming from the StakeDao Gauge ( SDT, CRV, 3CRV, BAL, **USDC**, FXS, FXN, PENDLE, ANGLE sdCRV, sdBAL … )
- YsDistributor  : Receives rewards from the treasury. The list of ERC20 can vary. Curve (CRV),Convex (CVX),StakeDao (SDT),Frax-Share (FXS),Prisma (PRISMA),(…),USDC,USDT,DAI

Here are the risks of a USDC Blacklist : 
**1. Claim Rewards Issue in `YSDistributor.sol`**
In the claimRewards function, rewards are distributed to users based on their share of ysCvg tokens. If a user is blacklisted by USDC, the transfer of USDC rewards will fail, effectively blocking the user from claiming any rewards.
```solidity
function _claimTokenRewards(uint256 tokenId, uint256 tdeId, uint256 share, address receiver) internal {
    ...
    for (uint256 i; i < tokens.length; ) {
        IERC20 _token = IERC20(tokens[i]);
        uint256 _amountUser = _calculateUserRewardAmount(tdeId, _token, share);
        _token.safeTransfer(receiver, _amountUser); // Fails if receiver is USDC blacklisted
        ...
    }
    ...
}
```
**2. Reward Pulling Issue in `SdtStakingPositionService.sol` and `SdtBuffer.sol` :**
The pullRewards function handles the distribution of various rewards, including USDC. If the `feeCollector` is blacklisted, transfers of USDC will fail, disrupting the rewards claiming process for all users.

```solidity
function pullRewards(address processor) external returns (ICommonStruct.TokenAmount[] memory) {
    ...
    if (token == _sdt) {
        ISdtFeeCollector _feeCollector = cvgControlTower.sdtFeeCollector();
        uint256 sdtFees = (_feeCollector.rootFees() * balance) / 100_000;
        token.transfer(address(_feeCollector), sdtFees); // Fails if feeCollector is USDC blacklisted
    }
    ...
    token.transfer(sdtRewardsReceiver, balance); // Fails if sdtRewardsReceiver is USDC blacklisted
    ...
}
```

## Impact
- Blocked Reward Claims: Users blacklisted by USDC are unable to claim their accrued rewards(Curve (CRV),Convex (CVX),StakeDao (SDT),Frax-Share (FXS),Prisma (PRISMA),(…),USDT,DAI), even those that are not in USDC, losing access to their rightful assets.
- System-wide Disruption: If critical protocol address like `feeCollector` is blacklisted, the reward distribution process for all users can be halted, regardless of their individual status.

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/YsDistributor.sol#L213
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBuffer.sol#L121
## Tool used

Manual Review

## Recommendation

Refactor the system to not be USDC dependant in case of blacklist