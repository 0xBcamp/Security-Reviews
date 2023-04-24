# BCAMP CHARLIE SMART CONTRACT SECURITY REVIEW

## Overview

- This document details the findings of a security review performed by Team Sierra on Team Charlie's project.

## Project Summary

The VaultFactory Project aims to create a service that allows users to copy trades from reputed whales on different DEXs and CEX futures.

## Table of Content

- Interface
- Proxy Smart Contract
- Implementation Contract
- Factory Contract

## Interface IVaultFactory.sol

```solidity
function fireVaultEvent(
  address proxy,
  bytes32 name,
  address trader,
  bool inverseCopyTrade,
  uint16 copySizeBPS,
  address defaultCollateral
) external;

```

### The Vulnerability

- There is no comment on the contract

### Preventative Techniques

- Contract codes should always be commented out to aid readility of the Smart contract

## VaultProxy.sol

```solidity
contract VaultProxy is ERC1967Proxy, OwnableUpgradeable {
  constructor(address initialImpl) ERC1967Proxy(initialImpl, "") initializer {
    _upgradeTo(initialImpl);
    __Ownable_init();
  }
}

```

### The Vulnerability

- There is no comment on the contract

### Preventative Techniques

- Contract codes should always be commented out to aid readility of the Smart contract

## VaultImplementationV1.sol

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

### The Vulnerability

- There is no comment on the contract

- Possible underflow/overflow bug on function withDrawProfit()

### Preventative Techniques

- Contract codes should always be commented out to aid readility of the Smart contract

- Use of OpenZeppelin SafeMath Library

```solidity
function withdrawETH(address recepient) public onlyOwner {
  payable(vaultFactory).transfer(
    (address(this).balance * MANAGEMENT_FEE) / BASIS_POINTS_DIVISOR
  );
  payable(recepient).transfer(address(this).balance);
}

```

### The Vulnerability

- There is no comment on the contract

- No check for address zero

- Possible underflow/overflow bug on function withdrawETH()

- Possible Re-Entrancy attack with the use of \_to.transfer()

- Unchecked return value

### Preventative Techniques

- Contract codes should always be commented out to aid readility of the Smart contract

- Check for address zero using require(address(0), "Address zero")

- Use of OpenZeppelin SafeMath Library on arimetic calculation

- Use of (bool sent, bytes memory data) = \_to.call{value: msg.value}("") to check for Re-Entrancy

### VaultFactory.sol

```solidity
function withdrawETH(address recepient) public onlyGov {
  payable(recepient).transfer(address(this).balance);
}

```

### The Vulnerability

- There is no comment on the contract

- No check for address zero

- Possible Re-Entrancy attack with the use of \_to.transfer()

- Unchecked return value

### Preventative Techniques

- Contract codes should always be commented out to aid readility of the Smart contract

- Check for address zero using require(address(0), "Address zero")

- Use of (bool sent, bytes memory data) = \_to.call{value: msg.value}("") to check for Re-Entrancy
