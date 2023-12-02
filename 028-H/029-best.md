Spare Leather Puma

high

# Reducing a gauge's weight might actually give it a significant advantage

## Summary
Reducing a gauge might give it an unfair advantage in comparison with other gauges

## Vulnerability Detail
Based on their voting escrows, users can vote for gauges within `GaugeController`. The gauges are allocated the corresponding `bias` and `slope`.
Currently, there is an admin-privileged function `change_gauge_weight` which can be used to change a gauge's weight (or more specifically, its `bias`) 
```solidity
@internal
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
It can be expected that in some circumstances it can be used to decrease a gauge's weight. However, it might end up as an actual boost under some circumstances.
Let's look into what happens after a gauge's weight is reduced.
Within `_get_weight` which calculates the gauge's weight, we will reach a state where `pt.bias < d_bias` (because the bias has been arbitrary decreased by an admin). 
```solidity
@internal
def _get_weight(gauge_addr: address) -> uint256:
    """
    @notice Fill historic gauge weights week-over-week for missed checkins
            and return the total for the future week.
    @param gauge_addr Address of the gauge
    @return Gauge weight
    """
    t: uint256 = self.time_weight[gauge_addr]
    if t > 0:
        pt: Point = self.points_weight[gauge_addr][t]
        for i in range(500):
            if t > block.timestamp:
                break
            t += WEEK
            d_bias: uint256 = pt.slope * WEEK
            if pt.bias > d_bias:
                pt.bias -= d_bias
                d_slope: uint256 = self.changes_weight[gauge_addr][t]
                pt.slope -= d_slope
            else:
                pt.bias = 0
                pt.slope = 0
            self.points_weight[gauge_addr][t] = pt
            if t > block.timestamp:
                self.time_weight[gauge_addr] = t
        return pt.bias
    else:
        return 0
```
This would result in entering the `else` statement, which would then set both `pt.bias` and `pt.slope` to 0. Note that even after this happens, we still have timestamps `t` for which `self.changes_weight[gauge_addr][t]` holds a non-zero value, and the gauge's slope is intended to decrease then.
Now, if users decide to vote for the gauge, the gauge will have normal weight until timestamp `t`. Then, its slope will artificially be decreased. Since the slope is decreased, this would mean that the gauge's bias will decrease at a slower rate, actually giving the gauge  more weight over time: 

Let's put it into an example: 
1. Gauge has 5,000 weight which should decrease over 4 weeks (bias = 5000, slope = 1250 / WEEK)
2. Admins decide to reduce the gauge's weight to 0. (reduce by 5,000).
3. A week goes by. The next call to `_get_weight` would set both the gauge's bias and slope to 0. 
4. User votes 50,000 weight which should decrease over 40 weeks. (bias = 50000, slope = 1250 / WEEK) (note: the 4 weeks since the original user's vote have still not yet passed).
5. After 3 weeks the call to `_get_weight` reduces the slope by the initial voter's slope (1250 / WEEK), therefore making the current slope = 0. The bias is now 46,250 (50,000 - 3 * 1250)
6. Now for the next 47 weeks, the gauge's weight is actually not decaying. The slope is 0 and the gauge's weight remains the same. 

If we look at the gauge 1 week before the 2nd voter's lock expires, the gauge will have weight of 46,250, because of the admins 'reducing' the gauges weight. If they hadn't 'reduced' it, the gauge's weight would be 1,250.


## Impact
Reducing a gauge's weight results in actually giving it extra weight.

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/GaugeController.vy#L568

## Tool used

Manual Review

## Recommendation
Fix is non-trivial. Would like to work with the team on coming up with a fix