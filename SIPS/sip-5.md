---
sip: 5
title: Contract Metadata Interface for Seaport Contracts
description: An interface to describe contract compliance with SIPs.
author: 0age (@0age), Ryan Ghods (@ryanio)
discussions-to: https://github.com/ProjectOpenSea/SIPs/discussions/3
status: Draft
type: Standards
category (*only required for Standards Track): Interface
created: 2023-01-12
requires (*optional):
---

_This document is currently WIP. Please suggest improvements or changes in the discussions-to link above._

## Abstract

A standard way for external Seaport contracts, like zones and contract offerers, to signal which SIPs are supported by the contract.

## Motivation

It is helpful to know which SIPs external Seaport contracts adhere to in order to interact with them properly.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

The implementing contract MUST have the following `Schema` struct and `getSeaportMetadata` view function to identify the supported standards:

```solidity
/**
 * @dev Zones and contract offerers can communicate which schemas they implement
 *      along with any associated metadata related to each schema.
 */
struct Schema {
    uint256 id; /// Seaport Improvement Proposal (SIP) ID
    bytes metadata; /// Optional additional metadata
}

/**
 * @dev Returns Seaport metadata for this contract, returning the
 *      contract name and supported schemas.
 *
 * @return name    The contract name
 * @return schemas The supported SIPs
 */
function getSeaportMetadata() external view returns (
    string memory name,
    Schema[] memory schemas
);
```

The `name` is RECOMMENDED to be a uniquely identifiable name for the contract.

The `schemas` returned MUST be in numerical order of SIP ID from smallest to largest.

The implementing contract MUST adhere to [EIP-165](https://eips.ethereum.org/EIPS/eip-165) and provide a `supportsInterface` function that returns `true` for the interface of `getSeaportMetadata()`: `0x2e778efc`. The contract MUST also return `true` for `supportsInterface` of the Seaport interface being implemented.

## Rationale

As external Seaport contracts are developed it is useful to have a standard way of understanding the SIPs they support.

## Backwards Compatibility

As a new standard there are no issues with backwards compatibility.

## Test Cases

## Reference Implementation

## Security Considerations

There are no security considerations as this specification only proposes a metadata view function.

Contract authors should note until a SIP has a "final" status it may continue to change.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
