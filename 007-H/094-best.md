Joyous Fuzzy Cobra

medium

# Lowering the gauge weight can disrupt accounting, potentially leading to both excessive fund distribution and a loss of funds.

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