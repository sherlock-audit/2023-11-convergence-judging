Proper Lemonade Sealion

high

# Killing a gague can lead to bricking of the protocol

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