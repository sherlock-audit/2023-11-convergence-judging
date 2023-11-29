Innocent Scarlet Quail

medium

# SdtRewardReceiver#_withdrawRewards has incorrect slippage protection and withdraws can be sandwiched

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