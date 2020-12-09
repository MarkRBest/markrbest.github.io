---
layout: post
title:  Paper vs Live Slippage Analysis
category:
- Algo Trading

---

I recently went live with a crypto mean reversion strategy on BYBIT. The reason for choosing to test the strategy on this exchange is mainly that they offer
rebate fees of -0.025% if you are a market maker. This is much more attractive than the fees on many other exchanges. BYBIT in my experience is pretty much a carbon copy of BITMEX
and seem to have a very stable API. BITMEX has it's issues in regard to becoming unresponsive in times of high volumes but for the most part their technology is useable.
I have a good amount of experience of algo trading on crypto exchanges and there are practical issues with strategy implementation on these exchanges relating to both their performance as well as the design of each exchanges API.
BYBIT and BITMEX for all their issues are in my favourite exchanges due to their fee models their API.

## Strategy

I recently completed testing on a new strategy. It is a medium frequency outright strategy that takes positions on various crypto currencies and bets on their reversion.
From simulation the equity chart looks like this.

![trace_plot](/assets/2020-12-09/pnl-chart.jpg)

There are some important points to note.

1. Simulation execution assumes to buy we join best bid and are filled when the ask crosses the bid.
2. The goal of the live trading exercise is mainly to validate the simulations rather than strategy optimisation.
3. Simulation round trip pnl is 0.1% which should be enough to ensure live trading is not too risky even if slippage is higher than expected.

My assumption is that the strategy is profitable and the $$ \alpha $$ is time stable.
My main concern is that due to the strategy using passive execution, slippage is high thus invalidating the simulated result.
Correcting the simulation for slippage is not as simple as just adding a penalty since slippage is often correlated to the trades that do the best since these fills are the competitive.
Imagine seeing a PS5 online for $300, is that stock likely to be available for long?
This is one of the drawbacks to averages. The average slippage might be small but if the pnl is concentrated in a few trades then a large slippage on those trades with disproportionately effect the live results.
This is an effect that will need to be explicitly checked for.

If the simulator is properly encapsulated then it is easy to run it alongside production but swapping the historic price feed with a live price feed.
Its very important that the backtesting software is designed to achieve this with minimal changes. Even with just changes to a live vs historic feed I would expect differences in the simulation and paper results.
There a couple of reasons for this.

* Problem 1
   * The forecast takes about 200 ms to calculate on a new bar thus the in paper and live the orders are sent later than in sim.
   * We can estimate the computation time for the forecast and delay the orders by this much in sim.

* Problem 2
    * Crypto exchanges do not have sequence ids on their feeds and in my experience of running multiple feeds via rest it seems
      that the data in two feeds is different.
    * We could record the market data on the live machine and use this at a later date with simulation. The main reason I don't do this is that I don't want to overload production trading with having to also do a massive amount if IO.

I should note my aim is not to get a perfect simulator as this task takes a lot of effort and
I would like to focus on strategy optimisation as much as possible given time constraints.
Also it should be noted the main reason for choosing to focus on trading medium frequency strategies
was so that I can minimise the impact of execution since it's notoriously difficult to simulate with passive strategies.
This however does not mean execution can be ignored completely.

## Deployment

The deployment of the different environments is shown.

![trace_plot](/assets/2020-12-09/deployment.png)

This gives us three estimates of the performance of the strategy simulation, paper and live.

By assuming that the returns of the strategy are time stable we can assume:

$$ \alpha_t = \alpha \hspace{1cm} \forall t $$

What we would like to test from the results is

$$ pnl_{sim} = \alpha dt + \sigma dW^{sim} $$

$$ pnl_{prod} = \alpha dt + \sigma dW^{prod} $$

It is easy in simulation to test the null hypothesis $$\alpha = 0 $$ what we would like to know is if $$ alpha_{sim} = alpha_{prod} $$ since if $$ alpha_{sim} = alpha_{prod}  \land  alpha_{sim} > 0 \implies alpha_{prod} > 0 $$.
If the execution is the same between simulation and production then this should be the case. At that point the only concern is the time stability of alpha.
What is mean but this is to make sure that just because we have historically made money doesnt mean that we will make money in the future.
This is a different topic that many books ([deprado][deprado], [masters][masters-book]) have been written about and something i plan to post more about at a later date.

## Results

| Environment | Buy Count | Buy Qty | Avg Sell Price | Sell Count | Sell Qty | Avg Buy Price |
| :--------| :-------------: | :---- | : ---- |
| Sim | 23 | 3726 | $18218.10 | 28 | 4115  | $18202.61 |
| Live | 23 | 3726 | $18220.39 | 28 | 4115  | $18204.24 |
| :--------| :-------------: | :---- | : ---- |
|  |  | $ 2.29 | | $-1.63    |

The volume weighted slippage is around 0.000557 % which is a lot lower than I was expecting and is actually even positive.

Positive slippage means that live out performed paper. This is not that surprising given the fact the simulation only fills orders by the market price crossing it.
In reality, orders can be filled at a better price in wider market by being aggressed without the bid offer price changing.
The current implementation is designed to be fast and simple. I mainly wanted to make sure slippage was no worse than the BYBIT rebate of 0.025% and definitely no worse than simulation profit per trade of 0.1%

## Note

In my opinion it's better to go live with a model as soon as possible. It is easy to spend a huge amount of time perfecting a model in sim, adding parameters and finding solutions to edge conditions only to find
when its traded live it does not make money. There are various reason for this related to data quality, simulated execution assumptions, latency, exchange outages etc.
The sooner its possible to get live results and to validate simulations the better. Doing this massively increases the future confidence in any pnl improvement from optimisations.


[deprado]: https://read.amazon.co.uk/kp/embed?asin=B079KLDW21&preview=newtab&linkCode=kpe&ref_=cm_sw_r_kb_dp_wLk0FbAJE9SSA
[masters-book]:https://read.amazon.co.uk/kp/embed?asin=B07JVKW1BT&preview=newtab&linkCode=kpe&ref_=cm_sw_r_kb_dp_BNk0FbAX8DBXZ
