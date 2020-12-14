---
layout: post
title:  Slippage Analysis - Part 4
category:
- Algo Trading
- Crypto
---

The slippage saga is thankfully coming to an end.
Even after all my best efforts, it seems that the issue was related to something I had not foreseen.
The experience has helped confirm the use of this blog as a [cardboard chewbacca](/joel-spolsky/).
During research about hidden size on BYBIT and book cleaning, I was reading the API change logs and noticed that the
open-api was in the process of being deprecated.
I didn't think too much of it other than the need to upgrade due to the old endpoints being decommissioned.

Then I looked at the results...

## Latency Results

![open](/assets/2020-12-12/open-box-plot.png)

These were the latency results using the open api from a machine with 7 ms ping to the gateway.
The average trade latency was 180 ms but there were very long tails. One order took 7 seconds which is not on the
chart since it was such a massive outlier, it made seeing the core of the distribution difficult.

![v2](/assets/2020-12-12/insert-modify-latency.png)

This code uses the v2 api and modifies orders rather than cancel replacing them (hence no cancellation latency).
Using the new api has made a massive difference not only to the average exchange response time but to the tails of the latency distribution.
The mean insert latency is now 16 ms, but the worst case was only 40 ms. The other benefit of using modify orders is that its latency is lower 
than using an cancel insert. The modify latency distribution has a longer upper tail however the need for modifies is 
conditional on the market moving. So every order has an insert but only order in markets moving against the order 
require modifies. Thus it is not surprising the tail of the modify distribution to be higher. 

## Slippage Results

This has also made a marked improvement on the overall slippage numbers.

~~~
buy paper avg price 18022.96
buy live avg price 18026.33

sell paper avg price 18159.69
sell live avg price 18159.70

total volume 16040 USD  0.8852 BTC
dollar slippage -2.5025 USD
slippage  -0.0138 %
~~~

The slippage numbers are now within my original expectations (I thought it would be about the BYBIT rebate of 25 bps).
Likely the true slippage is higher in volatile markets but I have much more confidence now the live platform is working as expected.

# LOB plots

Looking at the limit order book of paper and production fills they now line up closely.

![lob](/assets/2020-12-12/lob-fill.png)

## Conclusion

* The positive takeaway is that the simulator and paper trading seem to be doing their job.
* The api changes and move to a remote server have made a 15x improvement in average latency and a 50x improvement over the 99% tail latency.
* I need to make a note to review the exchange API change logs more often.
* Slippage is now a believable amount and within an acceptable level.

What I thought would be a couple of days of work turned out to be a week, but I am glad there was a happy ending.


## Next Steps

* plot charts of a longer trading session between sim, paper and live to compare.
