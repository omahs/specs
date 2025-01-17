
# Fees on Vega

Fees are incurred on every trade on Vega. 

An order may cross with more than one other order, creating multiple trades. Each trade incurs a fee which is always non-negative.

## Calculating fees

The trading fee is:

```
total_fee = infrastructure_fee + maker_fee + liquidity_fee`

infrastructure_fee = fee_factor[infrastructure] * trade_value_for_fee_purposes`

maker_fee =  fee_factor[maker]  * trade_value_for_fee_purposes

liquidity_fee = fee_factor[liquidity] * trade_value_for_fee_purposes
```

Fees are calculated and collected in the settlement currency of the market, collected from the general account. Fees are collected first from the trader's account and then margin from account balance. If the general account doesn't have sufficient balance, then the remaining fee amount is collected from the margin account. If this is still insufficient then different rules apply between continuous trading and auctions (details below).

Note that maker_fee = 0 if there is no maker, taker relationship between the trading parties (in particular auctions).

### Factors
- infrastructure: staking/governance system/engine (network wide)
- maker: market framework / market making (network wide)
- liquidity: market making system (per market)

The infrastructure fee factor is set by a network parameter `market.fee.factors.infrastructureFee` and a reasonable default value is `fee_factor[infrastructure] = 0.0005 = 0.05%`. 
The maker fee factor is set by a network parameter `market.fee.factors.makerFee` and a reasonable default value is `fee_factor[maker] = 0.00025 = 0.025%`. 
The liquidity fee factor is set by an auction-like mechanism based on the liquidity provisions committed to the market, see [setting LP fees](./0042-LIQF-setting_fees_and_rewarding_lps.md).

trade_value_for_fee_purposes:
* refers to the amount from which we calculate fee, (e.g. for futures, the trade's notional value = size_of_trade * price_of_trade)
* trade_value_for_fee_purposes is defined on the Product and is a function that may take into account other product parameters 

Initially, for futures, the trade_value_for_fee_purposes = notional value of the trade = size_of_trade * price_of_trade. For other product types, we may want to use something other than the notional value. This is determined by the Product.

NB: size of trade needs to take into account Position Decimal Places specified in the [Market Framework](./0001-MKTF-market_framework.md), and if trade/position sizes are stored as ints will need to divide by `10^PDP` where PDP is the configured number of Position Decimal Places for the market (or this division will need to be abstracted and done global by the position management component of Vega which may expose both a true and an integer position size, or something).

### Collecting and Distributing Fees

We need to calculate the total fee for the transaction.
Attempt to transfer the full fee from the trader into a temporary bucket, one bucket per trade (so we know who the maker is) from the trader general account. 
If insufficient, then take the remainder (possibly full fee) from the margin account.
The margin account should have enough left after paying the fees to cover maintenance level of margin for the trades.
If the transfer fails: 
1) If we are in continuous trading mode, than trades should be discarded, any orders on the book that would have been hit should remain in place with previous remaining size intact and the incoming order should be rejected (not enough fees error). 
This functionality requires to match orders and create trades without changing the state of the order book or passing trades downstream so that the execution of the transaction can be discarded with no impact on the order book if needed. 
Other than the criteria whether to proceed or discard, this is exactly the same functionality required to implement [price monitoring](./0032-PRIM-price_monitoring.md). 
1) If we are in auction mode, ignore the shortfall (and see more details below). 

The transfer of fees must be completed before performing the normal post-trade calculations (MTM Settlement, position resolution etc...). The transfers have to be identifiable as fee transfers and separate for the three components. 

Now distribute funds from the "temporary fee bucket" as follows:
1) Infrastructure_fee is transferred to infrastructure fee pool for that asset. Its distribution is described in [0061 - Proof of Stake rewards](./0061-REWP-pos_rewards.md). In particular, at the end of each epoch the amount due to each validator and delegator is to be calculated and then distributed subject to validator score and type.
1) The maker_fee is transferred to the relevant party. 
1) The liquidity_fee is distributed as described in [this spec](./0042-LIQF-setting_fees_and_rewarding_lps.md). 

### During Continuous Trading

The "aggressor or price taker" of each trade is the participant who submitted / amended the incoming order that caused the trade  (including automatic amendments like pegged orders).

The "aggressor or price taker" pays the fee. The "passive or price maker" party is the participant in the trade whose order was hit (i.e. on the order book prior to the uncrossing that caused this trade)

### Normal Auctions (including market protection and opening auctions)

During normal auctions there is no "price maker" both parties are "takers". Each side in a matched trade should contribute 1/2 of the infrastructure_fee + liquidity_fee. Note that this does not include a maker fee. 

Fees calculated and collected from general + margin as in continuous trading *but* if a party has insufficient capital to cover the trading fee then in auction the trade *still* *goes* *ahead* as long as the margin account should have enough left after paying the fees to cover maintenance level of margin for the orders and then converted trades. The fee is distributed so that the infrastructure_fee is paid first and only then the liquidity_fee. 

During an opening auction of a market, no fees are collected.

### Frequent Batch Auctions

Order that entered the book in the current batch are considered aggressive orders. This means that in some cases both sides of a trade will be aggressors in which case the fee calculation for normal auctions applies. Otherwise, the fee calculation for continuous trading applies.

### Position Resolution

The trades that were netted off against each other during position resolution incur no fees. 
During position resolution all of the parties being liquidated share the total fee for the network order, pro-rated by the size of position. 
As for fees in other cases, the fee is taken out of the general + margin account for the liable traders (the insurance pool is not used to top up fees that cannot be paid). If the general + margin account is insufficient to cover the fee then the fee (or part of it) is not going to get paid. In this case we first pay out the maker_fee (or as much as possible), then then infrastructure_fee (or as much as possible) and finally the liquidity_fee.

### Rounding

All fees are being rounded up (using math.Ceil in most math libraries).
This ensures that any trade in the network will require the party to pay a fee, even in the case that the trade would require a fee smaller than the smallest unit of the asset.
For example, Ether is 18 decimals (wei). The smallest unit, non divisible is 1 wei, so if the fee calculation was to be a fraction of a wei (e.g 0.25 wei), which you cannot represent in this currency, then the Vega network would round it up to 1.

## Acceptance Criteria
- Fees are collected during continuous trading and auction modes and distributed to the appropriate accounts, as described above. (<a name="0029-FEES-001" href="#0029-FEES-001">0029-FEES-001</a>)
- Fees are debited from the general (+ margin if needed) account on any market orders that during continuous trading, the price maker gets the appropriate fee credited to their margin account and the remainder is split between the market making pool and staging pool. (<a name="0029-FEES-002" href="#0029-FEES-002">0029-FEES-002</a>)
- Fees are debited from the general (+ margin if needed) account on the volume that resulted in a trade on any "aggressive / price taking" limit order that executed during continuous trading, the price maker gets the appropriate fee credited to their margin account and the remainder is split between the market making pool and staging pool.  (<a name="0029-FEES-003" href="#0029-FEES-003">0029-FEES-003</a>)
- Fees are debited from the general (+ margin if needed) account on any "aggressive / price taking" pegged order that executed during continuous trading, the price maker gets the appropriate fee credited to their margin account and the remainder is split between the market making pool and staging pool. (<a name="0029-FEES-004" href="#0029-FEES-004">0029-FEES-004</a>)
- Fees are collected in one case of amends: you amend the price so far that it causes an immediate trade.  (<a name="0029-FEES-005" href="#0029-FEES-005">0029-FEES-005</a>)
- During auctions, each side of a trade is debited 1/2 (infrastructure_fee + liquidity_fee) from their general (+ margin if needed) account. The infrastructure_fee fee is credited to the staking pool, the liquidity_fee is credited to the market making pool. (<a name="0029-FEES-006" href="#0029-FEES-006">0029-FEES-006</a>)
- During continuous trading, if a trade is matched and the aggressor / price taker has insufficient balance in their general (+ margin if needed) account, then the trade doesn't execute if maintenance level of trade is not met. (<a name="0029-FEES-007" href="#0029-FEES-007">0029-FEES-007</a>)
- During auctions, if either of the two sides has insufficient balance in their general (+ margin if needed) account, the trade still goes ahead only if :-) the margin account should have enough left after paying the fees to cover maintenance level of margin for the orders and then converted trades. (<a name="0029-FEES-008" href="#0029-FEES-008">0029-FEES-008</a>)
- Changing parameters (via governance votes) does change the fees being collected appropriately even if the market is already running.  (<a name="0029-FEES-009" href="#0029-FEES-009">0029-FEES-009</a>)
- A "buyer_fee" and "seller_fee" are exposed in APIs for every trade, split into the three components (after the trade definitely happened) (<a name="0029-FEES-010" href="#0029-FEES-010">0029-FEES-010</a>)
- Users should be able to understand the breakdown of the fee to the three components (by querying for fee payment transfers by trade ID, this requires enough metadata in the transfer API to see the transfer type and the associated trade.) (<a name="0029-FEES-011" href="#0029-FEES-011">0029-FEES-011</a>)
- The three component fee rates (fee_factor[infrastructure, fee_factor[maker], fee_factor[liquidity] are available via an API such as the market data API or market framework. (<a name="0029-FEES-012" href="#0029-FEES-012">0029-FEES-012</a>)
- A market is set with [Position Decimal Places" (PDP)](0052-FPOS-fractional_orders_positions.md) set to 2. A market order of size 1.23 is placed which is filled at VWAP of 100. We have fee_factor[infrastructure] = 0.001, fee_factor[maker] = 0.002, fee_factor[liquidity] = 0.05. The total fee charged to the party that placed this order is `1.23 x 100 x (0.001 + 0.002 + 0.05) = 6.519` and is correctly transferred to the appropriate accounts / pools. (<a name="0029-FEES-013" href="#0029-FEES-013">0029-FEES-013</a>)   
- A market is set with [Position Decimal Places" (PDP)](0052-FPOS-fractional_orders_positions.md) set to -2. A market order of size 12300 is placed which is filled at VWAP of 0.01. We have fee_factor[infrastructure] = 0.001, fee_factor[maker] = 0.002, fee_factor[liquidity] = 0.05. The total fee charged to the party that placed this order is `12300 x 0.01 x (0.001 + 0.002 + 0.05) = 6.519` and is correctly transferred to the appropriate accounts / pools. (<a name="0029-FEES-014" href="#0029-FEES-014">0029-FEES-014</a>)   
