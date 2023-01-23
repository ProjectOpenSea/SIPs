---
sip: 7
title: Zone Interface for Server-Signed Orders
description: A consistent zone interface for signed Seaport orders.
author: Ryan Ghods (@ryanio), 0age (@0age)
discussions-to: https://github.com/ProjectOpenSea/SIPs/discussions/5
status: Draft
type: Standards
category (*only required for Standards Track): Interface
created: 2022-12-22
requires (*optional): 5, 6
---

_This document is currently WIP. Please suggest improvements or changes in the discussions-to link above._

## Abstract

This SIP outlines an interface for zones or contract offerers to provide server-signed orders. This allows marketplaces and liquidity providers to provide just-in-time signatures for fulfilling orders, which can lead to features like gasless order invalidation as long as the signer refuses to provide a signature. This interface is proposed as a SIP to ensure fulfillers can follow a standard procedure for procuring signatures for orders.

## Motivation

This document describes an interface for signed orders so the Seaport ecosystem can rely on a standard way to procure order signatures. Server-side signed orders can provide helpful user-facing benefits, like gasless invalidation and protection against fulfilling orders against compromised items or fraudulent activity by ceasing signature output for certain orders.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Requirements

SIP-7 requires a single variable data array as part of supplied `extraData`.

Zones or contract orders that do not implement additional SIPs must support extraData version byte `0x00` in accordance with SIP-6, while zones or contract orders that implement additional SIPs with their own data requirements will require other version bytes.

Zones or contract orders implementing SIP-7 MUST return a schema with an `id` of 7 as part of the `schemas` array returned by `getSeaportMetadata()` in accordance with SIP-5. They also SHOULD return a URI linking to a description of any required documentation for the associated `metadata` parameter on the returned schema, especially if additional context data is expected by the implementing contract.

### Signature Verification

The zone MAY provide any method of signature verification. While the zone described in this specification is RECOMMENDED to use EIP-712 signatures for ease of consistency, an implementer MAY be interested in providing substitute or additional signature verification methods, like EIP-1271 for contract signature verification or BLS signatures for the ability to aggregate multiple signatures. If a new signature verification method is used, it is RECOMMENDED to write a new SIP requiring this one so others can follow the same format.

#### EIP-712

If the zone is implementing EIP-712 signatures, it MUST follow the below format for the typed structured data `SignedOrder` so zones and signature providers following this spec can be compatible with each other.

```javascript
SignedOrder: [
  { name: "fulfiller", type: "address" },
  { name: "expiration", type: "uint64" },
  { name: "orderHash", type: "bytes32" },
  { name: "context", type: "bytes" },
];
```

The `fulfiller` may be the zero address if the fulfillment is not restricted. If the fulfiller is not the zero address and the `fulfiller` from `validateOrder()` is not that address, it MUST revert with `error InvalidFulfiller(address expectedFulfiller, address actualFulfiller, bytes32 orderHash);`.

The data for verifying a signed order is sent as part of the order's `extraData` and must contain at least 92 bytes. The `extraData` MUST be formatted according to [SIP-6](./sip-6.md) based on the other SIPs returned in accordance with SIP-5.

| field                                               | bytes  |
| --------------------------------------------------- | ------ |
| expected fulfiller (SHOULD be zero address for any) | 0-20   |
| expiration timestamp (uint64)                       | 20-28  |
| signature (MUST be EIP-2098 64 byte compact sig)    | 28-92  |
| optional variable context data                      | 92-end |

If the signature is expired, the zone MUST revert with `error SignatureExpired(uint256 currentTimestamp, uint256 expiration, bytes32 orderHash);`

If the recovered signer is unknown, the zone MUST revert with `error SignerNotActive(address signer, bytes32 orderHash);`

If the `extraData` is unable to to be parsed properly due to unexpected size or format, the zone MUST revert with `error InvalidExtraData(string reason, bytes32 orderHash)`.

The `domainSeparator` used to recover the signer MUST check if the `block.chainid` has changed to recalculate the correct `domainSeparator` for security purposes around forks.

### Signer Authorization and Deauthorization

The zone MUST provide methods for adding and remove signers, `function addSigner(address signer)` and `function removeSigner(address signer)`.

When a signer is added it MUST emit the event `event SignerAdded(address signer);`. When removed it MUST emit the event `event SignerRemoved(address signer)`.

Once a signer is removed, it MUST NOT be able to be reactivated, to protect against compromised keys. If a removed signer is attempted to be added, the contract MUST revert with `error SignerCannotBeReauthorized()`.

If a duplicate signer is added it MUST revert with `error SignerAlreadyActive(address signer)`. If a signer not found is removed it MUST revert with `error SignerNotActive(address signer)`.

If a signer is trying to be added that is the zero address, it MUST revert with `error SignerCannotBeZeroAddress()`.

It is RECOMMENDED that methods for adding or removing signers or updating API information only allow an authorized owner such as a multi-signature wallet with a minimum number of confirmations for increased security.

### Zone Interface

The zone MUST provide a `validateOrder()` function that adheres to the Seaport zone interface to decode the extra data and validate the signature. If the signature is from an approved signer, it MUST return the validateOrder selector to signal success.

The zone MUST provide an `sip7Information()` view function, that returns the contract's EIP-712 domain separator and the API endpoint that follows the specification for API request and response payloads: `function sip7Information() external view returns (bytes32 domainSeparator, string memory apiEndpoint);`.

The zone MUST provide a method to update the API endpoint with `function updateAPIEndpoint();`

The zone MUST provide `getSeaportMetadata()` as described in [SIP-5](./sip-5.md), that returns this SIP as a valid schema.

If the zone allows for active signers to interact with the zone, it is RECOMMENDED for the zone to provide `function getActiveSigners() external view returns (address[] memory signers);` so signers with active permissions can be more easily tracked and queried.

### Signer API Request and Response Payload Format

The `apiEndpoint` MUST accept a JSON payload of:

```json
{
  "chainId": "0x01",
  "marketplaceContract": "0x...",
  "orderHash": "0x...",
  "fulfiller": "0x..."
}
```

The `apiEndpoint` MUST respond with a valid response for the order:

```json
{
  "extraData": "0x..."
}
```

OR an error:

```json
{
  "error": "UnknownOrder",
  "message": "The order cannot be found"
}
```

The returned extraData MUST pass validation on the zone when supplied as the order extraData

The valid error message responses are as follows:

| error                    | reason                                                                                                      |
| ------------------------ | ----------------------------------------------------------------------------------------------------------- |
| UnknownOrder             | If the order is not known to the signer                                                                     |
| UnknownZone              | If the zone is not known to the signer                                                                      |
| SignaturesNoLongerVended | If the signer is no longer vending signatures for the order, for example it has been fulfilled or cancelled |
| FulfillerRequired        | If the signer is requiring the order must require a specific fulfiller                                      |

The error `message` field MAY contain additional context or data.

It is RECOMMENDED to add a rate limit to the API endpoint so fulfillers cannot simply continue to request many signatures for orders they do not intend to fulfill. If the rate limit for the API endpoint is exceeded, it MUST return with HTTP Error 429.

The `apiEndpoint` MUST have a way of emitting an event that there is no longer intent to sign for an order. This is to help external integrators keep their order books up-to-date by invalidating orders that are no longer fulfillable. This is RECOMMENDED to be emitted only after the last vended signature for the order has expired, to avoid the order still being fulfilled.

## Rationale

This specification was developed to ensure a consistent experience to deliver and obtain signatures for signed orders.

## Backwards Compatibility

As a newly proposed standard there is no issue with backwards compatibility.

## Test Cases

Test cases are located in the Seaport repository at [`test/zones/SignedZone.spec.ts`](https://github.com/ProjectOpenSea/seaport/blob/024dcc5cd70231ce6db27b4e12ea6fb736f69b06/test/zones/SignedZone.spec.ts).

## Reference Implementation

The reference implementation can be found in the Seaport repository at [`contracts/zones/SignedZone.sol`](https://github.com/ProjectOpenSea/seaport/blob/024dcc5cd70231ce6db27b4e12ea6fb736f69b06/contracts/zones/SignedZone.sol).

## Security Considerations

There are several security considerations when using a zone for requiring server-signed orders.

1. Centralization of orders
   1. Server-signed orders lead to increased centralization since orders cannot be fulfilled without a valid signature. Care should be given to the liveness of the signing server to ensure orders can be readily fulfilled.
1. Compromised signers
   1. If a signer is compromised, orders that haven't reached their endTime are at risk of becoming fulfillable. The owner should immediately call `deauthorizeSigner()` so that its signatures are no longer valid.
1. Compromised zone ownership
   1. If the owner of the zone is compromised, orders that haven't reached their endTime are at risk of becoming fulfillable. The owner should immediately cancel high-value orders on-chain with `cancel()`, optionally encourage users to `incrementCounter()` to cancel their own orders, and begin creating new orders with a new zone that is properly owned and secured.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
