Big Cloth Gorilla

high

# ```gaugeController``` can add twice the same ```gaugeAddress``` to the ```gauges``` array, leading to faulty behavior of the ```removeGauge``` function.

## Summary

In the [```addGauge```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L129-L133) function, no check is made on the presence of ```gaugeAddress``` in the ```gauges``` array.

```gaugeController```'a attempt to remove the duplicated ```gaugeAddress``` would then lead to removing the wrong ```gauge``` from the array. 

## Vulnerability Detail

Let's say 2 ```gauges``` have been added and ```gaugeAddress3``` is not one of them.
Now let the ```gaugeController``` call ```addGauge``` with the parameter ```gaugeAddress3```.
We will then have ```gaugesId[gaugeAddress3] == 2``` and ```gauges == [gaugeAddress1, gaugeAddress2, gaugeAddress3]```

Now if ```gaugeController``` calls the ```addGauge``` function again with the same ```gaugeAddress3``` parameter, we will then have ```gaugesId[gaugeAddress3] == 3``` and ```gauges == [gaugeAddress1, gaugeAddress2, gaugeAddress3, gaugeAddress3]```.

We then want to remove ```gaugeAddress3``` from the ```gauges``` array by calling the [```removeGauge```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L139-L154) with parameter ```gaugeAddress3```.

We will then have ```idGaugeToRemove = gaugesId[gaugeAddress3]``` ie ```idGaugeToRemove == 3```. ```lastGauge = gauges[gauges.length - 1]``` i.e. ```lastGauge == gaugeAddress3```.

Next line sets ```gaugesId[lastGauge]``` to ```idGaugeToRemove``` i.e. ```gaugesId[gaugeAddress3] == 3```, then we set ```gaugesId[gaugeAddress3]``` to ```0```.

Finally, we set ```gauges[idGaugeToRemove] = lastGauge``` i.e. we set ```gauges[3]``` to ```gaugeAddress3``` and pop the last element of ```gauges```, meaning our ```gauges``` array now looks like ```[gaugeAddress1, gaugeAddress2, gaugeAddress3]```.

However we now have ``` gaugesId[gaugeAddress3] == 0``` because of the ```gaugesId[gaugeAddress] = 0;``` line.

Let's now call [```removeGauge```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L139-L154) again with the same ```gaugeAddress3``` parameter.

We then have similarly ```idGaugeToRemove == 0```, ```lastGauge == gaugeAddress3```.
We set ```gaugesId[gaugeAddress3]``` to ```0``` (it is already the case) and ```gaugesId[gaugeAddress3]``` to ```0``` (lines 145 and 147 do the same thing then).
Finally we set ```gauges[0]``` to ```gaugeAddress3``` and we ```pop``` the last element, meaning our ```gauges``` array now looks like ```[gaugeAddress3, gaugeAddress2]``` meaning we removed the wrong gauge address!


## Impact

We believe this vulnerability should be marked as HIGH as this leads to the ```removeGauge``` function to remove the wrong addresses as shown above, altering the list of gauges which is central to the good functioning of the contract.

## Code Snippet

Below are the definitions of the [```addGauge```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L129-L133) and [```removeGauge```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L139-L154) functions

```solidity
function addGauge(address gaugeAddress) external {
        require(address(cvgControlTower.gaugeController()) == msg.sender, "NOT_GAUGE_CONTROLLER");
        gauges.push(gaugeAddress);
        gaugesId[gaugeAddress] = gauges.length - 1;
     }
```
    
and 

```solidity
function removeGauge(address gaugeAddress) external {
        require(address(cvgControlTower.gaugeController()) == msg.sender, "NOT_GAUGE_CONTROLLER");
        uint256 idGaugeToRemove = gaugesId[gaugeAddress];
        address lastGauge = gauges[gauges.length - 1];

        /// @dev replace id of last gauge by deleted one
        gaugesId[lastGauge] = idGaugeToRemove;
        /// @dev Set ID of gauge as 0
        gaugesId[gaugeAddress] = 0;

        /// @dev moove last gauge address to the id of the deleted one
        gauges[idGaugeToRemove] = lastGauge;

        /// @dev remove last array element
        gauges.pop();
    }
```

## Tool used

Manual Review / Visual Studio

## Recommendation

The fix is relatively straightforward and we just need to check for the presence of ```gaugeAddress``` in the ```gaugesId``` mapping before adding it (note that we use ```gaugesId``` and not the ```gauges``` array directly as checking the presence of an element in the keys of a mapping is easier than checking the presence of an element in an array, as ```gaugesId[gaugeAddress]``` will simply return ```0``` if the element does not exist).

The [```addGauge```](https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L129-L133) function would then look like the below

```solidity
function addGauge(address gaugeAddress) external {
        require(address(cvgControlTower.gaugeController()) == msg.sender, "NOT_GAUGE_CONTROLLER");
        require(gaugesId[gaugeAddress] == 0, "Address already added");
        gauges.push(gaugeAddress);
        gaugesId[gaugeAddress] = gauges.length - 1;
 }
```