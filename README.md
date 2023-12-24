# Issue H-1: Lowering the gauge weight can disrupt accounting, potentially leading to both excessive fund distribution and a loss of funds. 

Source: https://github.com/sherlock-audit/2023-11-convergence-judging/issues/94 

## Found by 
0xDetermination, ZanyBonzy, bitsurfer, bughuntoor
## Summary
Similar issues were found by users [0xDetermination](https://github.com/code-423n4/2023-08-verwa-findings/issues/386) and [bart1e](https://github.com/code-423n4/2023-08-verwa-findings/issues/206) in the Canto veRWA audit, which uses a similar gauge controller type.
## Vulnerability Detail
 - When the _change_gauge_weight function is called, the `points_weight[addr][next_time].bias` and`time_weight[addr]` are updated - the slope is not.  
```L568
def _change_gauge_weight(addr: address, weight: uint256):
    # Change gauge weight
    # Only needed when testing in reality
    gauge_type: int128 = self.gauge_types_[addr] - 1
    old_gauge_weight: uint256 = self._get_weight(addr)
    type_weight: uint256 = self._get_type_weight(gauge_type)
    old_sum: uint256 = self._get_sum(gauge_type)
    _total_weight: uint256 = self._get_total()
    next_time: uint256 = (block.timestamp + WEEK) / WEEK * WEEK

    self.points_weight[addr][next_time].bias = weight
    self.time_weight[addr] = next_time

    new_sum: uint256 = old_sum + weight - old_gauge_weight
    self.points_sum[gauge_type][next_time].bias = new_sum
    self.time_sum[gauge_type] = next_time

    _total_weight = _total_weight + new_sum * type_weight - old_sum * type_weight
    self.points_total[next_time] = _total_weight
    self.time_total = next_time

    log NewGaugeWeight(addr, block.timestamp, weight, _total_weight)
  ```  
  
 - The equation  ***f(t) = c - mx*** represents the gauge's decay equation before the weight is reduced. In this equation, `m` is the slope. After the weight is reduced by an amount `k` using the `change_gauge_weight` function, the equation becomes ***f(t) = c - k - mx*** The slope m remains unchanged, but the t-axis intercept changes from ***t<sub>1</sub> = c/m*** to ***t<sub>2</sub>  = (c-k)/m***.
 - Slope adjustments that should be applied to the global slope when decay reaches 0 are stored in the `changes_sum` hashmap. And is not affected by changes in gauge weight. Consequently, there's a time window ***t<sub>1</sub> - t<sub>2</sub>*** during which the earlier slope changes applied to the global state when user called `vote_for_gauge_weights` function remains applied even though they should have been subtracted. This in turn creates a situation in which the global weightis less than the sum of the individual gauge weights, resulting in an accounting error.
 - So, in the `CvgRewards` contract when the `writeStakingRewards` function invokes the `_checkpoint`, which subsequently triggers the `gauge_relative_weight_writes` function for the relevant time period, the calculated relative weight becomes inflated, leading to an increase in the distributed rewards. If all available rewards are distributed before the entire array is processed, the remaining users will receive no rewards."
 - The issue mainly arises when a gauge's weight has completely diminished to zero. This is certain to happen if a gauge with a non-zero bias, non-zero slope, and a t-intercept exceeding the current time  is killed using `kill_gauge` function.
 - Additionally, decreasing a gauge's weight introduces inaccuracies in its decay equation, as is evident in the t-intercept.
## Impact
The way rewards are calculated is broken, leading to an uneven distribution of rewards, with some users receiving too much and others receiving nothing.
## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Locking/GaugeController.vy#L568C1-L590C1

https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L189

https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L235C1-L235C91

https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Locking/GaugeController.vy#L493

https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Locking/GaugeController.vy#L456C1-L475C17

https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Locking/GaugeController.vy#L603C1-L611C54
## Tool used

Manual Review

## Recommendation

Disable weight reduction, or only allow reset to 0.



## Discussion

**0xR3vert**

Hello,

Thanks a lot for your attention.

Thank you for your insightful observation. Upon thorough examination, we acknowledge that such an occurrence could indeed jeopardize the protocol. We are currently exploring multiple solutions to address this issue.
We are considering removing the function change_gauge_weight entirely and not distributing CVG inflation on killed gauges, similar to how Curve Protocol handles their gauges.

Therefore, in conclusion, we must consider your issue as valid.

Regards,
Convergence Team

# Issue H-2: LockingPositionDelegate::manageOwnedAndDelegated unchecked duplicate tokenId allow metaGovernance manipulation 

Source: https://github.com/sherlock-audit/2023-11-convergence-judging/issues/126 

## Found by 
bughuntoor, cawfree, cergyk, hash, lemonmon, r0ck3tz
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



## Discussion

**shalbe-cvg**

Hello,

Thanks a lot for your attention.

After an in-depth review, we have to consider your issue as Confirmed.
We will add a check on the values contained in the 3 arrays to ensure duplicates are taken away before starting the process.

Regards,
Convergence Team

# Issue H-3: LockPositionDelegate doesn't clear delegates on transfer which can be used to honeypot buyers 

Source: https://github.com/sherlock-audit/2023-11-convergence-judging/issues/175 

## Found by 
0x52, HChang26, Inspex, bughuntoor
## Summary

When transferring locked positions, the delegates are not cleared from the previous owner. This can be used to honeypot buyers. A malicious user would list their token which a large TDE claim available. They can then frontrun the legitimate user with a call setting themselves as the ysCVG delegate. Then after they can claim the TDE at the expense of the new owner who paid extra for the token since it was entitled to a TDE claim.

## Vulnerability Detail

[LockingPositionDelegate.sol#L237-L240](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionDelegate.sol#L237-L240)

    function delegateYsCvg(uint256 _tokenId, address _to) external onlyTokenOwner(_tokenId) {
        delegatedYsCvg[_tokenId] = _to;
        emit DelegateShare(_tokenId, _to);
    }

delegateYsCvg allows the owner of the token to set the receiver of yield sharing. It is also the only function that sets the delegatedYsCvg mapping. LockingPositionManager uses the standard implementation of transfers which never clears out this delegation when the token is transferred. As a result of this, all delegation will persist across transfers. This enables the honeypot scenario described above. 

## Impact

Users can be honeypotted by malicious token sellers

## Code Snippet

[SdtStakingPositionManager.sol#L21](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionManager.sol#L21)

## Tool used

Manual Review

## Recommendation

Overwrite the afterTokenTransfer method to forcefully clear delegation when a token is transferred



## Discussion

**shalbe-cvg**

Hello,

Thanks a lot for your attention.

After an in-depth review, we have to consider your issue as Confirmed.
You're right, after transferring (whether it's a purchase on a marketplace or a simple transfer) we should ensure that all delegatees associated to the token are removed. Otherwise, as you said, buyer (i.e. receiver) might not be aware of the delegatees and could be subject to a honeypot attack.

However we have decided to disagree with the severity and put it as a Medium issue.

Regards,
Convergence Team

**nevillehuang**

Hi @0xR3vert @shalbe-cvg @walk-on-me,

I think this could remain as high severity given explicit TDE claim can be hijacked. Why do you think this constitutes medium severity?

**nevillehuang**

Keeping this as high severity given delegate logic allows malicious user to perform the honeypot without any external limitations.

# Issue H-4: Tokens that are both bribes and StakeDao gauge rewards will cause loss of funds 

Source: https://github.com/sherlock-audit/2023-11-convergence-judging/issues/182 

## Found by 
0x52, GimelSec, detectiveking, lemonmon
## Summary

When SdtStakingPositionService is pulling rewards and bribes from buffer, the buffer will return a list of tokens and amounts owed. This list is used to set the rewards eligible for distribution. Since this list is never check for duplicate tokens, a shared bribe and reward token would cause the token to show up twice in the list. The issue it that _sdtRewardsByCycle is set and not incremented which will cause the second occurrence of the token to overwrite the first and break accounting. The amount of token received from the gauge reward that is overwritten will be lost forever.

## Vulnerability Detail

In [L559](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L559) of SdtStakingPositionService it receives a list of tokens and amount from the buffer.

[SdtBuffer.sol#L90-L168](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/StakeDAO/SdtBuffer.sol#L90-L168)

        ICommonStruct.TokenAmount[] memory bribeTokens = _sdtBlackHole.pullSdStakingBribes(
            processor,
            _processorRewardsPercentage
        );

        uint256 rewardAmount = _gaugeAsset.reward_count();

        ICommonStruct.TokenAmount[] memory tokenAmounts = new ICommonStruct.TokenAmount[](
            rewardAmount + bribeTokens.length
        );

        uint256 counter;
        address _processor = processor;
        for (uint256 j; j < rewardAmount; ) {
            IERC20 token = _gaugeAsset.reward_tokens(j);
            uint256 balance = token.balanceOf(address(this));
            if (balance != 0) {
                uint256 fullBalance = balance;

                ...

                token.transfer(sdtRewardsReceiver, balance);

              **@audit token and amount added from reward_tokens pulled directly from gauge**

                tokenAmounts[counter++] = ICommonStruct.TokenAmount({token: token, amount: balance});
            }

            ...

        }

        for (uint256 j; j < bribeTokens.length; ) {
            IERC20 token = bribeTokens[j].token;
            uint256 amount = bribeTokens[j].amount;

          **@audit token and amount added directly with no check for duplicate token**

            if (amount != 0) {
                tokenAmounts[counter++] = ICommonStruct.TokenAmount({token: token, amount: amount});

            ...

        }

SdtBuffer#pullRewards returns a list of tokens that is a concatenated array of all bribe and reward tokens. There is not controls in place to remove duplicates from this list of tokens. This means that tokens that are both bribes and rewards will be duplicated in the list.

[SdtStakingPositionService.sol#L561-L577](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L561-L577)

        for (uint256 i; i < _rewardAssets.length; ) {
            IERC20 _token = _rewardAssets[i].token;
            uint256 erc20Id = _tokenToId[_token];
            if (erc20Id == 0) {
                uint256 _numberOfSdtRewards = ++numberOfSdtRewards;
                _tokenToId[_token] = _numberOfSdtRewards;
                erc20Id = _numberOfSdtRewards;
            }

          **@audit overwrites and doesn't increment causing duplicates to be lost**            

            _sdtRewardsByCycle[_cvgStakingCycle][erc20Id] = ICommonStruct.TokenAmount({
                token: _token,
                amount: _rewardAssets[i].amount
            });
            unchecked {
                ++i;
            }
        }

When storing this list of rewards, it overwrites _sdtRewardsByCycle with the values from the returned array. This is where the problem arises because duplicates will cause the second entry to overwrite the first entry. Since the first instance is overwritten, all funds in the first occurrence will be lost permanently.

## Impact

Tokens that are both bribes and rewards will be cause tokens to be lost forever 

## Code Snippet

[SdtStakingPositionService.sol#L550-L582](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtStakingPositionService.sol#L550-L582)

## Tool used

Manual Review

## Recommendation

Either sdtBuffer or SdtStakingPositionService should be updated to combine duplicate token entries and prevent overwriting.



## Discussion

**0xR3vert**

Hello,

Thanks a lot for your attention.

Absolutely, if we kill a gauge or change a type weight during the distribution, it would distribute wrong amounts, even though we're not planning to do that. We can make sure it doesn't happen by doing what you said: locking those functions to avoid any problems.

Therefore, in conclusion, we must consider your issue as a valid.

Regards,
Convergence Team

# Issue H-5: Division difference can result in a revert when claiming treasury yield and excess rewards to some users 

Source: https://github.com/sherlock-audit/2023-11-convergence-judging/issues/190 

## Found by 
cergyk, hash, vvv
## Summary
Different ordering of calculations are used to compute `ysTotal` in different situations. This causes the totalShares tracked to be less than the claimable amount of shares  

## Vulnerability Detail
`ysTotal` is calculated differently when adding to `totalSuppliesTracking` and when computing `balanceOfYsCvgAt`.
When adding to `totalSuppliesTracking`, the calculation of `ysTotal` is as follows:

```solidity
        uint256 cvgLockAmount = (amount * ysPercentage) / MAX_PERCENTAGE;
        uint256 ysTotal = (lockDuration * cvgLockAmount) / MAX_LOCK;
```

In `balanceOfYsCvgAt`, `ysTotal` is calculated as follows

```solidity
        uint256 ysTotal = (((endCycle - startCycle) * amount * ysPercentage) / MAX_PERCENTAGE) / MAX_LOCK;
```

This difference allows the `balanceOfYsCvgAt` to be greater than what is added to `totalSuppliesTracking`

### POC
```solidity
  startCycle 357
  endCycle 420
  lockDuration 63
  amount 2
  ysPercentage 80
```

Calculation in `totalSuppliesTracking` gives:
```solidity
        uint256 cvgLockAmount = (2 * 80) / 100; == 1
        uint256 ysTotal = (63 * 1) / 96; == 0
```
Calculation in `balanceOfYsCvgAt` gives:
```solidity
        uint256 ysTotal = ((63 * 2 * 80) / 100) / 96; == 10080 / 100 / 96 == 1
```

### Example Scenario
Alice, Bob and Jake locks cvg for 1 TDE and obtains rounded up `balanceOfYsCvgAt`. A user who is aware of this issue can exploit this issue further by using `increaseLockAmount` with small amount values by which the total difference difference b/w the user's calculated `balanceOfYsCvgAt` and the accounted amount in `totalSuppliesTracking` can be increased. Bob and Jake claims the reward at the end of reward cycle. When Alice attempts to claim rewards, it reverts since there is not enough reward to be sent.

## Impact
This breaks the shares accounting of the treasury rewards. Some user's will get more than the actual intended rewards while the last withdrawals will result in a revert

## Code Snippet
`totalSuppliesTracking` calculation

In `mintPosition`
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L261-L263

In `increaseLockAmount`
https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L339-L345

In `increaseLockTimeAndAmount`
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L465-L470

`_ysCvgCheckpoint`
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L577-L584

`balanceOfYsCvgAt` calculation
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L673-L675

## Tool used

Manual Review

## Recommendation
Perform the same calculation in both places

```diff
+++                     uint256 _ysTotal = (_extension.endCycle - _extension.cycleId)* ((_extension.cvgLocked * _lockingPosition.ysPercentage) / MAX_PERCENTAGE) / MAX_LOCK;
---     uint256 ysTotal = (((endCycle - startCycle) * amount * ysPercentage) / MAX_PERCENTAGE) / MAX_LOCK;
```



## Discussion

**walk-on-me**

Hello

Indeed this is a real problem due the way that the invariant : 
*Sum of all balanceOfYsCvg > totalSupply*

And so some positions will become not claimable on the `YsDistributor`.

We'll correct this by computing the same way the ysTotal & ysPartial on the balanceYs & ysCheckpoint

Very nice finding, it'd break the claim for the last users to claim.

# Issue H-6: Killing a gague can lead to bricking of the protocol 

Source: https://github.com/sherlock-audit/2023-11-convergence-judging/issues/192 

## Found by 
Inspex, bughuntoor, hash
## Summary
Slopes are not handled correctly when a gauge is killed. This can cause incorrect accounting for `sum` and `total` weights and lead to bricking the protocol in certain scenarios.

## Vulnerability Detail
Points are accounted using `bias`,`slope` and `slope end time`. The bias is gradually decreased by its slope eventually reaching 0 at the `slope end time / lock end time`. In case the bias is suddenly reduced without handling/removing its slope, there will reach a time before the `slope end time` where the bias will drop below 0. 
`points_sum` uses points to calculate the sum of individual gauge weights of a particular type at any time. The above case of bias dropping below 0 is handled as follows in `_get_sum`:
```solidity
            d_bias: uint256 = pt.slope * WEEK
            if pt.bias > d_bias:
                pt.bias -= d_bias
                d_slope: uint256 = self.changes_sum[gauge_type][t]
                pt.slope -= d_slope
            else:
                pt.bias = 0
                pt.slope = 0
```
Whenever bias drops below 0 the slope is also made 0 without removing the `slope end accounting` in `changes_sum`. Afterwards, if a scenario occurs where the pt.bias regains a value greater than the `pt.slope * WEEK`, the true part of the if condition executes which can cause it to revert if at any point `self.changes_sum[gauge_type][t]` is greater than the current `pt.slope`.

The sudden drop in `points_sum` bias can happen due to the following:
1. killing of a gauge
2. admin calls `change_gauge_weight` with a lower value than current

From this moment the accounting of `points_sum` is broken and depending on the current `slope`, `changes_sum`, `bias` and activity following this, the `points_sum.bias` can go to 0 ( an attacker wanting to cause grief has the option to accelarate the dropping of bias to 0 by front-running a kill and increasing its slope )

Once bias goes to 0, it can regain a value greater than the `pt.slope * WEEK` due to the following:

1. another gauge of the same type is added later with a weight
2. a user votes for a gauge of the same type
3. admin calling the `change_gauge_weight` increasing a gauge's weight

Scenario 2 is almost sure to happen following which all calls to `_get_sum` can revert which will cause the protocol to brick since even the cycle update involves a call to `_get_sum`

### POC
```solidity
Gauges A and B are of same type

At t = 0
a_bias = b_bias = 100 , a_slope = b_slope = 1 , slope_end : t = 100 
sum_bias = 200 , sum_slope = 2 , slope_end : t = 100

At t = 50 before kill
a_bias = b_bias = 50 , a_slope = b_slope = 1 
sum_bias = 100 , sum_slope = 2

at t = 50 kill A
a_bias = 0 , a_slope = 1
b_bias = 50 , b_slope = 1
sum_bias = 50 , sum_slope = 2

at t = 75
b_bias = 25 , b_slope = 1
sum_bias = 0 , sum_slope = 0 (pt.slope set to 0 when bias !> d.slope)

at t = 75
another user votes for B with bias = 100 and slope = 1
b_bias = 125 , b_slope = 2
sum_bias = 100 , sum_slope = 1

at t = 100
sum_slope -= 2 ( a_slope_end = b_slope_end : t = 100)

this will cause _get_sum to revert
```

Runnable foundry test gist link:
https://gist.github.com/10xhash/b65867b99841a88e078e34f094fc0554

## Impact
Bricking of the protocol,locked user funds , incorrect `sum` and `total` weight values

## Code Snippet
`_change_gauge_weight` sets weight directly without handling slope
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/GaugeController.vy#L568-L582

`kill_gauge` calls `_change_gauge_weight`
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/GaugeController.vy#L603C5-L609

## Tool used

Manual Review

## Recommendation
When killing the gauge, decrease its slope from the sum and iterate over all future weeks till the max limit and remove any gauge-associated-slope changes from the `changes_sum`. Handle the user removing a vote from a killed gauge seperately from the general vote function.



## Discussion

**0xR3vert**

Hello,

Thanks a lot for your attention.

Thank you for your insightful observation. Upon thorough examination, we acknowledge that such an occurrence could indeed jeopardize the protocol. We are currently exploring multiple solutions to address this issue.
We are considering removing the function change_gauge_weight entirely and not distributing CVG inflation on killed gauges, similar to how Curve Protocol handles their gauges.

Therefore, in conclusion, we must consider your issue as valid.

Regards,
Convergence Team

# Issue M-1: If the multiple calls to `writeStakingRewards` cross a week's end, it will result in unfair distribution of rewards 

Source: https://github.com/sherlock-audit/2023-11-convergence-judging/issues/10 

## Found by 
bughuntoor, hash
## Summary
If the multiple calls to `writeStakingRewards` cross a week's end, it will result in unfair distribution of rewards

## Vulnerability Detail
The first call to `writeStakingRewards` calls `checkpoints` which makes sure all gauges are checkpointed up to the current week. However, there rises a issue if after `_checkpoints` the week end is crossed. This would allow for not up-to-date values of the gauges to be used. If the values are already added to the `totalWeightLocked`, its value will be inflated (as the gauge weights can only decrease in the time as votes are locked and time passes). 
```solidity
    function _setTotalWeight() internal {
        ICvgControlTower _cvgControlTower = cvgControlTower;
        IGaugeController _gaugeController = _cvgControlTower.gaugeController();
        uint128 _cursor = cursor;
        uint128 _totalGaugeNumber = uint128(gauges.length);

        /// @dev compute the theoric end of the chunk
        uint128 _maxEnd = _cursor + cvgRewardsConfig.maxLoopSetTotalWeight;
        /// @dev compute the real end of the chunk regarding the length of staking contracts
        uint128 _endChunk = _maxEnd < _totalGaugeNumber ? _maxEnd : _totalGaugeNumber;

        /// @dev if last chunk of the total weighted locked processs
        if (_endChunk == _totalGaugeNumber) {
            /// @dev reset the cursor to 0 for _distributeRewards
            cursor = 0;
            /// @dev set the step as DISTRIBUTE for reward distribution
            state = State.DISTRIBUTE;
        } else {
            /// @dev setup the cursor at the index start for the next chunk
            cursor = _endChunk;
        }

        totalWeightLocked += _gaugeController.get_gauge_weight_sum(_getGaugeChunk(_cursor, _endChunk));

        /// @dev emit the event only at the last chunk
        if (_endChunk == _totalGaugeNumber) {
            emit SetTotalWeight(_cvgControlTower.cvgCycle(), totalWeightLocked);
        }
    }
```

Then if any gauges have manually been checkpointed before the subsequent call to `_distributeCvgRewards` , it would mean that the sum of all weights of the gauges will be less than `totalWeightLocked`, meaning there will be underdistribution of rewards. If no gauges have been manually checkpointed, it would simply mean unfair distribution of rewards (as the values are not up-to-date).
```solidity
    function _distributeCvgRewards() internal {
        ICvgControlTower _cvgControlTower = cvgControlTower;
        IGaugeController gaugeController = _cvgControlTower.gaugeController();

        uint256 _cvgCycle = _cvgControlTower.cvgCycle();

        /// @dev number of gauge in GaugeController
        uint128 _totalGaugeNumber = uint128(gauges.length);
        uint128 _cursor = cursor;

        uint256 _totalWeight = totalWeightLocked;
        /// @dev cursor of the end of the actual chunk
        uint128 cursorEnd = _cursor + cvgRewardsConfig.maxChunkDistribute;

        /// @dev if the new cursor is higher than the number of gauge, cursor become the number of gauge
        if (cursorEnd > _totalGaugeNumber) {
            cursorEnd = _totalGaugeNumber;
        }

        /// @dev reset the cursor if the distribution has been done
        if (cursorEnd == _totalGaugeNumber) {
            cursor = 0;

            /// @dev reset the total weight of the gauge
            totalWeightLocked = 0;

            /// @dev update the states to the control_tower sync
            state = State.CONTROL_TOWER_SYNC;
        }
        /// @dev update the global cursor in order to be taken into account on next chunk
        else {
            cursor = cursorEnd;
        }

        uint256 stakingInflation = stakingInflationAtCycle(_cvgCycle);
        uint256 cvgDistributed;
        InflationInfo[] memory inflationInfos = new InflationInfo[](cursorEnd - _cursor);
        address[] memory addresses = _getGaugeChunk(_cursor, cursorEnd);
        /// @dev fetch weight of gauge relative to the cursor
        uint256[] memory gaugeWeights = gaugeController.get_gauge_weights(addresses);
        for (uint256 i; i < gaugeWeights.length; ) {
            /// @dev compute the amount of CVG to distribute in the gauge
            cvgDistributed = (stakingInflation * gaugeWeights[i]) / _totalWeight;

            /// @dev Write the amount of CVG to distribute in the staking contract
            ICvgAssetStaking(addresses[i]).processStakersRewards(cvgDistributed);

            inflationInfos[i] = InflationInfo({
                gauge: addresses[i],
                cvgDistributed: cvgDistributed,
                gaugeWeight: gaugeWeights[i]
            });

            unchecked {
                ++i;
            }
        }

        emit EventChunkWriteStakingRewards(_cvgCycle, _totalWeight, inflationInfos);
    }
```


Note: since the requirement on calling `checkpoint` is that at least 7 days have passed since the last distribution, it would mean that the delta of the checkpoint and the end of the week will gradually decrease every week, up until we once have a distribution crossing over a week's end. The issue above is bound to happen given long-enough timeframe., 

## Impact
Unfair distribution of rewards. Possible permanent loss of rewards.

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L279C1-L338C6

## Tool used

Manual Review

## Recommendation
Add time constraints to `writeStakingRewards` in order to make sure it does not happen close to the end of the week 



## Discussion

**walk-on-me**

Hello, 

Thanks a lot for your attention.

After a deep review of the issues, we have to consider all of your issues as valid.

We indeed noticed that if we don't synchronize our CvgCycle with the WEEKS, we could have some issues of desynchronization in the long term if the CVG cycle is not triggered immediatly after 1 week.

For example :
-   https://github.com/sherlock-audit/2023-11-convergence-judging/issues/10 && https://github.com/sherlock-audit/2023-11-convergence-judging/issues/193, mention that if the distribution of $CVG occurs just before a WEEK and finished just after, some gauges will have less CVG distributed.
-   https://github.com/sherlock-audit/2023-11-convergence-judging/issues/14, mention that some user can get some free mgCVG for 1 WEEK less than other users
-   https://github.com/sherlock-audit/2023-11-convergence-judging/issues/101, mention that in the last cycle of a lock, an user cannot extend his lock during the time between the last WEEK and the nextCvgCycle. 
-   https://github.com/sherlock-audit/2023-11-convergence-judging/issues/136 && https://github.com/sherlock-audit/2023-11-convergence-judging/issues/215, mention that in the last cycle of a lock, an user is locked for 1 more cycle than he planned
-   https://github.com/sherlock-audit/2023-11-convergence-judging/issues/178  , mention that extending a lock can be impossible
-   https://github.com/sherlock-audit/2023-11-convergence-judging/issues/59  , mention that during the mint of a position, the lock is done for one extra cycle on the veCVG

We are going to implement the solution for all of these issues that is to round our CVGCycle on the WEEK like veCVG is doing.
Regards,
Convergence Team

# Issue M-2: `mgCvg` balances are wrongfully calculated 

Source: https://github.com/sherlock-audit/2023-11-convergence-judging/issues/14 

## Found by 
bughuntoor
## Summary
Users with significant difference in their locks may have the same `mgCVG` voting power 

## Vulnerability Detail
The `mgCvgCreated` is based on the amount a user has used for voting and their `lockDuration`. However, due to rounding down within the VotingEscrow contract, same users may get unfairly rewarded in comparison to others. 
Let's look at the code responsible for the `mgCvgCreated` amount within `mintPosition`
```solidity
        if (ysPercentage != MAX_PERCENTAGE) {
            uint256 amountVote = amount * (MAX_PERCENTAGE - ysPercentage);

            /** @dev Timestamp of the end of locking. */
            _cvgControlTower.votingPowerEscrow().create_lock(
                tokenId,
                amountVote / MAX_PERCENTAGE,
                block.timestamp + (lockDuration + 1) * 7 days
            );
            /// @dev compute the amount of mgCvg
            _mgCvgCreated = (amountVote * lockDuration) / (MAX_LOCK * MAX_PERCENTAGE);

            /// @dev Automatically add the veCVG and mgCVG in the balance taken from Snapshot.
            if (isAddToManagedTokens) {
                _cvgControlTower.lockingPositionDelegate().addTokenAtMint(tokenId, receiver);
            }
        }
```
As we know, the voting escrow contract round downs the lock time to the nearest week. However, this is not accounted for when calculating the `mgCvgCreated`. The amount is entirely based on the `lockDuration`. 
Consider the following scenario: 
If two users with equal stake both call `mintPosition` with the same `lockDuration` of (let's say) 2 weeks), but one calls it at the beginning of the week and the other one calls it at the end of the week. Because of the rounding down to the nearest week, one of the users will have locked their tokens for ~2 weeks, while the other one will have locked them for ~1 week. However, both users will receive the same amount of `mgCvg`. This is unfair for both users.

## Impact
Unfair calculation of `mgCvg`

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L266C1-L282C10

## Tool used

Manual Review

## Recommendation
Base the `mgCvg` calculated on the block.timestamp

# Issue M-3: Certain functions should not be usable when `GaugeController` is locked. 

Source: https://github.com/sherlock-audit/2023-11-convergence-judging/issues/18 

## Found by 
bughuntoor
## Summary
Possible unfair over/under distribution of rewards

## Vulnerability Detail
When `writeStakingRewards` is invoked for the first time it calls `_checkpoints` which sets the lock in the GaugeController to true. What this does is it doesn't allow for any new vote changes. The idea behind it is that until the rewards are fully distributed there are no changes in the gauges' weights so the distribution of rewards is correct. 
However, there are multiple unrestricted functions which can alter the outcome of the rewards and result in not only unfair distribution, but also to many overdistributed or underdistributed rewards.
```solidity
    function _setTotalWeight() internal {
        ICvgControlTower _cvgControlTower = cvgControlTower;
        IGaugeController _gaugeController = _cvgControlTower.gaugeController();
        uint128 _cursor = cursor;
        uint128 _totalGaugeNumber = uint128(gauges.length);

        /// @dev compute the theoric end of the chunk
        uint128 _maxEnd = _cursor + cvgRewardsConfig.maxLoopSetTotalWeight;
        /// @dev compute the real end of the chunk regarding the length of staking contracts
        uint128 _endChunk = _maxEnd < _totalGaugeNumber ? _maxEnd : _totalGaugeNumber;

        /// @dev if last chunk of the total weighted locked processs
        if (_endChunk == _totalGaugeNumber) {
            /// @dev reset the cursor to 0 for _distributeRewards
            cursor = 0;
            /// @dev set the step as DISTRIBUTE for reward distribution
            state = State.DISTRIBUTE;
        } else {
            /// @dev setup the cursor at the index start for the next chunk
            cursor = _endChunk;
        }

        totalWeightLocked += _gaugeController.get_gauge_weight_sum(_getGaugeChunk(_cursor, _endChunk));

        /// @dev emit the event only at the last chunk
        if (_endChunk == _totalGaugeNumber) {
            emit SetTotalWeight(_cvgControlTower.cvgCycle(), totalWeightLocked);
        }
    }
```

If any of `change_gauge_weight` `change_type_weight` or  is called after the `totalWeightLocked` is calculated, it will result in incorrect distribution of rewards. When `_distributeCvgRewards` is called, some gauges may not have the same value that has been used to calculate the `totalWeightLocked` and this may result in distribution too many or too little rewards. It also gives an unfair advantage/disadvantage to the different gauges. 
```solidity
    function _distributeCvgRewards() internal {
        ICvgControlTower _cvgControlTower = cvgControlTower;
        IGaugeController gaugeController = _cvgControlTower.gaugeController();

        uint256 _cvgCycle = _cvgControlTower.cvgCycle();

        /// @dev number of gauge in GaugeController
        uint128 _totalGaugeNumber = uint128(gauges.length);
        uint128 _cursor = cursor;

        uint256 _totalWeight = totalWeightLocked;
        /// @dev cursor of the end of the actual chunk
        uint128 cursorEnd = _cursor + cvgRewardsConfig.maxChunkDistribute;

        /// @dev if the new cursor is higher than the number of gauge, cursor become the number of gauge
        if (cursorEnd > _totalGaugeNumber) {
            cursorEnd = _totalGaugeNumber;
        }

        /// @dev reset the cursor if the distribution has been done
        if (cursorEnd == _totalGaugeNumber) {
            cursor = 0;

            /// @dev reset the total weight of the gauge
            totalWeightLocked = 0;

            /// @dev update the states to the control_tower sync
            state = State.CONTROL_TOWER_SYNC;
        }
        /// @dev update the global cursor in order to be taken into account on next chunk
        else {
            cursor = cursorEnd;
        }

        uint256 stakingInflation = stakingInflationAtCycle(_cvgCycle);
        uint256 cvgDistributed;
        InflationInfo[] memory inflationInfos = new InflationInfo[](cursorEnd - _cursor);
        address[] memory addresses = _getGaugeChunk(_cursor, cursorEnd);
        /// @dev fetch weight of gauge relative to the cursor
        uint256[] memory gaugeWeights = gaugeController.get_gauge_weights(addresses);
        for (uint256 i; i < gaugeWeights.length; ) {
            /// @dev compute the amount of CVG to distribute in the gauge
            cvgDistributed = (stakingInflation * gaugeWeights[i]) / _totalWeight;

            /// @dev Write the amount of CVG to distribute in the staking contract
            ICvgAssetStaking(addresses[i]).processStakersRewards(cvgDistributed);

            inflationInfos[i] = InflationInfo({
                gauge: addresses[i],
                cvgDistributed: cvgDistributed,
                gaugeWeight: gaugeWeights[i]
            });

            unchecked {
                ++i;
            }
        }

        emit EventChunkWriteStakingRewards(_cvgCycle, _totalWeight, inflationInfos);
    }
```


## Impact
Unfair distribution of rewards. Over/underdistributing rewards.

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L244C1-L272C6

## Tool used

Manual Review

## Recommendation
Add a lock to `change_gauge_weight` `change_type_weight` 



## Discussion

**0xR3vert**

Hello,

Thanks a lot for your attention.

Absolutely, if we kill a gauge or change a type weight during the distribution, it would distribute wrong amounts, even though we're not planning to do that. We can make sure it doesn't happen by doing what you said: locking those functions to avoid any problems.

Therefore, in conclusion, we must consider your issue as a valid.

Regards,
Convergence Team

**nevillehuang**

I will keep this as medium as although on first look this could be "admin error", as sponsor mentioned, honest users claiming during killing of a gauge or weight change can result in inaccurate result distribution.

# Issue M-4: deposit in bond contract in CvgUtility has no possibility to express slippage control/&expiry 

Source: https://github.com/sherlock-audit/2023-11-convergence-judging/issues/93 

## Found by 
0xGoodess
## Summary
deposit in bond contract in CvgUtility has no possibility to express slippage control

## Vulnerability Detail
BondDepository allows depositor to deposit a token in exchange for CVG bond with discount, discount depends on min/max, bond duration as well as totalCvg already minted. The discount is applied on the current price of CVG.

However, since both the price of CVG and deposited token are fetched from the oracle in CVGControlTower, they are real-time and dynamic, and the users may receive the final CVG amount different, if the price of CVG, as well as the price of their input token fluctuate.

Right now, in both the BondDepository, as well as the integration contract, CvgUtility, does not have any argument for the depositor to express the minCVGReceived. This creates an issue since the user could suffer from price manipulation of CVG/& deposited token.

Consider:
1. Alice wants to deposit stETH in exchange for CVG bond. She calls depositBond on CvGUtility with 1 stETH, price of CVG is 0.001 stETH currently.
2. After some time T, price of CVG becomes 0.002 stETH, the deposit is executed and got Alice 500 CVG instead of 1000, which is the price at the time when Alice requested deposit.


```solidity
    function deposit(uint256 tokenId, uint256 amount, address receiver) external whenNotPaused {
        ICvgControlTower _cvgControlTower = cvgControlTower;
        IBondPositionManager bondPositionManager = _cvgControlTower.bondPositionManager();
        require(amount > 0, "LTE");
        ICvg _cvg = cvg;

        IBondStruct.BondParams memory _bondParams = bondParams;

        uint256 _tokenId;
        if (tokenId == 0) {
            _tokenId = bondPositionManager.nextId();
        } else {
            address tokenOwner = msg.sender == _cvgControlTower.cvgUtilities() ? receiver : msg.sender;
            require(bondPositionManager.ownerOf(tokenId) == tokenOwner, "TOKEN_NOT_OWNED");
            require(bondPositionManager.bondPerTokenId(tokenId) == address(this), "WRONG_BOND_DEPOSITORY");
            require(bondPositionManager.unlockingTimestampPerToken(tokenId) < block.timestamp, "TOKEN_TIMELOCKED");
            _tokenId = tokenId;
        }
        /// @dev Bond expired after 1/2 days x numberOfPeriods from the creation
        require(block.timestamp <= startBondTimestamp + _bondParams.bondDuration, "BOND_INACTIVE");

        (uint256 cvgPrice, uint256 assetPrice) = _cvgControlTower.cvgOracle().getAndVerifyTwoPrices(
            address(_cvg),
            _bondParams.token
        );
```
## Impact
depositor may suffer MEV or unexpected discrepenacy during deposit for CVG bond.
## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Bond/BondDepository.sol#L139

## Tool used

Manual Review

## Recommendation
Consider to add minCVGreceived as well as expiry control for the deposit function at Bond Depository and or CvgUtility for deposit.



## Discussion

**shalbe-cvg**

Hello,

Thanks a lot for your attention.

After an in-depth review and even though this contract (BondDepository) is Out of Scope, we have to consider your issue as Confirmed.
We will implement a check on the minimum amount that should be put into the Bond position depending on the deposited amount and the discount.

Regards,
Convergence Team

# Issue M-5: Risk of Permanently Lost Rewards due to Claiming Before Treasury Deposit 

Source: https://github.com/sherlock-audit/2023-11-convergence-judging/issues/110 

## Found by 
0xkaden, HChang26, wangxx2026
## Summary
When Treasury Bond `depositMultipleToken()` in current TDE and users try to `claimRewards()` for current TDE, users may lose their rightful rewards.

## Vulnerability Detail
The mechanism responsible for TreasuryBonds depositing rewards into `YsDistributor.sol` through the `depositMultipleToken()` function calculates the TDE for reward distribution based on the current cvgCycle. Each cvgCycle spans a week, and `TDE_DURATION` equals 12 weeks or 12 cvgCycles. This calculation of `_actualTDE` results in two potential outcomes: the current TDE or the subsequent one.
```solidity
        uint256 _actualCycle = cvgControlTower.cvgCycle();
        uint256 _actualTDE = _actualCycle % TDE_DURATION == 0
            ? _actualCycle / TDE_DURATION
            : (_actualCycle / TDE_DURATION) + 1;
```
The issue arises when users use `claimRewards()` for the current TDE before the execution of `depositMultipleToken()` within the same block. If, coincidentally, the TDE calculation lands on the current TDE and `claimRewards()` is executed first, users risk permanently losing their rewards. Once a specific TDE id is claimed, users are unable to make further claims, effectively trapping their rewards within the `YsDistributor.sol` contract.
```solidity
    function claimRewards(uint256 tokenId, uint256 tdeId, address receiver, address operator) external {
        ICvgControlTower _cvgControlTower = cvgControlTower;
        ILockingPositionService _lockingPositionService = lockingPositionService;
        ILockingPositionManager _lockingPositionManager = lockingPositionManager;
        require(_lockingPositionManager.unlockingTimestampPerToken(tokenId) < block.timestamp, "TOKEN_TIMELOCKED");

        if (msg.sender != _cvgControlTower.cvgUtilities()) {
            operator = msg.sender;
        }
        require(
            operator == _lockingPositionManager.ownerOf(tokenId) ||
                operator == lockingPositionDelegate.delegatedYsCvg(tokenId),
            "NOT_OWNED_OR_DELEGATEE"
        );

      ->uint256 cycleClaimed = tdeId * TDE_DURATION;

        require(_lockingPositionService.lockingPositions(tokenId).lastEndCycle >= cycleClaimed, "LOCK_OVER");

      ->require(_cvgControlTower.cvgCycle() >= cycleClaimed, "NOT_AVAILABLE");

      ->require(!rewardsClaimedForToken[tokenId][tdeId], "ALREADY_CLAIMED");

         and the totalSupply of ysCvg at the specified TDE. */
        uint256 share = (_lockingPositionService.balanceOfYsCvgAt(tokenId, cycleClaimed) * 10 ** 20) /
            _lockingPositionService.totalSupplyYsCvgHistories(cycleClaimed);

        require(share != 0, "NO_SHARES");

        _claimTokenRewards(tokenId, tdeId, share, receiver);

        rewardsClaimedForToken[tokenId][tdeId] = true;
    }
```
## Impact
The rewards are essentially locked in the contract if `claimRewards()` is executed before `depositMultipleToken()` in the same block, resulting in users being unable to claim their rightful rewards.

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/YsDistributor.sol#L101
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/YsDistributor.sol#L155
## Tool used

Manual Review

## Recommendation
```solidity
    function claimRewards(uint256 tokenId, uint256 tdeId, address receiver, address operator) external {
        ICvgControlTower _cvgControlTower = cvgControlTower;
        ILockingPositionService _lockingPositionService = lockingPositionService;
        ILockingPositionManager _lockingPositionManager = lockingPositionManager;
        require(_lockingPositionManager.unlockingTimestampPerToken(tokenId) < block.timestamp, "TOKEN_TIMELOCKED");

        if (msg.sender != _cvgControlTower.cvgUtilities()) {
            operator = msg.sender;
        }
        require(
            operator == _lockingPositionManager.ownerOf(tokenId) ||
                operator == lockingPositionDelegate.delegatedYsCvg(tokenId),
            "NOT_OWNED_OR_DELEGATEE"
        );

        uint256 cycleClaimed = tdeId * TDE_DURATION;

        require(_lockingPositionService.lockingPositions(tokenId).lastEndCycle >= cycleClaimed, "LOCK_OVER");

-       require(_cvgControlTower.cvgCycle() >= cycleClaimed, "NOT_AVAILABLE");
+       require(_cvgControlTower.cvgCycle() > cycleClaimed, "NOT_AVAILABLE");

        require(!rewardsClaimedForToken[tokenId][tdeId], "ALREADY_CLAIMED");

         and the totalSupply of ysCvg at the specified TDE. */
        uint256 share = (_lockingPositionService.balanceOfYsCvgAt(tokenId, cycleClaimed) * 10 ** 20) /
            _lockingPositionService.totalSupplyYsCvgHistories(cycleClaimed);

        require(share != 0, "NO_SHARES");

        _claimTokenRewards(tokenId, tdeId, share, receiver);

        rewardsClaimedForToken[tokenId][tdeId] = true;
    }
```



## Discussion

**0xR3vert**

Hello,

Thanks a lot for your attention.

That's a good catch. In reality, the claim is not designed to be used during a TDE cycle, but only afterwards. This allows a user to receive their rewards for the last TDE cycle. This oversight on our part is something we are grateful for you pointing out.

In conclusion, we must acknowledge your issue as valid.

Regards,
Convergence Team

# Issue M-6: GaugeController: `change_gauge_weight()` can be frontrun to improperly increase or decrease gauge weight 

Source: https://github.com/sherlock-audit/2023-11-convergence-judging/issues/122 

## Found by 
0xDetermination, ZanyBonzy, bughuntoor
## Summary
`change_gauge_weight()` can be easily frontrun to alter the expected result of the function call.
## Vulnerability Detail
Notice how weight is set by `change_gauge_weight()`:
```vyper
def _change_gauge_weight(addr: address, weight: uint256):
    ...
    self.points_weight[addr][next_time].bias = weight
    ...
```
Because the weight is directly set to the desired weight, the function can be frontrun to improperly increase the gauge weight. Let's see an example:
1. Admin submits a transaction to change a gauge's weight to some value X.
2. User who has an active vote for that gauge is contributing some nonzero value Y to the gauge's bias.
3. User frontruns the admin's transaction and votes zero for the gauge.
4. After waiting for `WEIGHT_VOTE_DELAY` (10 days), the user votes for that gauge again, increasing the gauge's bias by Y (approximately).
5. The admin expected the gauge's weight to be X, but it is instead X + Y.

In the above example, the user is effectively granted extra voting power. It's unlikely for the admin to detect that this exploit has occurred, since no suspicious actions were taken (voting is a normal interaction).
## Impact
Too many or too little rewards may be distributed. In the case of too many rewards distributed, the protocol loses funds.
## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Locking/GaugeController.vy#L578

https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Locking/GaugeController.vy#L16
## Tool used

Manual Review

## Recommendation
Consider modifying the function to increase/decrease the weight as desired instead of directly setting it:
```diff
-def _change_gauge_weight(addr: address, weight: uint256):
+def _change_gauge_weight(addr: address, weight: int256):
    ...
-    self.points_weight[addr][next_time].bias = weight
+    self.points_weight[addr][next_time].bias += weight
    ...
```



## Discussion

**0xR3vert**

Hello,

Thanks a lot for your attention.

Thank you for your insightful observation. Upon thorough examination, we acknowledge that such an occurrence could indeed jeopardize the protocol. We are currently exploring multiple solutions to address this issue.
We are considering removing the function change_gauge_weight entirely and not distributing CVG inflation on killed gauges, similar to how Curve Protocol handles their gauges.

Therefore, in conclusion, we must consider your issue as valid.

Regards,
Convergence Team

# Issue M-7: LockPositionService::increaseLockTime Incorrect Calculation Extends Lock Duration Beyond Intended Period 

Source: https://github.com/sherlock-audit/2023-11-convergence-judging/issues/136 

## Found by 
cergyk, jah, rvierdiiev
## Summary
`LockPositionService::increaseLockTime` uses `block.timestamp` for locking tokens, resulting in potential over-extension of the lock period. Specifically, if a user locks tokens near the end of a cycle, the lock duration might extend an additional week more than intended. For instance, locking for one cycle at the end of cycle N could result in an unlock time at the end of cycle N+2, instead of at the start of cycle N+2.

This means that all the while specifying that their $CVG should be locked for the next cycle, the $CVG stays locked for two cycles.

## Vulnerability Detail
The function `increaseLockTime` inaccurately calculates the lock duration by using `block.timestamp`, thus not aligned to the starts of cycles. This discrepancy leads to a longer-than-expected lock period, especially when a lock is initiated near the end of a cycle. This misalignment means that users are unintentionally extending their lock period and affecting their asset management strategies.

### Scenario:
- Alice decides to lock her tokens for one cycle near the end of cycle N.
- The lock duration calculation extends the lock to the end of cycle N+2, rather than starting the unlock process at the start of cycle N+2.
- Alice's tokens are locked for an additional week beyond her expectation.

## Impact

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L421

## Tool used
Users may have their $CVG locked for a week more than expected

## Recommendation
Align the locking mechanism to multiples of a week and use `(block.timestamp % WEEK) + lockDuration` for the lock time calculation. This adjustment ensures that the lock duration is consistent with user expectations and cycle durations.

# Issue M-8: When cvgCycle is incremented shortly after or at thursday midnight of TDE there is a possibility to mint big amount of veCVG and burn them shortly after 

Source: https://github.com/sherlock-audit/2023-11-convergence-judging/issues/158 

## Found by 
vvv
## Summary
When cvgCycle is incremented shortly after or right at thursday midnight of TDE there is a possibility to mint big amount of veCVG and burn them shortly after. Unlock time of veCVG is alsways set as the next thursday midnight of mint. When we mint veCVG in LockingPositionService.mintPosition with durationAdd == 0 shortly before the thursday midnight we can generate a big amount of veCVG and burn them when cvgCycle in incremented. 

## Vulnerability Detail

The unlock time of minted veCVG is always set as the next thursday midnight in `create_lock` in `veCVG.py`

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/veCVG.vy#L328

We are able to call `LockingPositionService.mintPosition` function with `lockDuration == 0` if the current `cvgCycle % TDE_DURATION == 0`, although we won't mint any ysCVG or mgCVG we will be able to mint `amount` of veCVG with such parameters. 

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L252

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L270-L273

If `cvgCycle` is incremented shortly after or right at the thursday midnight we will be able to burn previously minted veCVG right at the moment of the increment. At the midnight the `unlock_time` of veCVG will be valid for burn and after the increment the current cycle will be more than `endLockCycle`, thus allowing to `burnPosition`.

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/veCVG.vy#L404

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L518


## Impact

With such possibility we could perform a flash loan attack to generate a big amounts of veCVG in a short time. Having a big amount of veCVG could be used to maliciously affect protocol voting.

## Code Snippet

```solidity
mintPosition(0, <big amount of CVG from a flashloan>, 100, msg.sender, true) // shortly before the thursday midnight of TDE
<doing something malicious with the obtrained veCVG>
burnPosition(tokenId) // right after the increment of cvgCycle
```

## Tool used

Manual Review

## Recommendation

To disallow minting of veCVG without appopriate locking time we could add checks for `lastEndCycle > actualCycle` as in `LockingPositionService.increaseLockAmount` in other locking methods: `increaseLockTime` and `increaseLockTimeAndAmount`. Similary we should add check for `lockDuration > 0` in `LockingPositionService.mintPosition`.



## Discussion

**shalbe-cvg**

Hello,

Thanks a lot for your attention.

After an in-depth review, we have to consider your issue as Confirmed.
The locking duration should never be set to 0 cycle. As this doesn't make sense, we will add a new check preventing malicious users to create or increase the lock time of a position with a duration of 0.

Regards,
Convergence Team

# Issue M-9: cvgControlTower and veCVG lock timing will be different and lead to yield loss scenarios 

Source: https://github.com/sherlock-audit/2023-11-convergence-judging/issues/178 

## Found by 
0x52, 0xAlix2, cergyk
## Summary

When creating a locked CVG position, there are two more or less independent locks that are created. The first is in lockingPositionService and the other is in veCVG. LockingPositionService operates on cycles (which are not finite length) while veCVG always rounds down to the absolute nearest week. The disparity between these two accounting mechanism leads to conflicting scenario that the lock on LockingPositionService can be expired while the lock on veCVG isn't (and vice versa). Additionally tokens with expired locks on LockingPositionService cannot be extended. The result is that the token is expired but can't be withdrawn. The result of this is that the expired token must wait to be unstaked and then restaked, cause loss of user yield and voting power while the token is DOS'd.

## Vulnerability Detail

Cycles operate using block.timestamp when setting lastUpdateTime on the new cycle in [L345](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L345). It also requires that at least 7 days has passed since this update to roll the cycle forward in [L205](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L205). The result is that the cycle can never be exactly 7 days long and the start/end of the cycle will constantly fluctuate. 

Meanwhile when veCVG is calculating the unlock time it uses the week rounded down as shown in [L328](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/veCVG.vy#L328). 

We can demonstrate with an example:

Assume the first CVG cycle is started at block.timestamp == 1,000,000. This means our first cycle ends at 1,604,800. A user deposits for a single cycle at 1,400,000. A lock is created for cycle 2 which will unlock at 2,209,600. 

The lock on veCVG does not match this though. Instead it's calculation will yield:

    (1,400,000 + 2 * 604,800) / 604,800 = 4

    4 * 604,800 = 2,419,200

As seen these are mismatched and the token won't be withdrawable until much after it should be due to the check in veCVG [L404](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/veCVG.vy#L404).

This DOS will prevent the expired lock from being unstaked and restaked which causes loss of yield.

The opposite issue can also occur. For each cycle that is slightly longer than expected the veCVG lock will become further and further behind the cycle lock on lockingPositionService. This can also cause a dos and yield loss because it could prevent user from extending valid locks due to the checks in [L367](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/veCVG.vy#L367) of veCVG.

An example of this:

Assume a user locks for 96 weeks (58,060,800). Over the course of that year, it takes an average of 2 hours between the end of each cycle and when the cycle is rolled over. This effectively extends our cycle time from 604,800 to 612,000 (+7200). Now after 95 cycles, the user attempts to increase their lock duration. veCVG and lockingPositionService will now be completely out of sync:

After 95 cycles the current time would be:

    612,000 * 95 = 58,140,000

Whereas veCVG lock ended:

    612,000 * 96 = 58,060,800

According to veCVG the position was unlocked at 58,060,800 and therefore increasing the lock time will revert due to [L367](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/veCVG.vy#L367)

The result is another DOS that will cause the user loss of yield. During this time the user would also be excluded from taking place in any votes since their veCVG lock is expired.

## Impact

Unlock DOS that cause loss of yield to the user

## Code Snippet

[CvgRewards.sol#L341-L349](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L341-L349)

## Tool used

Manual Review

## Recommendation

I would recommend against using block.timestamp for CVG cycles, instead using an absolute measurement like veCVG uses.

# Issue M-10: SdtRewardReceiver#_withdrawRewards has incorrect slippage protection and withdraws can be sandwiched 

Source: https://github.com/sherlock-audit/2023-11-convergence-judging/issues/180 

## Found by 
0x52, 0xkaden, Bauer, CL001, FarmerRick, caventa, cducrest-brainbot, detectiveking, hash, lemonmon, r0ck3tz
## Summary

The _min_dy parameter of poolCvgSDT.exchange is set via the poolCvgSDT.get_dy method. The problem with this is that get_dy is a relative output that is executed at runtime. This means that no matter the state of the pool, this slippage check will never work.

## Vulnerability Detail

[SdtRewardReceiver.sol#L229-L236](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtRewardReceiver.sol#L229-L236)

            if (isMint) {
                /// @dev Mint cvgSdt 1:1 via CvgToke contract
                cvgSdt.mint(receiver, rewardAmount);
            } else {
                ICrvPoolPlain _poolCvgSDT = poolCvgSDT;
                /// @dev Only swap if the returned amount in CvgSdt is gretear than the amount rewarded in SDT
                _poolCvgSDT.exchange(0, 1, rewardAmount, _poolCvgSDT.get_dy(0, 1, rewardAmount), receiver);
            }

When swapping from SDT to cvgSDT, get_dy is used to set _min_dy inside exchange. The issue is that get_dy is the CURRENT amount that would be received when swapping as shown below:

    @view
    @external
    def get_dy(i: int128, j: int128, dx: uint256) -> uint256:
        """
        @notice Calculate the current output dy given input dx
        @dev Index values can be found via the `coins` public getter method
        @param i Index value for the coin to send
        @param j Index valie of the coin to recieve
        @param dx Amount of `i` being exchanged
        @return Amount of `j` predicted
        """
        rates: uint256[N_COINS] = self.rate_multipliers
        xp: uint256[N_COINS] = self._xp_mem(rates, self.balances)
    
        x: uint256 = xp[i] + (dx * rates[i] / PRECISION)
        y: uint256 = self.get_y(i, j, x, xp, 0, 0)
        dy: uint256 = xp[j] - y - 1
        fee: uint256 = self.fee * dy / FEE_DENOMINATOR
        return (dy - fee) * PRECISION / rates[j]

The return value is EXACTLY the result of a regular swap, which is where the problem is. There is no way that the exchange call can ever revert. Assume the user is swapping because the current exchange ratio is 1:1.5. Now assume their withdraw is sandwich attacked. The ratio is change to 1:0.5 which is much lower than expected. When get_dy is called it will simulate the swap and return a ratio of 1:0.5. This in turn doesn't protect the user at all and their swap will execute at the poor price.

## Impact

SDT rewards will be sandwiched and can lose the entire balance

## Code Snippet

[SdtRewardReceiver.sol#L213-L245](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtRewardReceiver.sol#L213-L245)

## Tool used

Manual Review

## Recommendation

Allow the user to set _min_dy directly so they can guarantee they get the amount they want



## Discussion

**shalbe-cvg**

Hello,

Thanks a lot for your attention.

After an in-depth review, we have to consider your issue as Confirmed.
Not only users can get sandwiched but in most cases this exchange directly on the pool level would rarely succeed as `get_dy` returns the exact amount the user could get. We will add a slippage that users will setup.

Regards,
Convergence Team

