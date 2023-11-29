Low Carrot Jellyfish

medium

# isNotTooLow and isNotTooHigh return from _getAndVerificationOracle is mixed

## Summary
isNotTooLow and isNotTooHigh return from _getAndVerificationOracle is mixed 

## Vulnerability Detail
In the consumption of return value from `_getDataForVerification`, the first bool is interpreted as `NotTooLow` and the second bool is interpreted as `NotTooHigh` 
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
```

However in both `_getDataForVerifyChainLink` and `_getDataForVerifyCurve`, the first bool checks `poolPriceUsd + delta > limitPrice` and the second bool checks `poolPriceUsd - delta < limitPrice`, verifying the upperBound & lowerBound is not violated respectively.


```solidity
    function _getDataForVerification(
        IOracleStruct.OracleParams memory oracleParams
    ) internal view returns (uint256, uint256, bool, bool, bool, bool) {
        if (uint256(oracleParams.poolType) < 3) {
            return _getDataForVerifyChainLink(oracleParams);
        }
        return _getDataForVerifyCurve(oracleParams);
    }

    function _getDataForVerifyChainLink(
        IOracleStruct.OracleParams memory oracleParams
    ) internal view returns (uint256, uint256, bool, bool, bool, bool) {
        /// @dev Fetch price through CvgOracle
        (, uint256 poolPriceUsd, bool isEthVerified) = _getPriceOracle(oracleParams);

        uint256 delta = (oracleParams.deltaLimitOracle * poolPriceUsd) / 10_000;

        (uint256 limitPrice, uint256 lastUpdateDate) = _getAggregatorPrice(oracleParams.aggregatorOracle);
        bool isNotStale = lastUpdateDate + oracleParams.maxLastUpdate > block.timestamp;

        return (
            poolPriceUsd,
            limitPrice,
            poolPriceUsd + delta > limitPrice,
            poolPriceUsd - delta < limitPrice,
            isEthVerified,
            isNotStale
        );
    }

    function _getDataForVerifyCurve(
        IOracleStruct.OracleParams memory oracleParams
    ) internal view returns (uint256, uint256, bool, bool, bool, bool) {
        /// @dev Get price
        (uint256 poolOraclePriceNotTransformed, uint256 poolPriceUsd, bool isEthVerified) = _getPriceOracle(
            oracleParams
        );
        bool isNotStale = ICrvPool(oracleParams.poolAddress).last_prices_timestamp() + oracleParams.maxLastUpdate >
            block.timestamp;

        uint256 lastPrice = _getCurveLastPrice(
            oracleParams.poolAddress,
            oracleParams.poolType,
            oracleParams.isReversed,
            oracleParams.twapOrK
        );

        uint256 delta = (oracleParams.deltaLimitOracle * poolOraclePriceNotTransformed) / 10_000;

        return (
            poolPriceUsd,
            lastPrice,
            poolOraclePriceNotTransformed + delta > lastPrice,
            poolOraclePriceNotTransformed - delta < lastPrice,
            isEthVerified,
            isNotStale
        );
    }
```
## Impact
isNotTooLow and isNotTooHigh are consumed wrongly.

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Oracles/CvgOracle.sol#L82-L83
## Tool used

Manual Review

## Recommendation
```solidity
-            bool isNotTooLow,
-            bool isNoTooHigh,
+            bool isNotTooHigh,
+            bool isNoTooLow,
            bool isEthVerified,
            bool isNotStale
        ) = _getDataForVerification(oracleParams);
```
