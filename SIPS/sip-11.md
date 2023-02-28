---
sip: 11
title: Percentage-of-sale-price Creator Earnings enforcement for Seaport-extended NFTs
description: A stardard for EIP-2981-compliant NFTs to enforce percentage-of-sale-price creator earnings payments.
author: 0age (@0age)
discussions-to: https://github.com/ProjectOpenSea/SIPs/discussions/13
status: Draft
type: Standards
category (*only required for Standards Track): Interface
created: 2023-2-27
requires (*optional): 5, 6, 10
---

_This document is currently WIP. Please suggest improvements or changes in the discussions-to link above._

## Abstract

This SIP outlines a methodology for Seaport-extended NFTs to enforce percentage-of-sales-price creator earnings. These NFTs provide orders that unlock transferability of specified tokens for the duration of the Seaport fulfillment, where the returned consideration item is based on the sale price of the NFT in a simultaneously-provided listing. This interface is proposed as a SIP to ensure fulfillers can follow a standard procedure for interacting with this variety of NFT.

## Motivation

This document describes an interface for Seaport-extended NFTs that enforce percentage-of-sales-price creator earnings so the broader ecosystem can rely on a standard way to interact with those NFTs. Creators can ensure that sales-price-based fees are enforced secondary sales in the vast majority of scenarios, without relying on the marketplaces where those NFTs are bought and sold to enforce them on their behalf.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Requirements

SIP-11 does not require any additional data arrays beyond the single variable data array required as part of SIP-10. Instead, it extends the SIP-10 substandards with the following:

| substandard ID | description | decoding scheme | notes |
| -- | ---------- | --------------------------------------------------------- | ----------------------------------------------- |
| 12 | transfer activation for a single, specific minted token | `(uint256, uint256)` | token and sale price represented by context, minimumReceived empty |
| 13 | transfer activation for multiple, arbitrary minted tokens | `(uint256, uint256)` | number of token transfers and total combined sale price represented by context (no support for multiple hops) |
| 14 | transfer activation for a single, specific minted token with additional data | `(uint256, uint256, bytes)` | token and sale price represented by context, minimumReceived empty |
| 15 | transfer activation for multiple, specific minted tokens with additional data | `(uint256[], uint256, bytes)` | tokens and total combined sale price represented by context, minimumReceived empty |
| 16 | transfer activation for multiple, arbitrary minted tokens with additional data | `(uint256, uint256, bytes)` | number of token transfers and total combined sale price represented by context (no support for multiple hops) |

Additional SIP-10 substandards MAY be specified in subsequent SIPs that inherit SIP-11. Additional data requirements can leverage the substandard IDs above but SHOULD incorporate other SIPs that will dicate how to format the additional data.

NFT contract offerers that do not implement additional SIPs must support extraData version byte `0x00` in accordance with SIP-6, while contract offerers that implement additional SIPs with their own data requirements will require other version bytes.

NFT contract offerers implementing SIP-11 MUST return a schema with an `id` of 11 as part of the `schemas` array returned by `getSeaportMetadata()` in accordance with SIP-5. They also MUST return a schema with an `id` of 10 along with any implemented substandards on the associated `metadata` parameter for the returned SIP-10 schema, decoded as `(uint256[] memory substandards, string memory documentationURI)`.

### Fee Derivation

The NFT MUST implement EIP-2981 and MUST adhere to the receiver and amount returned by `royaltyInfo` when deriving the fee for a given tokenId and sale price. As with EIP-2981, the fee MUST be denominated in the same item as the item used to derive the sale price.

### Interface & Implementation Requirements

NFT contracts implementing SIP-11 MUST implement a `validateOrder` function that adheres to the zone interface, and that observes a sale and records the sale price (or combined sale price where multiple items are involved). The NFT contract must also implement `generateOrder` and `ratifyOrder` functions that adhere to the Seaport contract offerer interface, where at least one of the specified functions will decode the extra data component and where the `ratifyOrder` function will confirm that the recorded sale price does not exceed the supplied sale price used to derive the payment. The implementing NFT contract MUST activate transferability of the given token(s) in `generateOrder` and MUST deactivate transferability of the given token(s) in `ratifyOrder`.

NFT contracts also MUST implement a `previewOrder` view function that adheres to the Seaport contract offerer interface and that will return the required `consideration` array for the requested input parameters and chosen substandard in direct accordance with the result returned from `royaltyInfo`; this array will then need to be supplied as part of the associated contract order as the `maximumSpent` array.

It is RECOMMENDED that the NFT contract return a URI for `documentationURI` describing the behavior of the contract in question in more detail; the contract offerer MAY return an empty string if no documentation is required or otherwise available.

The NFT contract MUST provide `getSeaportMetadata()` as described in [SIP-5](./sip-5.md), that returns this SIP and SIP-10 as a valid schema.

## Rationale

This specification was developed to ensure a consistent experience to enable fee enforcement on token transfers for Seaport-extended NFTs that require them.

## Backwards Compatibility

As a newly proposed standard there is no issue with backwards compatibility.

## Test Cases

Test cases are still under development.

## Reference Implementation

Reference implementation is still under development.

## Security Considerations

In addition to the potential routes for fee circumvention outlined in SIP-10, percentage-of-sales-based creator earnings are susceptible to circumvention methods involving out-of-band payments. By way of example, a buyer could circumvent most of a sales-based royalty by placing their actual sale tokens into a contract that can be only redeemed by the seller once a private listing with a much lower sale price has been fulfilled (by ensuring that the order status shows as such), or by utilizing a conditional listing.

This circumvention vector (and others raised in SIP-10) should be considered by NFT creators before deciding to implement this standard, with potential alternative methods for fee enforcement that can be activated.

Another important note is that implementation of this SIP naturally restricts what operators may transfer the underlying NFT, which can damage composability. While Seaport (and Seaport forks or analogues that implement the contract offerer interface) can still accomplish many of the same actions via native integrations, adapters, or other techniques, there is still the danger that other protocols may be incompatible with NFTs that implement this SIP. NFT creators should consider the relative tradeoffs carefully and potentially include mechanisms for adding and removing allowed operators or deactivating the transfer restriction mechanic entirely.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
