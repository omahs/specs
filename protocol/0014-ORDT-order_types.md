# Order types

## Summary

A market using a limit order book will permit orders of various types to be submitted depending on the market's current *trading mode* (see [Market Framework](./0001-MKTF-market_framework.md)). This specification encompasses multiple configurable aspects of an order including: triggering, time in force, price type, and execution constraints. It defines the allowable values for each, valid combinations of these, and their behaviour.

Notes on scope of current version of this spec:
- Includes only detailed specification for orders valid for *continuous trading*, does not specify behaviour of these order types in an auction.
- Does not include detailed specification for **stop** orders. Inclusion of stops in guide level explanation is a placeholder/indicator of future requirements.


## Guide-level explanation

### Order types:
1. **Immediate:** order is evaluated immediately (this is the default)
2. **Stop:** order is only evaluated if and when the _stop price_ is reached 
3. **Network:** an order triggered by the network. See [Network Orders](#network-orders)

### Order pricing methods:

*Price type and associated data if required (i.e. limit price, peg reference and offset) must be explicitly provided as one of the below three options and required data, there is no default.*

1. **Limit (+ limit price):** the order is priced with a static limit price, which is the worst price (i.e. highest buy price / lowest sell price) at which the order can trade. If the order has a persistent validity type it will remain on the order book until it fully executes, expires (as defined by the specific validity type), or is cancelled. 
1. **Pegged (+ reference, price offset):** the order is priced relative to a reference price in the market (i.e. best bid, mid, or best offer price) and is automatically repriced (losing time priority) when the reference price changes. Execution is as for a limit order at that price, including on entry and repricing. The order is removed from the book and 'parked' (in entry time priority) if the reference price is undefined, including during an auction. See the [Pegged Orders](./0037-OPEG-pegged_orders.md) spec for more detail.
1. **Market:** the order is not priced and will take volume at any price (i.e. equivalent to a zero priced sell order or an infinitely priced buy order). Only valid on non-persistent validity types.

### Time in Force / validity:

*Time in force must be explicitly provided, there is no default.*

 - **Persistent:**
	1. **Good 'Til Time (GTT):** order is valid until the supplied expiry time, which may be supplied either as an absolute date/time or a relative offset from the  timestamp on the order (i.e. the timestamp added by the core when it receives the order, which is deterministically the same on all nodes)
	1. **Good 'Til Cancelled (GTC):** order is valid indefinitely. 
- **Non-persistent:**
	1. **Immediate Or Cancel (IOC):** an order that trades as much of its volume as possible with passive orders already on the order book (assuming it is crossed with them) and then stops execution. It is never placed on the book even if it is not completely filled immediately, instead it is stopped/cancelled.
	1. **Fill Or Kill (FOK):** an order that either trades all of its volume immediately on entry or is stopped/cancelled immediately without trading anything. That is, unless the order can be completely filled immediately, it does not trade at all. It is never placed on the book, even if it does not trade.
- **Market-state:**
	1. **Good For Auction (GFA):** This order will only be accepted by the system if it arrives during an auction period, otherwise it will be rejected. The order can act like either a GTC or GTT order depending on whether the expiresAt field is set.
	1. **Good For Normal (GFN):** This order will only be accepted by the system if it arrived during normal trading, otherwise it will be rejected. Normal trading is defined as either continuous trading on a normal market or auction trading in a frequent batch auction market. The order can act like either a GTC or GTT order depending on whether the expiresAt field is set.


### Valid order entry combinations

##### Continuous trading

| Pricing method | GTT | GTC | IOC | FOK | GFA | GFN |
| -------------- |:---:|:---:|:---:|:---:|:---:|:---:|
| Limit          | Y   | Y   | Y*  | Y*  | N   | Y   |
| Pegged         | Y   | Y   | N** | N** | N   | Y   |
| Market         | N   | N   | Y   | Y   | N   | N   |

\* IOC/FOK LIMIT orders never rest on the book, if they do not match immediately they are cancelled/stopped.<br>
\** IOC/FOK PEGGED orders are not currently supported as they will always result in the cancelled/stopped state. This may change in the future if pegged orders are allowed to have negative offsets that can result in an immediate match.


##### Auction

| Pricing method | GTT | GTC | IOC | FOK | GFA | GFN |
| -------------- |:---:|:---:|:---:|:----|:---:|:---:|
| Limit          | Y   | Y   | N   | N   | Y   | N   |
| Pegged         | Y*  | Y*  | N   | N   | Y*  | N   |
| Market         | N   | N   | N   | N   | N   | N   |

\* Pegged orders will be [parked](./0024-OSTA-order_status.md#parked-orders) if placed during [an auction](./0026-AUCT-auctions.md), with time priority preserved. See also [0024-OSTA](./0024-OSTA-order_status.md#parked-orders)<br>

### Network orders
Network orders are used during [position resolution](./0012-POSR-position_resolution.md#position-resolution-algorithm). Network orders are orders triggered by Vega to close out positions for distressed traders. 
* Network orders have a counterparty of `Network`
* Network orders are a Fill Or Kill, Market orders
* Network orders cannot be submitted by any party, they are created during transaction processing.

# Acceptance Critieria
- Immediate orders, continuous trading:
	- An aggressive persistent (GTT, GTC) limit order that is not crossed with the order book is included on the order book at limit order price at the back of the queue of orders at that price. No trades are generated. (<a name="0014-ORDT-001" href="#0014-ORDT-001">0014-ORDT-001</a>)
	- An aggressive persistent (GTT, GTC) limit order that crosses with trades >= to its volume is filled completely and does not appear on the order book or in the order book volume. Trades are atomically generated for the full volume. (<a name="0014-ORDT-002" href="#0014-ORDT-002">0014-ORDT-002</a>)
	- An aggressive persistent (GTT, GTC) limit order that is partially filled generates trades commensurate with the filled volume. The remaining volume is placed on the order book at the limit order price, at the back of the queue of orders at that price. (<a name="0014-ORDT-003" href="#0014-ORDT-003">0014-ORDT-003</a>)
	- Any GTT limit order that [still] resides on the order book at its expiry time is cancelled and removed from the book before any events are processed that rely on its being present on the book, including any calculation that incorporate its volume and/or price level. (<a name="0014-ORDT-004" href="#0014-ORDT-004">0014-ORDT-004</a>)
	- A GTT order submitted at a time >= its expiry time is rejected. (<a name="0014-ORDT-005" href="#0014-ORDT-005">0014-ORDT-005</a>)
- No party can submit a [network order type](#network-orders)  (<a name="0014-ORDT-006" href="#0014-ORDT-006">0014-ORDT-006</a>)
# See also
- [0068-MATC-Matching engine](./0068-MATC-matching_engine.md)
