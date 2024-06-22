---
layout: post
title:  The Hidden Dangers of Writing an OMS
category:
- HFT
- Algo Trading
---

Writing an OMS for an HFT platform is a really difficult task that is often taken for granted.
It is made more difficult in crypto because the exchange infrastructure is unreliable.
It is not uncommon to simply be told "go away and come back later" when trying to call api functions.
There is also often a hidden danger when writing an OMS which I have not seen discussed.
This article is about what can be the most dangerous part of a high-frequency trading platform.

### Code

And here it is...

```python
if abs(oms.position(instrument_id) + order_qty) < POSTION_LIMIT and not oms.has_live_orders() {
    oms.place_order(instrument_id, order_price, order_qty);
}
```

The code is deliberately simplified to highlight the issue. In a real trading system, you would be able to place multiple orders for multiple instruments.
The code checks risk limits and places an order if there is not an order in the market already.
So if it is so simple, what's the issue?

What would happen if an order was placed and filled, but the order update arrives before the position update?

What would happen if the position feed subscription failed and the position no longer updates?


This leads me to the next important point. Risk is the most important metric in any trading system.
You might think it is pnl, but risk impacts the rate of change of the pnl and that is more important.
Not knowing your risk is an easy way to [blow up](https://amzn.eu/d/0NIcEaz).

### What Would This Look Like In Real Life

A much more complicated real life example can be found in the [Bitmex Sample Market Maker](https://github.com/BitMEX/sample-market-maker).
The code generates a ladder of orders and posts them to the exchange.
It has some simple logic to cancel/place orders if the required orders do not match those in the market.
It has one section in [market_maker.py](https://github.com/BitMEX/sample-market-maker/blob/master/market_maker/market_maker.py) where it builds the required orders
if the risk has not been exceeded.
Later it checks if the order exists in the market and if not it posts it.

#### Order creation
```python
for i in reversed(range(1, settings.ORDER_PAIRS + 1)):
    if not self.long_position_limit_exceeded():
        buy_orders.append(self.prepare_order(-i))
    if not self.short_position_limit_exceeded():
        sell_orders.append(self.prepare_order(i))
```

#### Order placement
```python
for order in existing_orders:
    try:
        if order['side'] == 'Buy':
            desired_order = buy_orders[buys_matched]
            buys_matched += 1
        else:
            desired_order = sell_orders[sells_matched]
            sells_matched += 1

        # Found an existing order. Do we need to amend it?
        if desired_order['orderQty'] != order['leavesQty'] or (
                # If price has changed, and the change is more than our RELIST_INTERVAL, amend.
                desired_order['price'] != order['price'] and
                abs((desired_order['price'] / order['price']) - 1) > settings.RELIST_INTERVAL):
            to_amend.append({'orderID': order['orderID'], 'orderQty': order['cumQty'] + desired_order['orderQty'],
                             'price': desired_order['price'], 'side': order['side']})
    except IndexError:
        # Will throw if there isn't a desired order to match. In that case, cancel it.
        to_cancel.append(order)
```

I would like to say, this is not just a dig at this code.
It is a nice solution to managing an order ladder, and I used it back in the day.
I emailed Bitmex about it and the CTO (who was the original creator) got back to me.
They were super helpful and in general, I have a soft spot for the old days of Bitmex.

The market maker would, often have more risk than it should due to it thinking there is no order in the market just after one just got filled.
It would then place another leading to it also getting immediately filled violating risk limits.

### The Impact

For market makers, this is more of an inconvenience risk might be exceeded by one clip or one fill might get a bad price.
If something like this happens to a taker strategy sending IOC orders it can be catastrophic.
Just ending up with a lot of risk is not even the worst possible outcome.
If the strategy buys too much due to a bug like this and then sells its position to clean up the risk, it can get in a loop.
This is known as flip-flopping, and the limit to how much this can lose is only bound by the account size.

It is always good to add sanity-checking logic to an OMS to specifically check for order spamming and flip-flopping to prevent IOC strategies from going postal.

### Solution

Unfortunately, there is not a simple good solution to this problem.
The issue is that you are dealing with knowing the state in an atomic way without slowing things down.
It is difficult to know both the outstanding risk and positions in an atomic fashion.
A simple solution is to flag the position as "dirty" and to wait after there is a fill for the position update.
Needing to wait is not an acceptable solution to an HFT problem.
One key observation is that positions are the summation of fills.
If you keep a running sum of all fills, you will know your position.
This can in practice be a little tricky since due to networking issues order updates can get missed.
So a better solution is to allow a certain discrepancy between the running sum of fills since the last position update.
This in essence will allow for a certain number of fills but will block more orders from being sent if there are a lot of fills without a position update.
On exchanges which have proper sequence numbers, this is simple as you can work out if a fill is included or not in the most recent position update.
If there are no sequence numbers then more care has to be taken. Given the position and recent fills, it is possible to process the fill list until the last position update is found.
This will let you know if all positions have been included.
Then the process can just carry on. This allows the position to have a maximum deviation between updates. If things are working normally there will be zero blocking.
If there is an issue however then the system will wait until the risk can be confirmed.

It seems simple to query the order state or query the positions via API to complement the websocket feed.
This is useful but is often used as a last resort as the exchange makes the cost of using it quite expensive.


### Final Note

The issue above, if not treated with the correct care, can cost the whole account.
Maybe it is an obvious problem. I don't think it is as I often use it as an interview question if anyone says they have worked in or around an OMS.
To be forewarned is to be forearmed, so at least now you know about the devil that may live under your bed.


