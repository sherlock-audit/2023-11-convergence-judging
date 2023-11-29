Atomic Velvet Nuthatch

medium

# It is possible to mint a position for 0 MgCVG or 0 YSCVG

## Summary
Due to the fact that the only restriction on amount value to be submitted when minting a position on `LockingPositionService.sol#mintPosition()` is that amount is > 0. It is possible to mint positions that gives no power of governance or no shares to claim for minter.

## Vulnerability Detail
To mint a position Alice has to call `LockingPositionService.sol#mintPosition()` , there are several restriction about number of cycles to lock a position, however the only restriction about the amount of CVG to lock is that amount must be > 0 : 
```solidity
    function mintPosition(uint96 lockDuration,uint256 amount,uint64 ysPercentage,address receiver,bool isAddToManagedTokens) external onlyWalletOrWhiteListedContract {
        require(amount > 0, "LTE");
        require(ysPercentage <= MAX_PERCENTAGE, "YS_%_OVER_100");
        require(ysPercentage % RANGE_PERCENTAGE == 0, "YS_%_10_MULTIPLE");
        require(lockDuration <= MAX_LOCK, "MAX_LOCK_96_CYCLES");
        ICvgControlTower _cvgControlTower = cvgControlTower;
        uint96 actualCycle = uint96(_cvgControlTower.cvgCycle());
        uint96 endLockCycle = actualCycle + lockDuration;
        require(endLockCycle % TDE_DURATION == 0, "END_MUST_BE_TDE_MULTIPLE");
       ...
```

Alice can choose how she wants to allocate it's cvg tokens between mgCvg/veCVG and YScvg with percentage (minimum 10% for a repartition of tokens), however the amount of mgCvg/veCVG and YScvg that will be allocated to the NFT minted representing a locked position is calculated during the process of minting. 
The problem is that there is no minimum value of tokens required once a position is minted meaning that if Alice has a low value wallet or she wants to create a position for a low number of cycles it's possible that she will deposit 2000 cvg tokens with a repartion of 10% of YSCvg tokens but do not have any YSCvg tokens in her position due to rounding error and no requirement.

### Example 
1. Alice wants to lock `475 CVG` with a repartition of 10/90 during `2 cycles` so she is waiting for a small amount of MgCVG/veCVG  but she won't receive anything and the position will be locked for 2 weeks
2. Alice wants to lock `80 CVG` for `12 cycles` with a repartition of 10/90 during 12 cycles so she is waiting for a small amount of MgCVG/veCVG  but she won't receive anything and the position will be locked for 12 weeks
3. Alice wants to lock `2000 CVG` for `2 cycles` with a repartition of 90/10 so she is waiting for a small amount of YSCvg but she won't receive anything and the position will be locked for 2 weeks
4. Alice wants to lock `80 CVG` for `12 cycles` with a repartition of 90/10 so she is waiting for a small amount of YSCvg but she won't receive anything and the position will be locked for 12 weeks

I took some example using `mintPosition()` but ,of course, the same behavior is possible using `increaseLockAmount()` or `increaseLockTimeAndAmount()`

### POC 
Add this to increase-time-and-amount-test.spec.ts : 
```Typescript
     it("Finding 2 : Mints but receive 0 amount of MgCvg", async () => {
        // MINT 2 CYCLE AND GET 0 MGCVG 
        await increaseCvgCycle(contractUsers, 9);
        // Cycle 10
        await (await cvgContract.connect(user1).approve(lockingPositionService, ethers.parseEther("10"))).wait();
        await lockingPositionService.connect(user1).mintPosition(2, 475, 90, user1, false);
        let balanceOfMgCvg1 = await lockingPositionService.balanceOfMgCvg(1);
        expect(balanceOfMgCvg1).to.be.eq(0);

        // MINT 12 CYCLE BUT LOW AMOUNT AND GET 0 MGCVG 
        await increaseCvgCycle(contractUsers, 2);
        // Cycle 12
        await lockingPositionService.connect(user1).mintPosition(12, 79, 90, user1, false);
        let balanceOfMgCvg2 = await lockingPositionService.balanceOfMgCvg(2);
        expect(balanceOfMgCvg2).to.be.eq(0);
        
        
        // MINT 2 CYCLE AND GET 0 YS
        await increaseCvgCycle(contractUsers, 10);
        // Cycle 22
        await lockingPositionService.connect(user1).mintPosition(2, 2000, 10, user1, false);
        await increaseCvgCycle(contractUsers, 1);
        // Cycle 23
        let ysCvgBalance1 = await lockingPositionService.balanceOfYsCvgAt(3, 23);
        expect(ysCvgBalance1).to.be.eq(0);
        
                
        // MINT 12 CYCLE BUT LOW AMOUNT AND GET 0 YS 
        await increaseCvgCycle(contractUsers, 1);
        // Cycle 24
        await lockingPositionService.connect(user1).mintPosition(12, 79, 10, user1, false);
        await increaseCvgCycle(contractUsers, 1);
        // Cycle 25
        let ysCvgBalance2 = await lockingPositionService.balanceOfYsCvgAt(4, 25);
        expect(ysCvgBalance2).to.be.eq(0);
    });
```

## Impact
As there is no requirement on the amount of CVG tokens to be deposited and the number of cycles chosen by a minter can be 1 minimum if it respects some requirements, **a minter expecting some governance or shares tokens to be minted for it's position can be 0**

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L231

## Tool used

Manual Review

## Recommendation
1. Add a minimum number of governance or share tokens to be emitted for a locked position 
2. Add a minimum amount and minimum number of cycle to be submitted as parameter when minting a position