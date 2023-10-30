# Global Passport NFT Security Review

## Overview

The protocol issues passports in the form of NFTs to users. whenever the user visits a country the attestation is made by the user themself in the form of a stamp and stored in the blockchain.

## [H-1] Should get an attestation from the valid authority before using the passport

### Vulnerability details

If the passport NFT is meant to remove the need for a physical passport then the user-minted NFT should get an attestation from a passport authority of the holder’s country. Unless this step is added the passport will not get validated by the country the user is visiting.

### Impact

Without the verified attestation the passport is of no use.

### Mitigation and addition

To achieve this we can add a function to request attestation from the specific authority onchain, for example, if a user is from India he/she can request attestation on the passport from the Indian passport authority. This will need to onboard an Indian passport authority onchain.

## [H-2] `createStamp()` will fail if called by anyone except the passport holder.

### Code Line

https://github.com/0xBcamp/Sept23_Passport_NFT/blob/d4d01c37f1e4a97f6fb1adecea18825fff4a43fc/src/PassportNFT.sol#L157

### Vulnerability Details

After users mint passports for themself and when they visit another country, the stamp should be put on this passport by the authority of the visiting country using the `createStamp()`, which will work as an attestation of the user’s passport being valid and his proof of visit.

But this is not the case with the `createStamp()`, because it checks the presence of the passport using `msg.sender` here

```js
     uint256 passportId = userToPassportId[msg.sender];
        Passport memory passport = passportIdToUser[passportId];
        if (passportId == 0) {
            revert PassportNFT__PassportNotMinted();
        }
```

### Impact

If the user can put stamp on their passport then the validity of the onchain attestations cannot be trusted. It is important that the function should be called by the authorized person or country.

### Mitigation

To mitigate this issue we can use the below code

```js
function createStamp(
        string calldata name,
        string calldata country,
        address visitor
    ) public onlyAuthorized returns (bytes32 attestationUID) {
        uint256 passportId = userToPassportId[visitor];
        Passport memory passport = passportIdToUser[passportId];
        if (passportId == 0) {
            revert PassportNFT__PassportNotMinted();
        }
        if (
            (bytes(name).length != bytes(passport.name).length) ||
            (keccak256(bytes(name)) != keccak256(bytes(passport.name)))
        ) {
            revert PassportNFT__DetailsMismatch();
        }
        attestationUID = makeStampAttestation(
            msg.sender,
            name,
            country,
            block.timestamp
        );
        passportIdToUser[passportId].stampAttestions.push(attestationUID);
    }
```

## [L-2] Add more validation when user create passport

## Code line

https://github.com/0xBcamp/Sept23_Passport_NFT/blob/c04a5d8dbaee6fff31358e5f1c647d5299998611/src/PassportNFT.sol#L123

### Vulnerability details

In `createPassport()` user can create passport with empty inputs.The function should verify the inputs before minting the passport

## [QA-1] No need to double check for name length in `createStamp()`

### Code Line

https://github.com/0xBcamp/Sept23_Passport_NFT/blob/30db73f25eeca96ce594c5ea68339f658eb31b29/src/PassportNFT.sol#L166

### Details

Here we don’t need to check for the length of the name. Instead checking if name is same is enough.

```js
if (
      (bytes(name).length != bytes(passport.name).length) ||
            (keccak256(bytes(name)) != keccak256(bytes(passport.name)))
        ) {
            revert PassportNFT__DetailsMismatch();
        }
```

### Mitigation

Remove the check for the length of the name

```js
if (
            keccak256(bytes(name)) != keccak256(bytes(passport.name))
        ) {
            revert PassportNFT__DetailsMismatch();
        }
```
