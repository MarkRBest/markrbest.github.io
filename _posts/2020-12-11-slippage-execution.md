---
layout: post
title:  Slippage Analysis - Part 3
category:
- Algo Trading

---

I ran some  backtests overnight to try to work out if latency is likely an issue. If the slippage issue is due to latency
in production and the simulation is accurate then we would expect to see the same negative effect of latency on pnl in the simulations.

Results from the simulation over the last 30 days.

![table](/assets/2020-12-11/table.png)

Latency has an effect on pnl, but it is not that significant and doesn't explain the amount of slippage
we are seeing in live trading. My deductions from this is the problem is possibly one few things

* An issue with the historical market data used for simulation
   *  I have gone to great lengths to make sure it is of the best quality possible.
    The timestamps between exchanges are synchronised and the data is clean and without gaps.
    I record data myself in multiple locations and have automated mechanisms for comparing the data sets and finding data issues.
* A problem with the simulation environment
    * This is more likely to be a problem but for performance reasons it is quite simple.
    This simplicity has the added benefit of making the code easy to unit test and check for correctness.
* Some other problem with the production execution with the production execution strategy
    * It would thus seem from what I know about the code, it is most likely issue is related to the production execution code.


## Issue

Order Management Code for Tracking Touch

{% highlight python linenos %}

if abs(trade_size) >= tracking_error:
    # orders that are currently active in the market
    active_orders = list(filter(lambda order: order.instrument == instrument, active_orders))
    # orders that are inflight i.e. sent to exchange unconfirmed or cancel unconfirmed
    pending_orders = list(filter(lambda order: order.instrument == instrument, pending_orders))

    # get current best bid/offer
    book = self.books[instrument.id]
    tob_bid = book.tobBid()[0]
    tob_ask = book.tobAsk()[0]
    price = tob_bid if trade_size > 0 else tob_ask

    # create order parameters
    req_basis_order = {"strategy": cfg.strategy, "exchange": instrument.exchange,
                       "instrument": instrument, "otype": OrderType.LIMIT,
                       "price": price, "amount": trade_size,
                       "post_only": True}

    # check if the order already in the market
    if active_orders:
        # we should only have one mm order at a time so cancel any extra orders
        if len(active_orders) > 1:
            self.logger.warning(f"multiple live orders {active_orders}")
            for oidx in range(len(active_orders)):
                ## cancel orders of the wrong direction first
                if np.sign(active_orders[oidx].amount) != np.sign(trade_size):
                    self.order_manager.cancelOrder(active_orders[oidx].order_id)

            # if there are still too many orders then cancel them
            for oidx in range(1, len(active_orders)):
                self.order_manager.cancelOrder(active_orders[oidx].order_id)
        else:
            # cancel order if it doesnt match target price
            for order in active_orders:
                if np.isclose(price, order.price) :
                    self.order_manager.cancelOrder(order.order_id)

    elif not pending_orders:
        # place new passive order if there are none outstanding and we are not waiting for an order confirmation
        self.order_manager.createOrder(**req_basis_order)

else:
    # cancel orders if risk matches target risk
    for order in active_orders:
        if order.instrument == instrument:
            self.order_manager.cancelOrder(order.order_id)

{% endhighlight %}

TLDR; Check if there are no orders and if not place one. If there is an order in the market but the price is different to touch cancel it.

The issue seemed to lie here on line 32

{% highlight python %}
if np.isclose(price, order.price) :
{% endhighlight %}

The code was repurposed from a different strategy and has some odd issues with the order book cleaning code.
Liquidity is cleaned from the limit order book when there is a trade at the touch price.
If the current best bid is 200 contracts @ 18000.05 and there is a sell trade on the public feed of 200 lots for 18000.05.
It is faster to then update the order book and remove the level @ 18000.05 compared with simply waiting for the change to be affirmed by the order book feed.

In exchanges like CME you would expect in these case the order was filled. How can more volume trade on the bid than volume available?
Either there is hidden size in the book which seems unlikely given there is not an
[order type](https://bybit-exchange.github.io/docs/inverse/#t-placeactive) for icebergs or hidden size.
There is the ability to set take profit and stop losses on orders in BYBIT so likely there are conditional orders
being executed that are not shown in the order book.
It might also just be BYBIT been doesn't have synchronisation between the REST order book feed and the trade feed.
In the chicago mercantile exchange (CME) for instance market data messages have monotonically increasing sequence ids
and unless there is packet loss you would expect a trade update and then a book update with the same sequence ids.
The code was actually pulling the order thinking the order book had moved but somehow the order was pulled before we were filled.
I will at some point have to do some more work on the book building but this is something that I have seen before and is a useful microstructure signal especially in times when the market spreads are wide.

The fix was to change line 32 with this.

{% highlight python %}

# EPSILON to avoid floating point issues in the equivalence check
TICK_SIZE = instrument.tick_size - EPSILON

if trade_size > 0:
    # only step bid if price moves up
    if price > order.price + TICK_SIZE:
        self.order_manager.updateOrder(order.order_id, new_price=price)
else:
    # only step ask if price moves down
    if price < order.price - TICK_SIZE:
        self.order_manager.updateOrder(order.order_id, new_price=price)

{% endhighlight %}

The main differences are

* Only move bid orders up and ask orders down
* Use order modify to improve the amount of time we are in the market and limits the risk of overfill.


## Limit Order Book Plots

Limit order book plots are a useful tool. Given a specific point in time the chart shows the limit order books,
strategy orders, exchange trades, fills etc. If there are any fills that look problematic or times the strategy lost money it's easy to isolate the information there.
The limit order books contain a lot of information and when that is extended over time its almost impossible to go through the data line by line.
The LOB chart is a powerful way to collate this data and to make it easier to understand what is going on.

Charts are normally just the start of the investigation, once some hypothesis have been defined they can either be tested
in simulation or experiments be designed to test them in production via A/B testing.

The limit order book charts allow us to drill down on fills which have bad slippage to see what happened.

![lob-plot1](/assets/2020-12-11/lob-plot1.png)

`A buy order with 30 USD of slippage.`

The live execution makes sense because the market never ticked down until around the time order was filled.
It however looks like the paper buy order was placed at an off market price which is why it was filled without the market moving.
The limit order book plot is really useful to understand the actions of the strategy and to investigate any executions of interest.

Next Steps

 * Understand what is going on with the paper fills
 * Collect more live trade data to confirm the effect of the code fix.