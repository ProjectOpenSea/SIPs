---
sip: 7
title: Interface for Server-Signed Orders
description: A consistent interface for Seaport zones and contract offerers that incorporate third-party signatures.
author: 0age (@0age), Ryan Ghods (@ryanio)
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

Zones or contract offerers that do not implement additional SIPs must support extraData version byte `0x00` in accordance with SIP-6, while zones or contract offerers that implement additional SIPs with their own data requirements will require other version bytes.

Zones or contract offerers implementing SIP-7 MUST return a schema with an `id` of 7 as part of the `schemas` array returned by `getSeaportMetadata()` in accordance with SIP-5. They also MUST return an associated `metadata` parameter on the returned schema, decoded as `(bytes32 domainSeparator, string memory apiEndpoint, uint256[] memory substandards, string memory documentationURI)`.

### Signature Verification

The zone or contract offerer MAY provide any method of signature verification. While it is RECOMMENDED to use EIP-712 signatures for ease of consistency, an implementer MAY be interested in providing substitute or additional signature verification methods, like EIP-1271 for contract signature verification or BLS signatures for the ability to aggregate multiple signatures. If a new signature verification method is used, it is RECOMMENDED to write a new SIP requiring this one so others can follow the same format.

#### EIP-712

If the zone or contract offerer is implementing EIP-712 signatures, it MUST follow the below format for the typed structured data `SignedOrder` so contracts and signature providers following this spec can be compatible with each other.

```javascript
SignedOrder: [
  { name: "fulfiller", type: "address" },
  { name: "expiration", type: "uint64" },
  { name: "orderHash", type: "bytes32" },
  { name: "context", type: "bytes" },
];
```

The `fulfiller` MAY be the zero address if the fulfillment is not restricted. If the fulfiller is not the zero address and the `fulfiller` from `validateOrder()` is not that address, it MUST revert with `error InvalidFulfiller(address expectedFulfiller, address actualFulfiller, bytes32 orderHash);`.

The data for verifying a signed order is sent as part of the order's `extraData` and must contain at least 92 bytes. The `extraData` MUST be formatted according to [SIP-6](./sip-6.md) based on the other SIPs returned in accordance with SIP-5.

| field                                                | bytes  |
| ---------------------------------------------------- | ------ |
| expected fulfiller (SHOULD be zero address for any)  | 0-20   |
| expiration timestamp (uint64)                        | 20-28  |
| signature (MUST be EIP-2098 64 byte compact sig)     | 28-92  |
| variable context data (format based on substandards) | 92-end |

If the signature is expired, the zone or contract offerer MUST revert with `error SignatureExpired(uint256 currentTimestamp, uint256 expiration, bytes32 orderHash);`

If the recovered signer is unknown, the zone or contract offerer MUST revert with `error SignerNotActive(address signer, bytes32 orderHash);`

If the `extraData` component is unable to to be parsed properly due to unexpected size or format, the zone or contract offerer MUST revert with `error InvalidExtraData(string reason, bytes32 orderHash)`.

The `domainSeparator` used to recover the signer MUST check if the `block.chainid` has changed to recalculate the correct `domainSeparator` for security purposes around forks.

### Signer Authorization and Deauthorization

When a new signer is added to a zone or contract offerer, that contract MUST emit the event `event SignerAdded(address signer);`. When removed it MUST emit the event `event SignerRemoved(address signer)`.

Once a signer is removed, it MUST NOT be able to be reactivated, to protect against compromised keys. If a removed signer is attempted to be added, the contract MUST revert with `error SignerCannotBeReauthorized()`.

If a duplicate signer is added it MUST revert with `error SignerAlreadyActive(address signer)`. If a signer not found is removed it MUST revert with `error SignerNotActive(address signer)`.

If a signer is trying to be added that is the zero address, it MUST revert with `error SignerCannotBeZeroAddress()`.

It is RECOMMENDED that methods for adding or removing signers or updating API information only allow an authorized owner such as a multi-signature wallet with a minimum number of confirmations for increased security.

If methods for adding and remove signers are incorporated into the zone or contract offerer, it is RECOMMENDED to utilize `function addSigner(address signer)` and `function removeSigner(address signer)` for that purpose.

### Interface

If the contract implementing SIP-7 is a zone, it MUST provide a `validateOrder()` function that adheres to the Seaport zone interface to decode the extra data component and validate the signature. If the signature is from an approved signer, it MUST return the `validateOrder` selector to signal success.

If the contract implementing SIP-7 is a contract offerer, it MUST provide `generateOrder` and `ratifyOrder()` functions that adheres to the Seaport contract offerer interface, where at least one of the specified functions will decode the extra data component and validate the signature. If the signature is from an approved signer, it MUST return a valid order from `generateOrder` in accordance with the specified inputs, and MUST return the `ratifyOrder` selector from the call to `ratifyOrder` to signal success.

The zone or contract offerer SHOULD provide an `sip7Information()` view function, that returns the contract's EIP-712 domain separator and the API endpoint that follows the specification for API request and response payloads: `function sip7Information() external view returns (bytes32 domainSeparator, string memory apiEndpoint, uint256[] memory substandards, string memory documentationURI)`. If included, this information MUST match the information returned as metadata by the associated `getSeaportMetadata()` schema. Inclusion of this view function is strongly recommended for improved compatibility with existing tooling, as otherwise custom decoding of the metadata returned by SIP-6 will be required.

If the zone or contract offerer is able to update the API endpoint directly, it is RECOMMENDED to utilize `function updateAPIEndpoint()` for that purpose.

It is RECOMMENDED that the zone or contract offerer return a URI for `documentationURI` describing the behavior of the contract in question in more detail; the zone or contract offerer MAY return an empty string if no documentation is required or otherwise available.

The zone or contract offerer MUST provide `getSeaportMetadata()` as described in [SIP-5](./sip-5.md), that returns this SIP as a valid schema and metadata encoded as `(bytes32 domainSeparator, string memory apiEndpoint, uint256[] memory substandards, string memory documentationURI)`.

If the zone or contract offerer allows for active signers to interact with the zone, it is RECOMMENDED for the contract to provide `function getActiveSigners() external view returns (address[] memory signers);` so signers with active permissions can be more easily tracked and queried.

### Substandards

The `context` argument will be populated based on the "substandards" specified by the zone or contract offerer; these substandards will be encoded in accordance with SIP-6 versioning with the assumption that all necessary data is to be treated as "variable" data arrays. The ordering for each encoded data segment included as part of context, supplied as part of the server API request, and returned as part of the server API response will be dictated by the order that the zone or contract offerer returns the substandard IDs.

If substandards are being used, each encoded data segment as part of the context MUST start with the SIP-7 byte identifying the substandard ID below. The byte SHOULD be 1-indexed, but for gas efficiency reasons, 00 MAY also be used as an alias to reference substandard ID 1. If no substandard is used there MUST be no substandard version byte or additional substandard data provided.

Initial substandards include:
| substandard ID | description | decoding scheme | substandard request data supplied to API | substandard response data returned from API |
| -------------- | --------------------------------------------------------- | ----------------------------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 1 | required identifier for first returned received item | `(uint256)` | `{"requestedIdentifier": "123..."}` | `{"requiredIdentifier": "123..."}` |
| 2 | required initial "tip" | `(uint8, address, uint256, uint256, address)` | `{"requestedTip": null OR {"itemType": "1", "token": "abc", ...}}` | `{"requiredTip": {"itemType": "1", "token": "abc", ...}}` |
| 3 | required hash of full ReceivedItem array | `(bytes32)` | `{"requestedReceivedItems": null OR [{"itemType": "1", "token": "abc", ...}, ...]}` | `{"requiredReceivedItems": [{"itemType": "1", ...}, ...], "requiredReceivedItemsHash": "0xabc..."}` |
| 4 | required order hashes included as part of fulfillment | `(bytes32[])` | `{"requestedIncludedOrderHashes": null OR ["0xabc...", ...]}` | `{"requiredIncludedOrderHashes": ["0xabc...", ...]}` |
| 5 | required order hashes NOT included as part of fulfillment | `(bytes32[])` | `{"requestedExcludedOrderHashes": null OR ["0xabc...", ...]}` | `{"requiredExcludedOrderHashes": ["0xabc...", ...]}` |
| 6 | required hash of full ReceivedItem array scaled to 100% of order | `(uint256, bytes32)` | `{}` | `{"originalFirstOfferItemAmount": "123...", "requiredReceivedItemsHash": "0xabc..."}` |

Additional substandards MAY be specified in subsequent SIPs that inherit SIP-7.

### Signer API Request and Response Payload Format

The `apiEndpoint` MUST accept a JSON payload of:

```json
{
  "chainId": "0x01",
  "marketplaceContract": "0x...",
  "orderParameters": "OrderParameters struct",
  "fulfiller": "0x...",
  "substandardRequests": ["..."]
}
```

`substandardRequests` MUST be an array of objects where each object is formatted in accordance with the appropriate API requests for the specified substandard with the corresponding index.

The `apiEndpoint` MUST respond with a valid response for the order:

```json
{
  "extraDataComponent": "0x...",
  "orderParameters": "OrderParameters struct",
  "substandardResponses": ["..."]
}
```

OR an error:

```json
{
  "error": "UnknownOrder",
  "message": "The order cannot be found"
}
```

`substandardResponses` MUST be an array of objects where each object is formatted in accordance with the appropriate API responses for the specified substandard with the corresponding index.

The returned `orderParameters` MUST pass validation when supplied, and must include any modified fields including required tips.

The returned `extraDataComponent` MUST pass validation on the zone when supplied as part of the order's extraData or on the contract offerer when supplied as part of the order's context in accordance with SIP-6. Note that for zones or contract offerers that implement SIP-7 and no other SIPs that require data components, the zone or contract offerer MUST pass validation on said zone or contract offerer with an SIP-6 version byte of `0x00` followed by the returned `extraDataComponent` bytes array.

The valid error message responses are as follows:

| error                    | reason                                                                                                       |
| ------------------------ | ------------------------------------------------------------------------------------------------------------ |
| UnknownOrder             | If the order is not known to the signer or does not contain the correct zone or contract offerer             |
| SignaturesNoLongerVended | If the signer is no longer vending signatures for the order, for example it has been fulfilled or cancelled  |
| FulfillerRequired        | If the signer is requiring the order must require a specific fulfiller or the fulfiller is not authenticated |
| SubstandardNotValid      | If the signer does not receive appropriate input data for a required substandard or cannot meet requirements |

The error `message` field MAY contain additional context or data.

It is RECOMMENDED to add a rate limit to the API endpoint so fulfillers cannot simply continue to request many signatures for orders they do not intend to fulfill. If the rate limit for the API endpoint is exceeded, it MUST return with HTTP Error 429.

If a fulfiller is specified, the caller MUST have first authenticated with the API via EIP-4361 and MUST provide an associated token as part of the header of the request. The server MUST reject the request with a `FulfillerRequired` error if a valid token is not supplied.

The `apiEndpoint` MUST have a way of emitting an event that there is no longer intent to sign for an order. This is to help external integrators keep their order books up-to-date by invalidating orders that are no longer fulfillable. This is RECOMMENDED to be emitted only after the last vended signature for the order has expired, to avoid the order still being fulfilled.

## Rationale

This specification was developed to ensure a consistent experience to deliver and obtain third-party signatures for Seaport orders that require them.

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
   1. If a signer is compromised, orders that haven't reached their endTime are at risk of becoming fulfillable. The owner should immediately deauthorize the compromised signer so that its signatures are no longer valid.
1. Compromised zone ownership
   1. If the owner of the zone or contract offerer is compromised, orders that haven't reached their endTime are at risk of becoming fulfillable. The owner should immediately cancel high-value orders on-chain with `cancel()`, optionally encourage users to `incrementCounter()` to cancel their own orders, and begin creating new orders with a new zone or contract offerer that is properly owned and secured.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
