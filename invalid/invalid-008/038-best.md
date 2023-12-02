Perfect Coffee Sawfish

high

# User can mint cvgSdt without paying sdt in exchange

## Summary
The mint function under CvgSDT.sol is making an external call to sdt.transferfrom contract to transfer sdt from the sender to multisig, however the transferfrom function is improperly implemented because it does not check for the return value. sdt contract transferfrom return a boolean under success or failure, so if an user call the mint function under CvgSDT and something unexpected happens instead of reverting the sdt.transferFrom function will just throw false. Failling to check the retrun value  allow malicious users to potentially mint CvgSDT for free

## Vulnerability Detail
There is an unchecked return value from external copntract call, and because of the monetary value protocol will lose of malicious users exploit this vulnerability i consider this to be a high ( users will be getting free CvgSDT).
The corresponding sdt transferfrom function can be found here: [sdt_token](https://etherscan.io/token/0x73968b9a57c6e53d41345fd57a6e6ae27d6cdb2f#code)

## Impact
CvgSDT contract will mint token to users addresses from free

## Code Snippet
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Token/CvgSDT.sol#L40

## Tool used
VsCode
Manual Review

## Recommendation
Check the return value of sdt.transferfrom before minting CvgSDT to caller address