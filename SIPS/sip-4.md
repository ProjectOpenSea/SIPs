---
sip: 4
title: P2P Specification
description: A P2P spec for sharing Seaport orders
author: Ryan Ghods (@ryanio)
discussions-to: https://github.com/ProjectOpenSea/SIPs/discussions/2
status: Draft
type: Standards
category (*only required for Standards Track): Networking
created: 12-26-22
requires (*optional):
---

_This document is currently WIP. Please suggest improvements or changes in the discussions-to link above._

## Abstract

This document is designed to formalize a peer-to-peer specification for sharing Seaport orders. This allows marketplaces and liquidity providers to share Seaport orders with each other in a decentralized manner.

## Motivation

Since Seaport orders are stored off chain to reduce the cost of listing, this P2P specification helps for participants in the Seaport ecosystem to share orders with each other to increase discoverability and liquidity.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

The client implementing this protocol is RECOMMENDED to use the libp2p stack, as it is the basis of this specification.

The client MUST use libp2p's PeerId for identifying itself in the peer ecosystem.

The client MUST use NOISE encryption for exchanging data with peers. While Seaport orders do not necessarily need to be encrypted, the libp2p protocol requires an encryption mechanism to be used for exchanging node identities on connection.

The client MUST use Kademlia DHT for peer and content discovery and routing.

The client MUST use mplex for data multiplexing.

The client MUST use Gossipsub v1.1 for exchanging event-related data.

The client MAY use any method for bootstrapping into the peer network. It is RECOMMENDED to use libp2p-bootstrap and public nodes via dnsaddr.

The client MUST use Simple Serialize (SSZ) for encoding and decoding messages over the wire. The SSZ types for each message are included below.

### Order Validation

The client is RECOMMENDED to use [seaport-order-validator](https://github.com/ProjectOpenSea/seaport-order-validator) to validate orders, as it simplifies many calls to one contract call. The rules for order validation MUST at a minimum be the set of rules checked in seaport-order-validator.

The client SHOULD revalidate orders in the db on an interval to ensure orders have the latest validity information. It is RECOMMENDED to revalidate orders at least onc every 5 minutes to keep their status fresh. Clients MAY also keep track of token balances to revalidate orders on a more continual basis, rather than explicitly checking individual orders.

The client is RECOMMENDED to remove orders in the db that are permanently unfulfillable: if the order is fully fulfilled, cancelled, or invalid in some other way. Orders that are temporarily invalid due to insufficient balances or approvals MAY be kept and revalidated until the order becomes valid again, or may be removed if it has been invalid for a length of time.

### Wire protocol messages

The client MUST adhere to the following specification for exchanging wire protocol messages.

The examples define SSZ types in TypeScript using the npm package `@chainsafe/ssz`. Note: the `limit` as set in the second parameter for `ListCompositeType` is only used for merklization of the list and thus the values below are only for illustrative purposes of the expected maximums, you may adjust them as needed.

#### Status (0x00)

```ts
const ContractOrders = new ContainerType({
  address: Bytes20,
  validOrderCount: Uint256,
  invalidOrderCount: Uint256,
});
const Status = new ContainerType({
  version: Uint8,
  chainid: Uint256,
  blockHeight: Uint256,
  contracts: new ListCompositeType(ContractOrders, 1000),
});
```

Inform a peer of its current state. This message should be sent just after the connection is established and prior to any other protocol messages.

- `version`: the current protocol version (1 for this specification)
- `chainid`: the chain id identifying the blockchain
  - For a community curated list of chain IDs, see [https://chainid.network](https://chainid.network).
- `blockHeight`: the node's current block height
- `contracts`: order information by contract
  - `address`: the contract address
  - `validOrderCount`: the number of currently valid orders in the db for the contract
  - `invalidOrderCount`: the number of currently invalid orders in the db for the contract

The `contracts` list MUST be in descending order of `validOrderCount` and MAY return a maximum of 100 contracts, even if the node contains more.

#### GetOrders (0x01)

```ts
const OrderHashes = new ContainerType({
  reqId: UintNum64,
  hashes: new ListCompositeType(Bytes32, 1_000_000),
});
```

Request a set of orders by their hashes. If the responding peer does not have an order, it may omit it from the response, and return as many orders as it has information about in its own db.

#### Orders (0x02)

```ts
const Orders = new ContainerType({
  reqId: UintNum64,
  orders: new ListCompositeType(Order, 1_000),
});
```

Return a set of orders by their requested hashes. A responding peer need only return information about orders it has in its own database and should not request the network for missing hashes, as the requesting peer may do that itself.

#### GetOrderHashes (0x03)

```ts
enum OrderSort {
  NEWEST,
  OLDEST,
  ENDING_SOON,
  PRICE_ASC,
  PRICE_DESC,
  RECENTLY_FULFILLED,
  RECENTLY_VALIDATED,
  HIGHEST_LAST_SALE,
}
enum OrderFilter {
  OFFERER_ADDRESS,
  TOKEN_ID,
  BUY_NOW,
  ON_AUCTION,
  SINGLE_ITEM,
  BUNDLES,
  CURRENCY,
}
enum Side {
  BUY,
  SELL,
}
const GetOrdersOpts = new ContainerType({
  side: Uint8,
  count: Uint32,
  offset: Uint32,
  sort: Uint8,
  filter: new ListCompositeType(OrderFilter, 20),
});
const OrderQuery = new ContainerType({
  reqId: UintNum64,
  address: Bytes20,
  opts: GetOrdersOpts,
});
```

Request a set of orders by query criteria. Responds with a list of order hashes, which the individual order data may be requested with GetOrders.

#### OrderHashes (0x04)

```ts
const OrderHashes = new ContainerType({
  reqId: UintNum64,
  hashes: new ListCompositeType(Bytes32, 1_000_000),
});
```

Return a set of orders by their hashes after running a `GetOrderHashes` query.

#### GetCriteria (0x05)

```ts
const GetCriteria = new ContainerType({
  reqId: UintNum64,
  hash: Bytes32,
});
```

Request the list of items in a criteria hash. If the responding peer does not have the criteria in its db, it should respond with an empty list.

#### Criteria (0x06)

```ts
const CriteriaItems = new ContainerType({
  reqId: UintNum64,
  hash: Bytes32,
  items: new ListBasicType(UintBn256, 10_000_000),
});
```

Response to GetCriteria with the list of item IDs in the criteria. If the responding peer does not have the criteria in its db, it should respond with an empty list.

### Gossipsub protocol messages

The client MUST adhere to the following specification for exchanging Gossipsub messages.

Whenever an implementing client receives an event it MUST validate, format and gossip it as a Gossipsub message.

The following enum describes version 1 of this Gossipsub protocol:

```ts
enum OrderEvent {
  FULFILLED,
  CANCELLED,
  VALIDATED,
  INVALIDATED,
  COUNTER_INCREMENTED,
  NEW,
}
```

The topic for a gossipsub message MUST be a token contract address on the order. The client MUST send a Gossipsub message for every token contract address (ERC20, ERC721, ERC1155) on the order.

#### Gossipsub Message ID Function

Each gossipsub message requires an ID to uniquely identify it.

For this specification, the gossipsub message ID function should be the following bytes concatenated together:

`[topic: Bytes20, event: Bytes8, orderHash: Bytes32, blockHash: Bytes32]`

#### Gossipsub Event

Each gossipsub message event must conform to the following:

```ts
const Order = new ContainerType({
  offer: new ListCompositeType(OfferItem, 100),
  consideration: new ListCompositeType(ConsiderationItem, 100),
  offerer: Bytes20,
  signature: new ByteVectorType(65),
  orderType: Uint8,
  startTime: Uint32,
  endTime: Uint32,
  counter: UintBn256,
  salt: UintBn256,
  conduitKey: Bytes32,
  zone: Bytes20,
  zoneHash: Bytes32,
  chainId: UintBn256,

  // Basic Order
  additionalRecipients: new ListCompositeType(Bytes20, 50),

  // Advanced Order
  numerator: UintBn256,
  denominator: UintBn256,
  extraData: Bytes32,
});
const GossipsubEvent = new ContainerType({
  event: Uint8,
  orderHash: Root,
  order: Order,
  blockNumber: UintNum64,
  blockHash: Root,
});
```

The block number and block hash are included for receiving peers to verify the validity of the data.

If the block number and block hash do not match the received data, the client MUST reject the gossipsub message and MUST NOT gossip it further. If the client knows a valid block number and hash for the event, it MAY gossip the valid message.

#### Gossipsub Event Specification

#### Fulfilled

An order has been fulfilled.

#### Cancelled

An order has been cancelled.

#### Validated

An order has been validated and its status is currently valid as per order validation rules.

#### Invalidated

An order has been invalidated and its status is currently invalid and unfulfillable as per order validation rules.

#### Counter Incremented

The counter for an offerer has been incremented, invalidating all orders from the offerer under their new counter.

#### New

A new order has been created.

## Rationale

This specification was designed to be as minimal as possible while providing helpful functionality for relaying Seaport orders. Libp2p was chosen as the basis of the P2P stack as the success of IPFS over the years has led to a hardening of its code, protocols, and cross-platform stack. In the future as alternative P2P stacks mature they can be evaluated for use and supplementary SIPs can be written that require this SIP and extend it to be used on a different stack.

## Backwards Compatibility

As this SIP is the first of its kind there are no issues of backwards compatibility.

## Test Cases

Test cases for this SIP can be found in the [seaport-gossip](https://github.com/ProjectOpenSea/seaport-gossip) repository, which is the first client implementation of this SIP.

## Reference Implementation

A reference implementation for this SIP can be found in the [seaport-gossip](https://github.com/ProjectOpenSea/seaport-gossip) repository, the first client implementation of this SIP.

## Security Considerations

This specification strictly requires checking the validity of peer-gossiped data so bad data cannot propagate throughout the network. Inbound data should always be checked against the appropriate chain for validity to score peers and their messages before passing on messages to other peers.

P2P networks have challenges against spam and poison attacks due to the negligible cost of creating new node identities. IP addresses should be used and enforced when possible to mitigate bad peers.

Order offerers can also drown out other listings by creating many bogus orders that, while valid on chain, take up storage in a node's database and crowds out other valid orders. For this reason, it is recommended that P2P clients implementing this specification put an adjustable limit on how many orders are stored per offerer, so one offerer cannot spam just their own orders. This also makes it more challenging for one attacker to crowd out an orderbook with valid but bogus orders as they would have to give multiple accounts valid balances and approvals.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
