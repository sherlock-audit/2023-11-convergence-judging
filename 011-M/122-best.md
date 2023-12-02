Soft Fern Donkey

high

# GaugeController: `change_gauge_weight()` can be frontrun to improperly increase or decrease gauge weight

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
