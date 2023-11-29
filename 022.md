Polite Berry Chipmunk

high

# Potential loss of SDT token inside CvgSDT

## Summary
Potential loss of SDT token inside CvgSDT

## Vulnerability Detail
Upon `mint`, `CvgSDT` transfers `SDT` from `msg.sender` to `veSdtMultisig`

If `veSdtMultisig` is the zero address, this will lead to SDT being sent to the zero address. 

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Token/CvgSDT.sol#L40

## Impact

Loss of SDT

## Code Snippet

## Tool used

Manual Review

## Recommendation
Check `veSdtMultisig` is not zero address

```solidity

    function mint(address account, uint256 amount) external {
        address veSdtMultisig = cvgControlTower.veSdtMultisig();
        require(veSdtMultisig != address(0));
        sdt.transferFrom(msg.sender, veSdtMultisig, amount);
        _mint(account, amount);
    }

```
