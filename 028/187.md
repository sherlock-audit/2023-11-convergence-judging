Keen Mulberry Hawk

medium

# Unable to add a killed gauge back again.

## Summary
If admin kills the gauge for a staking contract, it is not possible to add it back again.

## Vulnerability Detail
In `add_gauge`, the gauge type of `addr` is first compared with 0 to avoid adding same gauge twice (L405). The `addr`'s gauge type is then set to a non-zero value (L412).
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/GaugeController.vy#L396-L412
```vyper
396: def add_gauge(addr: address, gauge_type: int128, weight: uint256):
397:     """
398:     @notice Add gauge `addr` of type `gauge_type` with weight `weight`.
399:     @param addr Gauge address
400:     @param gauge_type Gauge type
401:     @param weight Gauge weight
402:     """
403:     assert msg.sender == self.admin
404:     assert (gauge_type >= 0) and (gauge_type < self.n_gauge_types)
405:->   assert self.gauge_types_[addr] == 0 , "GAUGE_ALREADY_ADDED" # dev: cannot add the same gauge twice
406:     assert (self.control_tower).isStakingContract(addr), "NOT_A_STAKING_CONTRACT"
407: 
408:     n: int128 = self.n_gauges
409:     self.n_gauges = n + 1
410:     self.gauges[n] = addr
411：
412:->   self.gauge_types_[addr] = gauge_type + 1
```

But the gauge type is not cleared while killing a gauge. The result is if admin tries to add the killed gauge back again, the gauge type check in `add_gauge` will fail and revert with `GAUGE_ALREADY_ADDED`.
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/GaugeController.vy#L603-L611
```vyper
603: def kill_gauge(addr: address):
604:     """
605:     @notice Change weight of gauge `addr` to `weight`.
606:     @param addr `GaugeController` contract address
607:     """
608:     assert msg.sender == self.admin, "NOT_ADMIN"
609:     self._change_gauge_weight(addr, 0)
610:     self.killed_gauges[addr] = True
611:     self.control_tower.cvgRewards().removeGauge(addr)
```

POC: Let's change the following test case and kill the last gauge and then add it back again.
```TypeScript
 85:    it("Success : Adding gauges in the Control tower", async () => {
 86:        for (let index = 0; index < gauges.length - 3; index++) {
 87:            const gauge = gauges[index];
 88:            await cvgControlTower.connect(treasuryDao).toggleStakingContract(gauge);
 89:            await gaugeController.connect(treasuryDao).add_gauge(gauge, 0, 0);
 90:            await gaugeController.connect(treasuryDao).toggle_vote_pause(gauge);
 91:            expect(await cvgRewards.gauges(index)).to.be.eq(await gauge.getAddress());
 92:            expect(await cvgRewards.gaugesId(gauge)).to.be.eq(index);
 93:        }
 94:
 95:        for (let index = gauges.length - 3; index < gauges.length; index++) {
 96:            const gauge = gauges[index];
 97:            await cvgControlTower.connect(treasuryDao).toggleStakingContract(gauge);
 98:            await gaugeController.connect(treasuryDao).add_gauge(gauge, 1, 0);
 99:            await gaugeController.connect(treasuryDao).toggle_vote_pause(gauge);
100:            expect(await gaugeController.gauges(index)).to.be.eq(await gauge.getAddress());
101:            expect(await cvgRewards.gaugesId(gauge)).to.be.eq(index);
102:        }
103:
104:+       // First kill the last gauge
105:+       await gaugeController.connect(treasuryDao).kill_gauge(gauges[gauges.length - 1]);
106:+
107:+       // Then add the last gauge back again => will revert
108:+       await gaugeController.connect(treasuryDao).add_gauge(gauges[gauges.length - 1], 0, 0);
109:    });
```
The log of the test case is as follows. We can see that the `add_gauge` revert with `GAUGE_ALREADY_ADDED`.
```TypeScript
  1) CvgRewards / write staking rewards full
       Success : Adding gauges in the Control tower:
     Error: VM Exception while processing transaction: reverted with reason string 'GAUGE_ALREADY_ADDED'
    at <UnrecognizedContract>.<unknown> (0x4458acb1185ad869f982d51b5b0b87e23767a3a9)
```

## Impact
Unable to add a killed gauge back again.

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/GaugeController.vy#L396-L412

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/GaugeController.vy#L603-L611

## Tool used

Manual Review

## Recommendation
1. Clear gauge type of the input `addr` in `kill_gauge`.
```vyper
@@ -608,6 +610,7 @@ def kill_gauge(addr: address):
     assert msg.sender == self.admin, "NOT_ADMIN"
     self._change_gauge_weight(addr, 0)
     self.killed_gauges[addr] = True
+    self.gauge_types_[addr] = 0
     self.control_tower.cvgRewards().removeGauge(addr)
```
2. The killed gauge is recorded in `kill_gauge` (L610), we need to reset it in `add_gauge`.
```vyper
@@ -405,10 +405,13 @@ def add_gauge(addr: address, gauge_type: int128, weight: uint256):
     assert self.gauge_types_[addr] == 0 , "GAUGE_ALREADY_ADDED" # dev: cannot add the same gauge twice
     assert (self.control_tower).isStakingContract(addr), "NOT_A_STAKING_CONTRACT"
 
-    n: int128 = self.n_gauges
-    self.n_gauges = n + 1
-    self.gauges[n] = addr
-
+    if self.killed_gauges[addr] == True:
+        self.killed_gauges[addr] = False
+    else:
+        n: int128 = self.n_gauges
+        self.n_gauges = n + 1
+        self.gauges[n] = addr
+        
     self.gauge_types_[addr] = gauge_type + 1
     next_time: uint256 = (block.timestamp + WEEK) / WEEK * WEEK
```
