Recumbent Shamrock Marmot

high

# Attacker can potentially own upto 99% of staking vault share and claim all CVG token rewards using flashloan at every cycle

## Summary
At the end of every cycle(right before the `CvgRewards::writeStakingRewards()` call) one can deposit in `SdtStakingPositionService::deposit()` and immediately after the `writeStakingRewards()` call he/she can claim CVG rewards and withdraw the position from `SdtStakingPositionService::withdraw()`. This opened up an opportunity to flash loan and deposit the asset before the `writeStakingRewards()` which allows the attacker to potentially own up to 99% of the vault based on the loan and already staked amount. And, make a claim of CVG rewards then withdraw the asset and repay the loan.

## Vulnerability Detail
One can claim CVG rewards `SdtStakingPositionService::claimCvgRewards()` only in the 
 `nextCycle` https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L687  after deposit. So, anyone can deposit right before the staking cycle change and claim a CVG reward immediately after a cycle change which is triggered by `CvgRewards::writeStakingRewards()`. Additionally, this is an external function and anyone can call it.  So, an attacker can take advantage of it and create an attacking contract that will execute the following steps to own the majority of shares of a vault and claim CVG.

1. The attacker requests and gets a flash loan of an asset to deposit.
2. Attacker deposits the asset from `SdtStakingPositionService::deposit()`
    - Now, an attacker owns the majority of shares of a vault depending on the loan amount.
3. Initiate `CvgRewards::writeStakingRewards()`, which increases the staking cycle count.
4. Claim CVG rewards from `SdtStakingPositionService::claimCvgRewards()`
5. Withdraw the asset that was deposited.
6. Repay the flash loan.

## Impact
An attacker can claim more CVG Rewards at every cycle practically without staking. It results in a huge percentage of CVG minted to an attacker and in grievances of innocent staker’s reward share.

## Code Snippet
```js
    it('should deposit, claim and withdraw', async() => {

        //Go to cycle 2
        await increaseCvgCycle(contractsUsers, 1);

        //The end of cycle 2 and beginning of cycle 3
        await time.increase(7 * 86400);

        //1. Get a flash loan

        //2. Deposit the asset
        await sdCRVStaking.connect(user1).deposit(0, ethers.parseEther("10"), ethers.ZeroAddress);

        //3. Call writeStakingRewards that will checkpoint and distribute CVG reward.
        await (await contractsUsers.contracts.rewards.cvgRewards.connect(user1).writeStakingRewards()).wait();
        await (await contractsUsers.contracts.rewards.cvgRewards.connect(user1).writeStakingRewards()).wait();
        await (await contractsUsers.contracts.rewards.cvgRewards.connect(user1).writeStakingRewards()).wait();
        await (await contractsUsers.contracts.rewards.cvgRewards.connect(user1).writeStakingRewards()).wait();


        console.log(await cvg.balanceOf(await user1.getAddress())); //0
        //4. Claim the CVG reward
        await sdCRVStaking.connect(user1).claimCvgRewards(1);

        console.log(await cvg.balanceOf(await user1.getAddress())); //24704026442307693484766

        //5. Withdraw the deposited asset.
        await sdCRVStaking.connect(user1).withdraw(1, ethers.parseEther("10"));

        //6. Repay the loan.
    });
```

The above code can be tested in `claimMultipleCvgRewards.spec.ts` file

## Tool used

Manual Review

## Recommendation
The mitigation step is to introduce a waiting period to claim the CVG reward after the deposit. The waiting period can be as simple as a 5 or 10-minute window.
