Low Carrot Jellyfish

medium

# areLimitsVerified in getDataForVerification uses a wrong inequality leading to wrong representation of return

## Summary
areLimitsVerified in getDataForVerification uses a wrong inequality leading to wrong representation of return

## Vulnerability Detail
`limitPirce` is fetched from `_getDataForVerification` which is then conpared against oracle minPrice and maxPrice. However in `getDataForVerification` the comparison is done wrongly.

```solidity
    function getDataForVerification(
        address erc20Address
    )
....
areLimitsVerified = limitPrice > oracleParams.minPrice && limitPrice > oracleParams.maxPrice;
```

This is a mistake since the comparison is  done differently in `_getAndVerificationOracle`

```solidity
    function _getAndVerificationOracle(IOracleStruct.OracleParams memory oracleParams) internal view returns (uint256) {
        (
            uint256 poolOraclePrice,
            uint256 limitPrice,
            bool isNotTooLow,
            bool isNoTooHigh,
            bool isEthVerified,
            bool isNotStale
        ) = _getDataForVerification(oracleParams);

        require(isNotTooLow, "LIMIT_TOO_LOW");
        require(isNoTooHigh, "LIMIT_TOO_HIGH");
        require(isEthVerified, "ETH_NOT_VERIFIED");
        require(isNotStale, "STALE_PRICE");
        require(_verifyStableInPath(oracleParams.stablesToCheck), "STABLES_NOT_VERIFIED");
        require(limitPrice > oracleParams.minPrice && limitPrice < oracleParams.maxPrice, "PRICE_OUT_OF_LIMITS");
        return poolOraclePrice;
    }
```

## Impact
areLimitsVerified in getDataForVerification uses a wrong inequality leading to wrong representation of return value

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Oracles/CvgOracle.sol#L96
## Tool used

Manual Review

## Recommendation
```solidity
-       areLimitsVerified = limitPrice > oracleParams.minPrice && limitPrice > oracleParams.maxPrice;
+      areLimitsVerified = limitPrice > oracleParams.minPrice && limitPrice < oracleParams.maxPrice;
```