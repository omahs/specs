# Market decimal places

This document aims to outline how we are to handle the decimal places of a given market, if said market is configured with fewer decimal places than its settlement asset. For example: a market settling in ETH can be configured to use only 9 decimal places (GWei) compared to ETH's 18. A market cannot specify _more_ decimal places than its settlement asset supports.

## Terminology

* Settlement asset: the asset in which transactions for a given market are made (margin balances, fees, settlements, etc...).
* Market precision: the number of decimal places a market uses (as mentioned previously, a market where the smallest unit of ETH is a GWei has a 9 decimal places, so the market precision is 9). Synonymous with _market tick_.
* Asset precision: the number of decimal places for a given asset. Again, a market with precision 9 that settles in ETH will have a market precision of 9, whereas the asset precision is 18.

## Mechanics

It is possible to configure a market where orders can only be priced in increments of a specific size. This is done by specifying a different (smaller) number of decimal places than its settlement asset supports. Simply put: a market that settles in GBP can be configured to have 0 decimal places, in which case the price levels on the orderbook will be at least separated by £1, rather than the default penny.

This effectively means that prices of submitted orders should be treated as a value that is an order of magnitude greater than what the user will submit. This is trivial to calculate, and is done when the market is created by passing in the asset details (which specify how many decimal places any given asset supports):

```
priceExponent = 10**(asset_precision - market_precision)
// for GBP markets with 0 decimal places this is:
priceExponent = 10 ** (2 - 0) == 100
// for ETH markets with 9 decimal places, this is:
priceExponent = 10 ** (18 -9) == 1,000,000,000
```

When an order is submitted, amended, or otherwise updated, the core emits an event for the data-node (and any other interested parties). The price in this order event should still be represented as a value in market precision. Updating the price on the order internally, for reasons we shall elaborate on later, should not effect the market data on the event bus. To clarify:

```
// given market decimal places == 0, settlement precision == 2
SubmitOrder(Order{
    Size: 10,
    Price: 1,
})
// internally:

order.Price *= market.priceExponent // ie multiply by 100
broker.Send(events.NewOrderEvent(ctx, order)) // should still emit the event where the order is priced at 1
```

In short, market related events should specify prices in the _"unit"_ the market uses.

### Benefits

By allowing markets to specify their own precision, the price levels can be more closely controlled, and any changes in the mark price could be made as _"significant"_ as we want. By converting the prices internally to asset precision, we are able to accurately calculate fees and margin levels based on the asset precision. By operating a market with a lower precision than the asset itself, fees and margin requirements can be calculated with a higher level of precision.

## Changes required

This change has an impact throughout the system, but it can broadly be broken down in the following parts

### Orders

Order submissions and amendments are received, the submitted price (aka market or original price) is copied to a private field, so the events can be constructed in an accurate way. Before submitting the order to the book, or amending it, the price we received from the chain is multiplied by the price exponent. From that point on, the core will operate on the value that has the asset precision. Calculating fees and margin requirements, updating/creating the price levels, etc... all happens in asset precision. Any order events the core sends out will look exactly the same as before.

### Pegged orders

When (re-)pricing pegged orders, the offset values are multiplied by the same price factor before we add/subtract them from the reference price. The offsets themselves (as in: the field on the order object) is not updated. Events that contain this data, therefore, will still look exactly the same as they do now.


### Liquidity provisions

Orders created for an LP work pretty much exactly the same as pegged orders. The offsets will, again, be multiplied by the price exponent when the price is calculated, but the LP shape object is not updated.
When repricing LP orders, we ensure the price of the orders fall inside the upper and lower price bounds. These values are going to be calculated based on the prices used internally (in asset precision). So as to not create orders at a price point that is more precise than the market is configured to support, we floor the max price bound, and ceil the minimum price, as those values will effectively be the max and min prices that are allowed without trades resulting in an auction being triggered.

### Market data

The market data returns the same min/max prices mentioned above. As the name implies, _"MarketData"_ is clearly market related data... The min/max price values we return from this call should therefore be floored (max) and ceiled (min) in the same way.


### Trades

Trades of course result in transfers. The amounts transferred (for the trade as well as the MTM) happen at asset precision. The trade events the core sends out, however, are once again market related data. The prices on these trade events will be represented as a value in market precision.

## Acceptance criteria


- As a user, I can propose a market with a different precision than its settlement asset
  - This proposal is valid if the precision is NOT greater than the settlement asset (<a name="0070-MKTD-001" href="#0070-MKTD-001">0070-MKTD-001</a>)
  - This proposal is NOT valid if the precision is greater than the settlement asset (<a name="0070-MKTD-002" href="#0070-MKTD-002">0070-MKTD-002</a>)
- As a user all orders placed (either directly or through LP) are shown in events with prices in market precision (<a name="0070-MKTD-003" href="#0070-MKTD-003">0070-MKTD-003</a>)
- As a user all transfers (margin top-up, release, MTM settlement) are calculated and communicated (via events) in asset precision (<a name="0070-MKTD-004" href="#0070-MKTD-004">0070-MKTD-004</a>)
- As a user I should see the market data prices using market precision. (<a name="0070-MKTD-005" href="#0070-MKTD-005">0070-MKTD-005</a>)
- Price bounds are calculated in asset precision, but enforced rounded to the closest value in market precision in range (<a name="0070-MKTD-006" href="#0070-MKTD-006">0070-MKTD-006</a>)
- As a user, offsets specified in pegged orders and LP shapes represent the smallest value according to the market precision (<a name="0070-MKTD-007" href="#0070-MKTD-007">0070-MKTD-007</a>)
- Trades prices, like orders, are shown in market precision. The transfers and margin requirements are in asset precision. ( <a name="0070-MKTD-008" href="#0070-MKTD-008">0070-MKTD-008</a>)
