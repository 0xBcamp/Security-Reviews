# BCAMP Team ComicCrafter SMART CONTRACT SECURITY REVIEW

## Overview

- This document details the findings of a security review performed by Team tx-metadata on Team ComicCrafter's project.

## About  Project

- ComicCrafter is a commic book digitalisation as NFT Basic game implementation. Allow game creators digitalise for gaming according to a set of predefined game elements based on the commic book.

## Codebase

- review commit hash - [0df2c80e09e5dc4a383480083057267ac26be269](https://github.com/0xBcamp/ComicCrafter_June2023/tree/0df2c80e09e5dc4a383480083057267ac26be269/comic-crafter)

## Vulnerability Level

| Level      | Description                                                                                           |
| ---------- | ----------------------------------------------------------------------------------------------------- |
| Critical   | Critical severity vulnerabilities will have a significant impact on the security of the DeFi project. |
| High       | High severity vulnerabilities will affect the normal operation of the DeFi project.                   |
| Medium     | Medium severity vulnerability will affect the operation of the DeFi project.                          |
| Low        | Low severity vulnerabilities may affect the operation of the DeFi project in certain scenarios.       |
| Weakness   | There are safety risks theoretically, but it is extremely difficult to reproduce in engineering.      |
| Suggestion | There are better practices for coding or architecture.                                                | 

## Findings

- String Duplication
- ERC20/ERC721 allowance frontrun attack
- Dependencies folder missing
- Lack of administration
- Commenting / Coding style

### String Duplication

- Level - High
- `_tokenToUri` is to record the uri which is used. However, mapping type in solidity is a hash map implementation. The key will be `keccak256(string)` and offchain service might trim spaces after fetch the uri in contract. Therefore, uris can be duplicated.
- Uri will be duplicated by adding extra spaces behind the string. i.e.
	- `http://abc.efg/`
	- `http://abc.efg/________` (extra space)

```solidity
// mapping(string => uint256) private _tokenToUri;

function mint(address _to,  string memory uri) public {
	if(_tokenToUri[uri] != 0){
		revert DoulicatedURI();
	}
	...
}
```

### ERC20/ERC721 allowance frontrun attack

- Level - Weakness
- Allowance frontrun attack is a specification attack.
- attack scenario
	1. Alice had approved some balance of token to Bob before.
	2. Now, Alice broadcast a transaction to decrease/revoke the approval.
	3. Bob listen to the mempool find out  what Alice wants to do and broadcast a transaction to frontrun and move the token first.
- more details [here](https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA\_jp-RLM/edit#heading=h.m9fhqynw2xvt)

### Dependencies folder missing

- Level - Suggestion
- Foundry uses git submodule to manage the dependencies and repo should have a `lib` folder for dependencies usually. Missing to commit this folder will let auditor unable to know which version of Openzeppelin contracts is used.

### Lack of administration

- Level - Suggestion
- Some important functions like `mint` probably need administration. For example, only game creators can execute some operations to adjust the system in order to release new maps, items and etc.

### Commenting / Coding style

- Level - Suggestion
- Some of the code lacks indentation or adheres to the linter's formatting.

**recommendation**

- Run `forge fmt` before commit and add in CI/CD config.
