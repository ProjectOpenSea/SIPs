---
sip: 10
title: Interface for Seaport-extended NFTs
description: A consistent interface for NFTs that also operate as Seaport contract offerers.
author: 0age (@0age)
discussions-to: https://github.com/ProjectOpenSea/SIPs/discussions/
status: Draft
type: Standards
category (*only required for Standards Track): Interface
created: 2023-2-16
requires (*optional): 5, 6
---

_This document is currently WIP. Please suggest improvements or changes in the discussions-to link above._

## Abstract

This SIP outlines an interface for NFTs to serve as contract offerers. These NFTs then provide orders that unlock transferability of specified tokens for the duration of the Seaport fulfillment. This allows these contracts to declare consideration items based on the tokens being transfered, and Seaport will ensure that those items are transferred to the named recipients. In short, Seaport acts similarly to an "embedded" marketplace for the implementing NFTs. This interface is proposed as a SIP to ensure fulfillers can follow a standard procedure for interacting with Seaport-extended NFTs, either for primary sales, secondary sales, or both.

## Motivation

This document describes an interface for Seaport-extended NFTs so the broader ecosystem can rely on a standard way to interact with those NFTs. Creators can ensure that fees are enforced for both primary and secondary sales in the vast majority of scenarios, without relying on the marketplaces where those NFTs are bought and sold to enforce them on their behalf.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Requirements

SIP-10 requires a single variable data array as part of supplied `extraData`.

NFT contract offerers that do not implement additional SIPs must support extraData version byte `0x00` in accordance with SIP-6, while contract offerers that implement additional SIPs with their own data requirements will require other version bytes.

NFT contract offerers implementing SIP-10 MUST return a schema with an `id` of 10 as part of the `schemas` array returned by `getSeaportMetadata()` in accordance with SIP-5. They also MUST return an associated `metadata` parameter on the returned schema, decoded as `(uint256[] memory substandards, string memory documentationURI)`.

### Fee Derivation

The contract offerer MAY provide any method of fee derivation. Examples include fixed fees, fees based on contract-native bids, oracles or registry lookups, fees based on the items in additional conditional orders, mechanisms such as VRGDAs, and fees based on the level of activity on the contract. If a new fee derivation method is used, it is RECOMMENDED to write a new SIP requiring SIP-10 so others can follow the same format.

#### Top-level Data Formatting

The data for verifying a signed order is sent as part of the order's `extraData`. The `extraData` MUST be formatted according to [SIP-6](./sip-6.md) based on the other SIPs returned in accordance with SIP-5. If no subsequent data is provided, substandard 0 is assumed; otherwise, the first byte of data (following the SIP-6 version byte) represents the substandard in question and the remaining data MUST be formatted according to the named substandard. If the supplied substandard is not supported, the contract offerer MUST revert with `error InvalidSubstandard(uint8 substandard)`.

If the `extraData` component is unable to to be parsed properly due to unexpected size or format, the contract offerer MUST revert with `error InvalidExtraDataEncoding(uint8 version)`.

### Interface & Implementation Requirements

NFT contracts implementing SIP-10 MUST provide `generateOrder` and `ratifyOrder` functions that adheres to the Seaport contract offerer interface, where at least one of the specified functions will decode the extra data component. If the given substandard activates transferability (including the ability to mint currently unminted tokens), the implementing NFT contract MUST activate transferability of the given token(s) in `generateOrder` and MUST deactivate transferability of the given token(s) in `ratifyOrder`.

The NFT contract SHOULD provide a `canTransfer(uint256 tokenId) external view returns (bool isTransferable)` function that returns the current transferable status of a given tokenID. If included, this information MUST match the underlying transferable status of the given token. Inclusion of this view function is strongly recommended for improved compatibility with external integrations.

It is RECOMMENDED that the NFT contract return a URI for `documentationURI` describing the behavior of the contract in question in more detail; the contract offerer MAY return an empty string if no documentation is required or otherwise available.

The NFT contract MUST provide `getSeaportMetadata()` as described in [SIP-5](./sip-5.md), that returns this SIP as a valid schema and metadata encoded as `(uint256[] memory substandards, string memory documentationURI)`.

### Substandards

The `context` argument will be populated based on the "substandards" specified by the NFT contract offerer; these substandards will be encoded in accordance with SIP-6 versioning with the assumption that all necessary data is to be treated as "variable" data arrays. The ordering for each encoded data segment included as part of context, supplied as part of the server API request, and returned as part of the server API response will be dictated by the order that the NFT contract offerer returns the substandard IDs.

Initial substandards include:
| substandard ID | description | decoding scheme | notes
| -------------- | --------------------------------------------------------- | ----------------------------------------------- |
| 0 | transfer activation for a single, specific unminted token | n/a | token represented as a single minimumReceived item |
| 1 | transfer activation for a single, specific minted token | `(uint256)` | token represented by context, minimumReceived empty |
| 2 | transfer activation for multiple, specific minted tokens | `(uint256[])` | tokens represented by context, minimumReceived empty |
| 3 | transfer activation for multiple, specific unminted tokens | n/a | tokens represented as a group of minimumReceived items |
| 4 | transfer activation for multiple, arbitrary minted tokens | `(uint256)` | number of token transfers represented by context (no support for multiple hops) |
| 5 | transfer activation for multiple, arbitrary unminted ERC721 tokens | `(bytes)` | number to mint represented by single 1155 "synthetic" minimumReceived item |
| 6 | transfer activation for a single, specific unminted token with additional data | `(bytes)` | token represented as a single minimumReceived item |
| 7 | transfer activation for a single, specific minted token with additional data | `(uint256, bytes)` | token represented by context, minimumReceived empty |
| 8 | transfer activation for multiple, specific minted tokens with additional data | `(uint256[], bytes)` | tokens represented by context, minimumReceived empty |
| 9 | transfer activation for multiple, specific unminted tokens with additional data | `(bytes)` | tokens represented as a group of minimumReceived items |
| 10 | transfer activation for multiple, arbitrary minted tokens with additional data | `(uint256, bytes)` | number of token transfers represented by context (no support for multiple hops) |
| 11 | transfer activation for multiple, arbitrary unminted ERC721 tokens with additional data | `(bytes)` | number to mint represented by single 1155 "synthetic" minimumReceived item |

Additional substandards MAY be specified in subsequent SIPs that inherit SIP-10. Additional data requirements can leverage the substandard IDs above but SHOULD incorporate other SIPs that will dicate how to format the additional data.

## Rationale

This specification was developed to ensure a consistent experience to enable fee enforcement on token transfers for Seaport-extended NFTs that require them.

## Backwards Compatibility

As a newly proposed standard there is no issue with backwards compatibility.

## Test Cases

Test cases are still under development.

## Reference Implementation

A reference implementation implementing substandards 0 and 1 can be found in the Seaport repository (currently contained on the `creator-earnings-enforcer` working branch) at [`contracts/contractOfferers/SeaportExtendedNFT.sol`](https://github.com/ProjectOpenSea/seaport/blob/53ee96cbd995e9b379c4eab1f9399e0bf55ed256/contracts/contractOfferers/SeaportExtendedNFT.sol).

## Security Considerations

Security considerations will be provided as this SIP is further developed.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
