Feature name: Pegged and Market Maker Orders
Start date: 2020-02-10
Specification PR: https://github.com/vegaprotocol/product/pull/262

# Pegged and Market Maker orders

## Acceptance Criteria
* Pegged orders can only be LIMIT orders, all other types are rejected
* Pegged orders can only be GTT, GTC, IOC and FOK orders
* A markets ability to handle pegged orders is set at creation time
* Pegged orders are removed from the order book when going into an auction and are parked.
* Parked orders are returned to the order book once continuous trading is resumed
* Pegged orders are repriced when their reference price moves
* Cancelling a pegged order removes it from the book and the pegged/parked slice
* An expired pegged order is removed from the book and the pegged/parked slice

## Summary

Market Makers and some other market participants are interested in maintaining limit orders on the order book that are a defined distance from a reference price (i.e. best bid, mid and best offer/ask) rather than at a specific limit price. In addition to being impossible to achieve perfectly through simple Amend commands, this method also creates many additional transactions. These problems are enough of an issue for centralised exchanges that many implement pegged orders, which are automatically repriced when the reference price moves. For decentralised trading with greater constraints on throughput and potentially orders of magnitude higher latency, pegged orders are all but essential to maintain a healthy and liquid order book.

Pegged orders are limit orders where the price is specified of the form `REFERENCE +/- OFFSET`, therefore 'pegged' is a _price type_, and can be used for any limit order that is valid during continuous trading. A pegged order's price is calculated from the value of the reference price on entry to the order book. Pegged orders that are persistent will be repriced, losing time priority, _after processing any event_ which causes the `REFERENCE` price to change. Pegged orders are not permitted in some trading period types, most notably auctions, and pegged orders that are on the book at the start of such a period will be parked (moved to a separate off-book area) in time priority until they are cancelled or expire, or the market enters a period that allows pegs, in which case they are re-priced and added back to the order book. Pegged orders entered during a period that does not accept them will be added to the parked area. Pegged orders submitted to a market with a main trading mode that does not support pegged orders will be rejected.

Marker Maker orders are a special order type that must be used by Market Makers to fulfil their liqudity provision commitments. Market maker orders consist of a set of peg instructions and sizes which can be used to distribute liquidity over the order book at various distances from the current BBO (Best Bid / Offer). When entered, the total size of a market maker order provide equal to or greater than the amount of liquidity required by their commitment. Where the **probability density function** changes so as to cause a Market Maker's commitment to become umnet, an additional level of 'virtual' volume will be added to the worst priced order on each side of the book as needed to meet the requirements. This virtual volume will scale up and down to exactly meet the requirements, but the size specified in the Market Maker order will never be reduced even if the Market Maker is providing more liqudiity than requird **OR** the Market Maker may be in breach of their requirement until they amend the order of the requirement changes. After _fully_ processing an event (incoming transaction) that causes a Market Maker order to trade, the order will be refreshed, that is, the remaining size at each price level will be returned to the original size specified in the order, assuming the Market Maker has sufficient collateral to meet the margin requirements of the refreshed order along with their updated position. 


## Guide-level explanation

### Reference Price
This is the price against which the final order priced is calculated. Possible options are best bid/ask and mid price. 

### Offset
This is a value added to the reference price. It can be negative.

When a party submits a new pegged order, only a LIMIT order is accepted. The party also specifies the reference price to which the order will be priced along with an offset to apply to this price. The reference price is looked up from the live market and the final price is calcuated and used to insert the new order. If the price would result in a hit, the order is executed in the same way as a normal LIMIT order. If the price does not hit, the order is placed on the book at the back of the calculated price level.

Whenever the market changes the price uses as a reference any orders which use that reference price need to be repriced. We run through a time sorted list of all the pegged orders and remove each order from the book, recalculate it's price and then reinsert it into the orderbook at the back of the price queue. Following a price move margin checks take place on the positions of the parties. If a p[egged order is to be inserted at a price level that does not currently exist, that price level is created. Likewise if a pegged order is the only order at a price level and it removed, the price level is removed as well.

Pegged orders can be GTC, GTC, IOC or FOK TIF orders. This means they might never land on the book or they can hit the book and be cancelled at any time and in the case of GTT they can expire and be removed from the book in the same way that normal GTT orders can.

If the reference point moves to such a value that it would create an invalid order once the offset was applied, the pegged order is parked. As the reference price moves, any orders on the parked list will be evaluated to see if they can come back into the order book.

When a pegged order is removed from the book due to cancelling, expiring or filling. The order details is removed from the pegged/parked orders list.

# Reference-level explanation

Pegged orders are restricted in what values can be used when they are created, these can be defined by a list of rules each order must abid with.
* Buy orders must be pegged against best bid or mid and the offset must be negative (<0)
* Sell order must be pegged agaisnt best ask or mid and the offset must be positive (>0)


# Pseudo-code / Examples
Each market has a slice containing all the pegged orders. New pegged orders are added to the end of the slice to maintain time ordering.

    PeggedOrder{
        PeggedType type
        OrderID    orderID
    }
    PeggedOrders []PeggedOrder

When a reference price is changed we scan through the pegged orders to update them

    for each item in the PeggedOrders slice
    { 
        if type is equal to the reference price change type
        {
            Remove order from the orderbook
            Update the order price
            Insert the order back into the orderbook at the back of the new price level
        }
    }

Extra functionality will be added to the expiring and cancelling steps

    for each order to cancel/expire
    {
        cancel/expire the order
        if order is pegged
        {
            remove order details from pegged list
        }
        if order is parked
        {
            remove order from parked list
        }
    }


# Test cases
Some plain text walkthroughs of some scenarios that would prove that the implementation correctly follows this specification.
* Insert a pegged order using all of the available reference types and an offset to make the order persistent
* Insert a pegged order using all of the available reference types and an offset to make the order fill
* Insert a pegged order using all of the available reference types and an offset to make the order partially fill
* Insert a pegged order with TIF=GTT and let the order expire while still on the book
* Insert all the pegged order types and cancel them
* Insert a pegged order with a large negative offset and drive the price low to make the pegged price <= 0, verify that the order is parked. Move the price higher and verify that the order is unparked.
* Try to submit valid pegged orders during auction.
* Switch a market to auction and make sure the pegged orders are parked.
* Switch a market from auction to continous trading to make sure the orders are unparked
* Try to insert non LIMIT orders and make sure they are rejected