---
layout: post
title:  Bitcoin Microstructure
category:
- Microstructure
- Bitcoin
---

I find bitcoin market microstructure very interesting since it is so different to any market I have looked at.
I come from a fixed income HFT background, and a lot of the instruments I have previously traded tend to be deep markets.
The 10Y UST T-Note future (ZN) contract on CME-Globex normally has about 5000 contracts of 100k USD value per contract each price level.
Moving the price even a few ticks in one go requires very deep pockets.
Bitcoin, however, has a very different microstructure in that the touch prices are very deep 2-4 million USD but there is almost no liquidity
at any of the other levels.

I recently improved my limit order plots to add the depth, so thought it would be interesting to share them.

![lob1](/assets/2020-12-14/lob-plot-1.png)

`The offer can be seen to disappear about a second before the price moves.`

![lob2](/assets/2020-12-14/lob-plot-2.png)

`The offer is seen to overshoot on trading activity and revert immediately after`

![lob3](/assets/2020-12-14/lob-plot-3.png)

`The chart is one of the cases where the non-touch levels have comparable volume.`

The depth is now shown per price level. This uses the fill_between feature on plotly.

I have heard about people running 0+ strategies in crypto but I'm not sure they would work since in normal financial markets
the fees of aggressively hedging are not prohibitive. 0+ tries to always be in front of the queue by being the first strategy
to join a new price level. This is the sort of strategy that FPGAs are great for since the logic is as simple as

~~~
if (new_bid > bid)
    join(new_bid)
~~~

This logic can be executed in hardware by what is essentially a network card and thus avoiding the cost of getting the
operating system or any software involved. A fast software solution might have a tick to trade of 3-4 microseconds.
I don't have first-hand experience using FPGAs so don't exactly how fast they are but according to [fpga-in-high-frequency-trading](https://www.velvetech.com/blog/fpga-in-high-frequency-trading/)

> FPGA chips have very specific technical characteristics that enable them to execute certain types of trading algorithms up to 1000 times faster than traditional software solutions.

This would suggest and FPGA can work around single-digit nanoseconds.

My suspicion there are other arbitrages which require passive fills to be profitable due to the fee strucutre.
Conditionally on being filled on one exchange then its highly likely the trade can be completed profitably elsewhere.
This is the only reason I can think of that explains this micro-structure behaviour.
