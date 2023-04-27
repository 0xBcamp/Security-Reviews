# BCAMP Team CHARLIE SMART CONTRACT SECURITY REVIEW

## Overview

- This document details the findings of a security review performed by Team Sierra on Team Charlie's project.

## Project Summary

The VaultFactory Project aims to create a service that allows users to copy trades from reputed whales on different DEXs and CEX futures.

## Vulnerability Summary

| Number of Findings |     Findings      |                            Findings Summary                            |
| :----------------- | :---------------: | :--------------------------------------------------------------------: |
| 0                  |   **Critical**    | Critical risks are those that impact the safe functioning ofa platform |
|                    |                   |   and must be addressed before launch. Usersshould not invest in any   |
|                    |                   |                project with outstanding criticalrisks.                 |
|                    |                   |                                                                        |
| 0                  |     **Major**     | Major risks can include centralization issues and logicalerrors. Under |
|                    |                   |   specific circumstances, these major riskscan lead to loss of funds   |
|                    |                   |                     and/or control of the project.                     |
|                    |                   |                                                                        |
| 0                  |    **Medium**     | Medium risks may not pose a direct risk to users’ funds, but they can  |
|                    |                   |             affect the overall functioning of a platform.              |
|                    |                   |                                                                        |
| 2                  |     **Minor**     |      Minor risks can be any of the above, but on a smaller scale.      |
|                    |                   | They generally do not compromise the overallintegrity of the project,  |
|                    |                   |          but they may be less efficient thanother solutions.           |
|                    |                   |                                                                        |
| 3                  | **Informational** | Informational errors are often recommendations to improve the style of |
|                    |                   | the code or certain operations to fallwithin industry best practices.  |
|                    |                   |     They usually do not affectthe overall functioning of the code.     |

## Table of Content

- Findings
- Code Base
- Scope
- Approach and Method

## Findings

- 1.0 Underflow/overflow
- 1.2 Usage Of transfer() For Sending ETH
- 1.3 Unused return value
- 1.4 unchecked address(0)

## Code Base

- [https://github.com/0xBcamp/Charlie]

## Security Review Scope

- Team Charlie Smart Contract

## Approach and Method

This report has been prepared for Bcamp Team Charlie - BcTC to discover issues and vulnerabilities in the source code of the Team Charlie project as well as any contract dependencies that were not part of an officially recognized library. A good examination has been performed, utilizing Manual Review.

## UnderFlow/OverFlow Bug- Minor

### Implementation Contract: VaultImplementationV1.sol

```solidity
function withDrawProfit(
  address token,
  address recepient,
  uint256 amount
) public onlyOwner {
  if (token != address(0)) {
    IERC20(token).transfer(
      recepient,
      (amount * (BASIS_POINTS_DIVISOR - MANAGEMENT_FEE)) / BASIS_POINTS_DIVISOR
    );
    IERC20(token).transfer(
      vaultFactory,
      (amount * MANAGEMENT_FEE) / BASIS_POINTS_DIVISOR
    );
  }
}

```

```solidity
function withdrawETH(address recepient) public onlyOwner {
  payable(vaultFactory).transfer(
    (address(this).balance * MANAGEMENT_FEE) / BASIS_POINTS_DIVISOR

```

### The Vulnerability

- Possible underflow/overflow bug on function withDrawProfit() when performing arithematic operation on (amount \* (BASIS_POINTS_DIVISOR - MANAGEMENT_FEE)) / BASIS_POINTS_DIVISOR.

- Possible underflow/overflow bug on function withdrawETH() when performing arithematic operation on (address(this).balance \* MANAGEMENT_FEE) / BASIS_POINTS_DIVISOR

### Preventative Techniques

- Use of OpenZeppelin SafeMath Library provides a standard library for performing arithematic operation on uint

## Usage Of transfer() For Sending ETH - Minor

### Implementation Contract: VaultImplementationV1.sol

```solidity
function withdrawETH(address recepient) public onlyOwner {
  payable(vaultFactory).transfer(
    (address(this).balance * MANAGEMENT_FEE) / BASIS_POINTS_DIVISOR
  );
  payable(recepient).transfer(address(this).balance);
}

```

### Factory Contract: VaultFactory.sol

```solidity
function withdrawETH(address recepient) public onlyGov {
  payable(recepient).transfer(address(this).balance);
}

```

### The Vulnerability

- Possible Re-Entrancy attack with the use of \_to.transfer()

### Preventative Techniques

- Use of (bool sent, bytes memory data) = \_to.call{value: msg.value}("") to check for Re-Entrancy or using the **Address.sendValue()** function from OpenZeppelin.Since Address.sendValue() may allow reentrancy, we also recommend guarding against reentrancy attacks by utilizingthe Checks-Effects-Interactions Pattern or applying OpenZeppelin ReentrancyGuard.

## Unchecked/Unused Return Value - Informational

```solidity
returns (bool success) {}

```

### The Vulnerability

- Unchecked return value

### Preventative Techniques

- we recommend assigning return variables or writing explicit return statements to avoid implicitly returning default values.Also, if there are local variables duplicating named return variables, we recommend removing the local variables and use the return variables instead.

## Unchecked address input Value against address(0x0) - Informational

```solidity
require(address != address(0x0), "Invalid Address")

```

### The Vulnerability

- Unchecked input address

### Preventative Techniques

- we recommend checking for address zero (address(0)) to prevent tokens being transferred to invalid address.

## commenting/coding Style - Informational

### The Vulnerability

- There is no comment throughout the smart contract

### Preventative Techniques

- Contract codes should always be commented out to aid readility of the Smart contract
