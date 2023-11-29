Low Carrot Jellyfish

medium

# deposit in bond contract in CvgUtility has no possibility to express slippage control/&expiry

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