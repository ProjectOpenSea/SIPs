---
sip: 6
title: Multi-Zone ExtraData
description: A format for supplying order extraData for multiple zones.
author: 0age (@0age), Ryan Ghods (@ryanio)
discussions-to: https://github.com/ProjectOpenSea/SIPs/discussions/4
status: Draft
type: Standards
category (*only required for Standards Track): Interface
created: 2023-01-12
requires (*optional): 5
---

_This document is currently WIP. Please suggest improvements or changes in the discussions-to link above._

## Abstract

A standardized method for constructing order `extraData` for consumption by zones and `context` for consumption by contract offerers.

## Motivation

Restricted orders (involving zones) and contract orders (involving contract offerers) often require the fulfiller to construct and supply valid `extraData` as part of fulfillment. Moreover, these zones and contract offerers may implement multiple SIPs at once — for example, a zone might require both a proof of inclusion in an allowlist as well as a server-side signature. This SIP defines a standard for constructing order `extraData` that can be consumed by a zone or contract offerer that implements an arbitrary number of SIPs.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

Each zone-specific SIP that relies on SIP-6 MUST designate whether it requires no extra data, a "fixed" data array, a "variable" data array, or both a "fixed" and "variable" data array. Each contract-order-specific SIP that relies on SIP-6 MUST designate whether it requires no extra data (e.g. context) or a "variable" data array.

A "fixed" data array indicates that the data to be supplied is fully specified at the time of order creation in accordance with the relevant SIP, and will be compared to the supplied `zoneHash` parameter on the order in question. The methodology for deriving the hash to compare against will depend on the number of fixed data arrays required by the zone. Any zone requiring fixed data arrays MUST hash those fixed data arrays in accordance with the specified version and compare against the supplied zone hash, and reject any extraData that does not result in a hash of fixed data that matches the supplied zone hash.

A "variable" data array indicates that the supplied data is not fully specified at the time of order creation, but instead has some dynamic component that will be provided by the fulfiller in accordance with the relevant SIP. SIPs for both zones and contract orders may depend on variable data (contract orders do not specify fixed data).

If `extraData` is supplied as part of a SIP-6-compliant order, it MUST be prefixed with a `version` byte that follows the format in the below table.

| version byte | description                                                  | decoding scheme                                 | fixed data hashing scheme                                                                   |
| ------------ | ------------------------------------------------------------ | ----------------------------------------------- | ------------------------------------------------------------------------------------------- |
| 0x00         | single variable data array                                   | `extraData[1:]` (no offset / length)            | n/a                                                                                         |
| 0x01         | single fixed data array                                      | `extraData[1:]` (no offset / length)            | `keccak256(fixedDataArray)`                                                                 |
| 0x02         | single variable data array and single fixed data array       | `abi.decode(extraData[1:], (bytes, bytes))`     | `keccak256(fixedDataArray)`                                                                 |
| 0x03         | multiple variable data arrays                                | `abi.decode(extraData[1:], (bytes[]))`          | n/a                                                                                         |
| 0x04         | multiple fixed data arrays                                   | `abi.decode(extraData[1:], (bytes[]))`          | `keccak256(abi.encode(keccak256(fixedDataArrays[0]), keccak256(fixedDataArrays[1]), ...]))` |
| 0x05         | multiple variable data arrays and multiple fixed data arrays | `abi.decode(extraData[1:], (bytes[], bytes[]))` | `keccak256(abi.encode(keccak256(fixedDataArrays[0]), keccak256(fixedDataArrays[1]), ...]))` |

Zones and contract offerers MUST accept extraData encoded using the simplest possible version that adheres to the requirements of the SIPs they implement. Zones and contract offerers MUST reject encodings that do not provide for sufficient variable and/or fixed data arrays to satisfy each of the SIPs they implement by reverting with an `UnsupportedExtraDataVersion(uint8 version)` custom error. Zones and contract offerers MAY support more complex encodings than strictly required by the SIPs they support. Note that the hashing scheme for fixed data arrays will vary based on whether a single array or multiple arrays are required; in these cases, the simplest possible version MUST be used when selecting a fixed data hashing scheme for constructing the zone hash as part of order creation.

Zones and contract offerers MUST decode relevant data arrays based on the order that schema IDs are returned by `getSeaportMetadata()` in accordance with SIP-5. If the supplied extraData cannot be decoded using the specified version, or does not contain sufficient array elements to satisfy each of the SIPs implemented, the zone or contract offerer MUST revert, and SHOULD revert with an `InvalidExtraDataEncoding(uint8 version)` custom error.

By way of example, if a zone implements SIP-X, SIP-Y, and SIP-Z and returns schema IDs in the same order when calling `getSeaportMetadata()`, and each SIP requires a single variable data array, then the zone will accept version 0x03 (and optionally version 0x05), rejecting any other versions, and will decode the first element (index 0) in the array of variable data arrays as the variable data array for SIP-X, the second element (index 1) as the variable data array for SIP-Y, and the third element (index 2) as the variable data array for SIP-Z.

When a contract is both a zone and a contract offerer, and implements both zone-specific SIPs and contract-order-specific SIPS, extraData construction MUST be based on the current context. In other words, if the contract is being interacted with as a zone, then contract-order-specific SIPs are to be disregarded; alternately, when the contract is being interacted with as a contract offerer, then zone-specific SIPs are to be disregarded.

New version bytes MUST be added in new SIPs that require SIP-6.

## Rationale

As Seaport zones are developed, it is useful to have a standard way of constructing `extraData` in a composable fashion so that zones and contract offerers can implement multiple SIPs at once, and can broadcast their requirements based solely on those SIPs without requiring fulfillers to have additional context on how to supply the required data for each independent SIP.

## Backwards Compatibility

As a new standard there are no issues with backwards compatibility.

## Test Cases

## Reference Implementation

## Security Considerations

Zone and contract offerer implementations may deviate from this standard, either erroneously or maliciously — fulfillers and marketplaces should still analyse and simulate interactions with zones and contract offerers on an ongoing basis.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
