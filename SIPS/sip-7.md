---
sip: 7
title: Zone Interface for Server-Signed Orders
description: A consistent zone interface for signed Seaport orders.
author: Ryan Ghods (@ryanio)
discussions-to: https://github.com/ProjectOpenSea/SIPs/discussions/5
status: Draft
type: Standards
category (*only required for Standards Track): Interface
created: 2022-12-22
requires (*optional): 5, 6
---

_This document is currently WIP. Please suggest improvements or changes in the discussions-to link above._

## Abstract

This SIP outlines an interface for zones to provide server-signed orders. This allows marketplaces and liquidity providers to provide just-in-time signatures for fulfilling orders, which can lead to features like gasless order cancellations. This interface is proposed as a SIP to ensure fulfillers can follow a standard procedure for procuring signatures for orders.

## Motivation

This document describes an interface for signed orders so the Seaport ecosystem can rely on a standard way to procure order signatures. Server-side signed orders can provide helpful user-facing benefits, like gasless cancellations and protection against fulfilling orders against stolen items or fraudulent activity by ceasing signature output for certain orders.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Signature Verification

The zone MAY provide any method of signature verification. While the zone described in this specification is RECOMMENDED to use EIP-712 signatures for ease of consistency, an implementer MAY be interested in providing substitute or additional signature verification methods, like EIP-1271 for contract signature verification or BLS signatures for the ability to aggregate multiple signatures. If a new signature verification method is used, it is RECOMMENDED to write a new SIP requiring this one so others can follow the same format.

#### EIP-712

If the zone is implementing EIP-712 signatures, it MUST follow the below format for the typed structured data `SignedOrder` so zones and signature providers following this spec can be compatible with each other.

```javascript
SignedOrder: [
  { name: "fulfiller", type: "address" },
  { name: "expiration", type: "uint256" },
  { name: "orderHash", type: "bytes32" },
  { name: "context", type: "bytes" },
];
```

The `fulfiller` may be the zero address if the fulfillment is not restricted. If the fulfiller is not the zero address and the `fulfiller` from `validateOrder()` is not that address, it MUST revert with `error InvalidFulfiller(address expectedFulfiller, address actualFulfiller, bytes32 orderHash);`.

The data for verifying a signed order is sent as part of the order's `extraData`. The `extraData` MUST be formatted according to [SIP-6](./sip-6.md) with version byte prefix `00` for variable-length `extraData`.

| field                                            | bytes  |
| ------------------------------------------------ | ------ |
| SIP-6 version byte: MUST be 00                   | 0-1    |
| expected fulfiller (can be zero address for any) | 1-21   |
| expiration timestamp (uint96)                    | 21-33  |
| signature (EIP-2098 64 byte compact sig)         | 33-97  |
| optional variable context data                   | 97-end |

If the signature is expired, the zone MUST revert with `error SignatureExpired(uint256 currentTimestamp, uint256 expiration, bytes32 orderHash);`

If the recovered signer is unknown, the zone MUST revert with `error SignerNotApproved(address signer, bytes32 orderHash);`

If the `extraData` is missing the zone MUST revert with `error MissingExtraData()`.

If the `extraData` is unable to to be parsed properly due to an expected size or format, the zone MUST revert with `error InvalidExtraData()`.

### Signer Authorization and Deauthorization

The zone MUST provide methods for authorizing and deauthorizing signers only callable by the contract owner, `function authorizeSigner(address signer)` and `function deauthorizeSigner(address signer)`.

When a signer is added it MUST emit the event `event SignerAdded(address signer);`. When removed it MUST emit the event `event SignerRemoved(address signer);`.

Once a signer is deauthorized, it MUST NOT be able to be reactivated, to protect against compromised keys. If a deauthorized signer is attempted to be reauthorized, the contract MUST revert with `error SignerCannotBeReauthorized()`

If a duplicate signer is added it MUST revert with `error SignerAlreadyActive(address signer)`. If a signer not found is removed it MUST revert with `error SignerNotActive(address signer);`

Because the contract owner of the zone is in ultimate control of the zone and approving orders, it is RECOMMENDED to use a multi-signature wallet with a minimum number of confirmations for increased security.

### Zone Interface

The zone MUST provide a `validateOrder()` function that adheres to the Seaport zone interface to decode the extra data and validate the signature. If the signature is from an approved signer, it MUST return the validateOrder selector to signal success.

The zone MUST provide an `information()` view function, that returns the contract's EIP-712 domain separator.

The zone MUST provide `getSeaportMetadata()` as described in [SIP-5](./sip-5.md), that returns this SIP as a valid schema.

The zone MUST provide a view function to get the API endpoint to request signatures `function getAPIEndpoint() external view returns (memory string apiEndpoint)` that MUST follow the specification for API request and response payloads.

#### Signer API Request and Response Payload Format

The `apiEndpoint` MUST accept a JSON payload of:

```json
{ "orders": [{}, {}], "fulfiller": "0x..." }
```

The `apiEndpoint` MUST respond with a valid response containing formatted orders and/or errors of:

```json
{
  "orders": [{}, {}],
  "errors": { "${orderHash}": { "error": "", "message": "" } }
}
```

The returned orders MUST include `extraData` formatted with the data that would pass validation in the zone.

The valid error message responses are as follows:

| error                    | reason                                                                                                      |
| ------------------------ | ----------------------------------------------------------------------------------------------------------- |
| UnknownOrder             | If the order is not known to the signer                                                                     |
| UnknownZone              | If the zone is not known to the signer                                                                      |
| SignaturesNoLongerVended | If the signer is no longer vending signatures for the order, for example it has been fulfilled or cancelled |
| FulfillerRequired        | If the signer is requiring the order must require a specific fulfiller                                      |

The error `message` field MAY contain additional context or data.

If the rate limit for the API endpoint is exceeded, it MUST return with HTTP Error 429.

## Rationale

This specification was developed to ensure a consistent experience to deliver and obtain signatures for signed orders.

## Backwards Compatibility

As a newly proposed standard there is no issue with backwards compatibility.

## Test Cases

(WIP) Test cases are located in the Seaport repository at [`test/zones/SignedZone.spec.ts`](https://github.com/ProjectOpenSea/seaport/blob/9c91cf0a1f4a42c42ca093c16f4f907661ae5502/test/zones/SignedZone.spec.ts).

## Reference Implementation

(WIP) The reference implementation can be found in the Seaport repository at [`contracts/zones/SignedZone.ts`](https://github.com/ProjectOpenSea/seaport/blob/9c91cf0a1f4a42c42ca093c16f4f907661ae5502/contracts/zones/SignedZone.ts).

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
