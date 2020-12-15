---
layout: post
title:  Volatility and Time of Day
category:
- Volatility
---

Given transaction costs are non-zero and in funding, rates are around 10 bps per 8 hours (depending on the exchange) it is
a good idea to trade around times when the market was busy. I have done this analysis on other markets but have not applied it to cryptos.
I was interested to know if volatility tends to cluster around certain days or hours. Before doing the analysis I had a suspicion that there tended to be
a lot of activity over the weekend and also during Asian hours.

## Volatility

![volatility-plot](/assets/2020-12-15/box-plot-hour.png)

![volatility-weekday-plot](/assets/2020-12-15/box-plot-day.png)

The plots are the range of 1H bars in percent for the last 3 months, the timestamps are all UTC.

It seems that my initial suspicion was incorrect and in fact, the busiest days are Tuesday to Thursday
It also seems that most of the volatility is when Europe and the US are awake and seemingly not Asian times.
There is more economic news around these times so it would suggest now there are BTC futures on CME that bitcoin is more integrated into the wider financial system. It seems like the first bar of the day is also quite volatile which I can only assume is people trading the daily time frames.

## Returns

![returns-hour-plot](/assets/2020-12-15/returns-hour.png)

![returns-weekday-plot](/assets/2020-12-15/returns-day.png)

Returns are in USD per hour.

It doesn't look like there is much of a relationship between returns and day other than Wednesday being more volatile.
Returns at different times of day seem to correlate to the times when the market is volatile which is no real surprise.

I didn't expect any predictive association from the pnl analysis, I was just interested to see if there was any association between volatility and pnl.
Given I am market making it is not surprising that there is.
