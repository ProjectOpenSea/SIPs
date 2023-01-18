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
requires (*optional):
---

_This document is currently WIP. Please suggest improvements or changes in the discussions-to link above._

## Abstract

A standard way to construct order `extraData` that can be consumed by multiple zones.

## Motivation

To allow orders to interface with multiple zones requiring different `extraData`, this SIP defines a standard for constructing order `extraData` that can be consumed by an end zone that checks multiple zones.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

The order `extraData` MUST be prefixed with a `version` byte that follows the format in the below table.

| version byte | `extraData` format                                                                                                                                                                             |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0            | `0x00<SIP-XXX-variable-data>`                                                                                                                                                                  |
| 1            | `0x01<SIP-XXX-fixed-data>`                                                                                                                                                                     |
| 2            | `0x02<offset to fixed><offset to variable><size of fixed><pointer to each fixed data array><size of variable><pointer to each variable data array><SIP-XXX-fixed-data><SIP-XXX-variable-data>` |

The SIP data in the `extraData` MUST be ordered numerically based on the SIP number.

New version bytes MUST be added in a new SIP that requires this one.

## Rationale

As Seaport zones are developed, it is useful to have a standard way of constructing `extraData` to be able to provide different data to multiple zones.

## Backwards Compatibility

As a new standard there are no issues with backwards compatibility.

## Test Cases

## Reference Implementation

## Security Considerations

To prevent data collisions this SIP takes into account both fixed and variable length data in `extraData`.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
