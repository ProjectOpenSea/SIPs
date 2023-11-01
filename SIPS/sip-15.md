---
sip: 15
title: Interface for Dynamic Traits Enforcement
description: A Seaport interface for specifying and enforcing values of ERC-7496 Dynamic Traits.
author: Ryan Ghods (@ryanio), James Wenzel (emo.eth)
discussions-to: https://github.com/ProjectOpenSea/SIPs/discussions/19
status: Draft
type: Standards
category (*only required for Standards Track): Interface
created: 2023-08-14
requires (*optional): 5, 6
---

## Abstract

This SIP defines a standard interface for Seaport zones and contract offerers to specify and enforce ERC-7496 Dynamic Traits during order fulfillment.

## Motivation

Token metadata is typically specified at offchain locations but with ERC-7496 Dynamic Traits certain metadata values can be queried onchain. This can help protect NFTs against frontrunning during purchase where the value of traits changes the expected value of the NFT. With this SIP, zones and contract offerers can enforce the value of certain traits at time of order fulfillment.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Requirements

This SIP requires a single fixed data array as part of supplied `extraData`.

Zones or contract offerers that do not implement additional SIPs must support extraData version byte `0x00` in accordance with SIP-6, while zones or contract offerers that implement additional SIPs with their own data requirements will require other version bytes.

Zones or contract offerers implementing this SIP MUST return a schema with an `id` of X as part of the `schemas` array returned by `getSeaportMetadata()` in accordance with SIP-5. They also MUST return an associated `metadata` parameter on the returned schema, decoded as `(bytes32 domainSeparator, string memory apiEndpoint, uint256[] memory substandards, string memory documentationURI)`.

### Metadata Verification

The data for verifying a protected order is sent as part of the order's `extraData` and must contain at least XX bytes. The `extraData` MUST be formatted according to [SIP-6](./sip-6.md) based on the other SIPs returned in accordance with SIP-5.

| field                                                | bytes |
| ---------------------------------------------------- | ----- |
| variable context data (format based on substandards) | 0-end |

If the order does not conform to the expected metadata values, the zone or contract offerer MUST revert with `error InvalidDynamicTraitValue(address token, bytes32 traitKey, bytes32 expectedTraitValue, bytes32 actualTraitValue);`

### Interface

If the implementing contract is a zone, it MUST provide a `validateOrder()` function that adheres to the Seaport zone interface to decode the extra data component.

If the implementing contract is a contract offerer, it MUST provide `previewOrder()`, `generateOrder()`, and `ratifyOrder()` functions that adheres to the Seaport contract offerer interface that decodes the extra data.

### Substandards

The `context` argument MUST be populated based on the "substandards" specified by the zone or contract offerer; these substandards will be encoded in accordance with SIP-6 versioning with the assumption that all necessary data is to be treated as "variable" data arrays.

The `context` MUST start with a byte identifying the substandard ID below. The byte SHOULD be 1-indexed, but for gas efficiency reasons, 00 MAY also be used as an alias to reference substandard ID 1.

todo MUST be compact

Initial substandards include:

| substandard ID | description                                                                                                  | decoding scheme                                                                 |
| -------------- | ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------- |
| 0             | first consideration item, comparison "equal to", single trait key, zero trait value  | `(bytes32 traitKey)` TODO think about this should be in the zone hash, when no extra data is provided |
| 1             | token address and id from first offer item | `(uint8 comparisonEnum, bytes32 traitValue, bytes32 traitKey)` |
| 2             | token address and id from the first consideration item | `(uint8 comparisonEnum, bytes32 traitValue, bytes32 traitKey)` |
| 3              | single token id, single trait key and value(single)                                                                    | `(uint8 comparisonEnum, address token, uint256 tokenId, bytes32 traitValue, bytes32 traitKey)`   |
| 4              | multiple token ids, single trait key and value(multiple)                                                                | `(uint8 comparisonEnum, address token, uint256 tokenId, bytes32 traitValue, bytes32 traitKey)[]` |
| 5             | single token id, multiple traitKeys and values | `` |

| comparison enum | behavior                 |
| --------------- | ------------------------ |
| 0               | equal to                 |
| 1               | not equal to             |
| 2               | less than                |
| 3               | less than or equal to    |
| 4               | greater than             |
| 5               | greater than or equal to |

Additional substandards MAY be specified in subsequent SIPs that inherit this SIP.

## Rationale

This specification was developed to ensure a consistent experience to protect orders against frontrunning valuable traits.

## Backwards Compatibility

As a newly proposed standard there is no issue with backwards compatibility.

## Test Cases

Test cases are located in the [Dynamic Traits repository](https://github.com/ProjectOpenSea/dynamic-traits/blob/main/src/lib/DynamicTraits.sol).

## Reference Implementation

The reference implementation can be found in the [Dynamic Traits repository](https://github.com/ProjectOpenSea/dynamic-traits/blob/main/test/ERC721DynamicTraits.t.sol).

## Security Considerations

- For the zone to work as intended tokens must properly implement ERC-7496 Dynamic Traits.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
