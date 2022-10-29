---
layout: post
title:  Building candlesticks in Rust
category:
- Programming
- Deprado
- Algo Trading
- HFT
- Rust
---

Candlesticks are a common way to represent price and volume of an asset over a period of time.
There are various common types of bars such as time, volume, tick bars, hieken-ashi, renko to name a few.
There is a lot of information about the implementations of these on the internet so their details will not be covered here.
The aim of this article is to share some tips for implementation and also a solution to the implementation of Deprdo's volume imbalance bars from the Famous book "Advances in financial machine learning".

This article assumes that the reader is familiar with what a candlestick is,
and the common types of methods of building candlesticks.
If not then this [Alternative Candlestick Types](https://www.tradingsetupsreview.com/trading-charts-without-time-range-tick-volume/) covers the basics.

There is also the question of why use volume bars or tick bars over time bars. The general idea is that due to volatility clustering
asset returns are [heteroskedastic](https://en.wikipedia.org/wiki/Homoscedasticity_and_heteroscedasticity) so the alternative candlesticks
aim to sample by information rate as opposed to time. This is a similar idea to stochastic volatility models such as the
[Heston Model](https://en.wikipedia.org/wiki/Heston_model) in which the process for the volatility is separately modelled and then just assumes everything else is normal.

This article covers a lot of the information regarding not only these alternative bars but the impact on the statistical properties of the resulting return series [Bars](http://www.sefidian.com/2021/06/12/introduction-to-advanced-candlesticks-in-finance-tick-bars-dollar-bars-volume-bars-and-imbalance-bars/).
The TLDR; is kurtosis is bad. You don't want to find out the trend has changed, but by waiting for a new candle stick close
the position is 20% underwater.

Let's start by defining a bar

```rust
#[derive(Debug, Clone, Serialize)]
pub struct Bar {
    pub instrument_id: u32,        // instrument id
    pub open: f64,                 // open price
    pub high: f64,                 // high price
    pub low: f64,                  // low price
    pub close: f64,                // close price
    pub buy_volume: f64,           // total buy volume
    pub sell_volume: f64,          // total sell volume
    pub vwap_price: f64,           // volume weighted average price for bar
    pub open_ts: u64,              // timestamp in ns
    pub close_ts: u64,             // timestamp in ns
}


```
There a couple of extra variables of importance compared with OHLCV bars.
Volume should be signed since there is information in the imbalance between buys and sells.
Also volume weighted average price (vwap) can be tracked over the period since close-vwap contains information about momentum/commitment.

# Time Bars

define the BarBuilder trait ( i.e. interface).
```rust
pub trait BarBuilder {
    fn trade_update(&mut self, timestamp: &u64, price: &f64, qty: &f64) -> Option<Bar>;
}
```

Define TimeBarBuilder class. Notice in rust it doesn't implement the BarBuilder interface.
```rust
pub struct TimeBarBuilder {
    time_window: u64,
    next_bar_time: u64,
    bar: Bar,
}
```
Constructor implementation
```rust
impl TimeBarBuilder {
    pub fn new(instrument_id: u32, time_window: u64) -> Self {
        TimeBarBuilder {
            time_window,
            next_bar_time: 0,
            bar: Bar::new(instrument_id),
        }
    }
}
```
Implement trade_update trait for the TimeBarBuilder
```rust
impl BarBuilder for TimeBarBuilder {
    fn trade_update(&mut self, timestamp: &u64, price: &f64, qty: &f64) -> Option<Bar> {
        let mut res: Option<Bar> = None;

        // check if there is a new bar
        if *timestamp > self.next_bar_time {
            // set the next time a bar should publish
            let rem: u64 = *timestamp % self.time_window;
            self.next_bar_time = *timestamp - rem + self.time_window;

            self.bar.close = *price;
            self.bar.close_ts = *timestamp;

            // compute the volume weighted average price for the bar
            self.bar.vwap_price /= self.bar.buy_volume - self.bar.sell_volume;

            if self.bar.open_ts != 0 {
                // take a copy of the bar information
                res = Some(self.bar.clone());
            }

            // reset the tracking values
            self.bar.reset(price, timestamp);
        }

        // update high/low prices
        if *price > self.bar.high { self.bar.high = *price; }
        else if *price < self.bar.low { self.bar.low = *price; }

        // update the vwap and quantity information
        self.bar.vwap_price += *price * (*qty).abs();
        if *qty > 0.0 { self.bar.buy_volume += *qty; }
        else { self.bar.sell_volume += *qty; }

        return res;
    }
}

```

# Volume bars

Define VolumeBarBuilder class

```rust
pub struct VolumeBarBuilder {
    volume_threshold: f64,
    bar: Bar,
}
```
Constructor implementation
```rust
impl VolumeBarBuilder {
    pub fn new(instrument_id: u32, volume_threshold: f64) -> Self {
        VolumeBarBuilder {
            volume_threshold,
            bar: Bar::new(instrument_id),
        }
    }
}
```
Implement trade_update trait for the VolumeBarBuilder
```rust
impl BarBuilder for VolumeBarBuilder {
    fn trade_update(&mut self, timestamp: &u64, price: &f64, qty: &f64) -> Option<Bar> {
        let mut res: Option<Bar> = None;

        // a new bar is created when the volume threshold is crossed
        if (self.bar.buy_volume - self.bar.sell_volume) > self.volume_threshold {
            self.bar.close = *price;
            self.bar.close_ts = *timestamp;

            // compute the volume weighted average price for the bar
            self.bar.vwap_price /= self.bar.buy_volume - self.bar.sell_volume;

            if self.bar.open_ts != 0 {
                // take a copy of the bar information
                res = Some(self.bar.clone());
            }

            // reset the tracking values
            self.bar.reset(price, timestamp);
        }

        // update high/low prices
        if *price > self.bar.high { self.bar.high = *price; }
        else if *price < self.bar.low { self.bar.low = *price; }

        // update the vwap and quantity information
        self.bar.vwap_price += *price * (*qty).abs();
        if *qty > 0.0 { self.bar.buy_volume += *qty; }
        else { self.bar.sell_volume += *qty; }

        return res;
    }
}
```

### Volume Time Bars

What is the correct threshold for a volume bar builder? Rather than set it to a single value it might be convenient to set it to a dynamic value
based on the volume of recent X time bars. So a 5m volume time bar would create a new bar when the volume is larger than
the recent average of 5 minute bars. This helps in two ways, one is that when working with different assets you don't need to specify
values for each individual asset, the second is that the average then naturally includes measure of seasonality.
The aim of volume bars is to sample more frequently when "something unusual is happening" so it's useful to have a benchmark of normal.
It would be possible to go further in regard to seasonality modelling and if you are interested there is a review of using spline models
to build seasonality models in my [Masters Thesis](/assets/2021-02-09/MarkBestThesis.pdf).

This idea can be used to solve some hyper parameter problem associated with Marcos Lopez de Prado's volume imbalance bars.
The idea of a volume imbalance bar is that a new bar should be created if either a volume threshold is passed or
the ratio of buys to sells exceeds a threshold. The implementation of this can be tricky since the thresholds need to be set.
If the thresholds are calculated using the generated bars they can have degenerate behaviour where expectations tend towards zero or infinity.


My solution to this is to set the imbalance threshold to be a % of the expected volume so if the volume is greater than x or
the imbalance threshold is 20% then a new bar will happen if the imbalance exceeds 0.2 * x. This means only two parameter are needed are the
time aggregation for the time builder and the imbalance threshold percent.
Since the threshold is not a function of the generated imbalance bars the system does not have issues with the expectations tending to zero or infinity.

The following is the implementation of a dynamic volume imbalance bar builder.

```rust
// implementation of a volume imbalance builder

pub struct DynVolumeImbBarBuilder<T: BarBuilder> {
    imb_threshold: f64,    // imbalance threashold [0,1]
    exp_volume: f64,       // volume threshold
    exp_imbalance: f64,    // imbalance threshold
    builder: T,            // bar bulider for generating samples for threshold
    bar: Bar,              // storage
    volume_info: Sma,      // moving average calulating recent volume
}
```
note the `<T: BarBuilder>` which means the builder uses another builder to be its "clock".
Normally a time bar is used but other types of bar could also be used such a range bar.
This would give a new bar when an amount of volume trades that would normally move the market by x%.

```rust

impl<T: BarBuilder> DynVolumeBarBuilder<T> {
    pub fn new(instrument_id: u32, builder: T, imb_threshold: f64, vol_halflife: u16) -> Self {
        debug_assert!(0.0 <= imb_threshold && imb_threshold <= 1.0);

        DynVolumeImbBarBuilder {
            imb_threshold,
            exp_volume: 0.0,
            exp_imbalance: 0.0,
            builder,
            bar: Bar::new(instrument_id),
            volume_info: Sma::new(vol_halflife),
        }
    }

    pub fn trade_update(&mut self, timestamp: &u64, price: &f64, qty: &f64) -> Option<Bar> {
        let mut res: Option<Bar> = None;

        // call the driver bar builder to estimate the expected volume per bar
        match self.builder.trade_update(timestamp, price, qty) {
            Some(bar) => {
                if !is_close(bar.high, bar.low) {
                    let volume = bar.buy_volume + bar.sell_volume.abs();
                    self.exp_volume = self.volume_info.update(volume);
                    self.exp_imbalance = volume * self.imb_threshold;
                }
            }
            None => {}
        }

        // check the threshold is set
        if self.exp_volume == 0.0 {
            return None;
        }

        // n.b sell_volume is -ve
        let total_volume = self.bar.buy_volume - self.bar.sell_volume;
        let volume_imb = (self.bar.buy_volume + self.bar.sell_volume).abs();
        if total_volume > self.exp_volume || volume_imb > self.exp_imbalance {
            // set the next time a bar should publish
            self.bar.close = *price;
            self.bar.close_ts = *timestamp;

            // compute the volume weighted average price for the bar
            self.bar.vwap_price /= self.bar.buy_volume - self.bar.sell_volume;

            let mut res: Option<Bar> = None;
            if self.bar.open_ts != 0 {
                // take a copy of the bar information
                res = Some(self.bar.clone());
            }

            // reset the tracking values
            self.bar.reset(price, timestamp);
        }

        // update high/low prices
        if *price > self.bar.high { self.bar.high = *price; }
        else if *price < self.bar.low { self.bar.low = *price; }

        // update the vwap and quantity information
        self.bar.vwap_price += *price * (*qty).abs();
        if *qty > 0.0 { self.bar.buy_volume += *qty; }
        else { self.bar.sell_volume += *qty; }

        return res
    }
}
```

### Multi Bar Volume Imbalance Bars

I need to write an article about correlation structure in cryptos since its fascinating.
It's mostly fascinating since there isn't any!The first principal component of the correlation matrix is about 85% of variance and the first two components explained about 95% of the market.
So what does that means in practical terms? When bitcoin moves, or etherium moves, so does everything else.
There are good posts by [tr8dr](https://tr8dr.wordpress.com/2009/12/30/equity-clusters/) in regard to correlation clustering and [transfer-entropy](https://tr8dr.wordpress.com/2010/08/29/transfer-entropy/)
When this is applied to crypto currencies the results is not a very complex graph but you can see the structures between L0 coins, L1 coins DeFi/NFTs coins etc.

Conceptually I think volume should not be thought of as per asset, but per risk factor. So what we can do is make a volume bar builder
which ticks not when the volume in asset X breaks a threshold but when the amount of risk in a certain risk factor breaks a threshold.

A very simple version of this can be done by monitoring multiple assets and aggregating their volume. A new sample is taken when this aggregated volume crosses a threshold.
This is basically assuming that correlations are 1 for all assets which in crypto is not a particularly bad assumption.
n.b. The below implementation assumes all assets are in the same quoted currency.

Define the trait for a multi bar builder
```rust
pub trait MultiBarBuilder {
    fn trade_update( &mut self, timestamp: &u64, instrument_id: &u32,
        price: &f64, // price
        qty: &f64    // signed quantity
    ) -> Option<Vec<Bar>>;
}
```
Define the MultiDynVolumeImbBarBuilder class
```rust
pub struct MultiDynVolumeImbBarBuilder<T: MultiBarBuilder> {
    imb_threshold: f64,            // threshold parameter
    exp_volume: f64,               // expected volume
    exp_imbalance: f64,            // expected imbalance
    total_buy_volume: f64,         // total buys of all assets
    total_sell_volume: f64,        // total sells of all assets
    time_bar_counter: u32,         // number of time bars
    wait_time_bars: u32,           // number of bars to build threshold
    builder: T,
    bars: HashMap<u32, Bar>,       // collection of bars

    volume_info: Sma,
}
```
Implement the constructor and update methods
```rust
impl<T: MultiBarBuilder> MultiDynVolumeImbBarBuilder<T> {

    /* constructor */
    pub fn new(
        instrument_ids: &Vec<u32>,
        builder: T,
        imb_threshold: f64,
        vol_halflife: u16,
        wait_time_bars: u32,
    ) -> Self {
        debug_assert!(0.0 <= imb_threshold && imb_threshold <= 1.0);

        let mut bars: HashMap<u32, Bar> = HashMap::new();
        for iid in instrument_ids {
            bars.insert(*iid, Bar::new(*iid));
        }

        MultiDynVolumeImbBarBuilder {
            imb_threshold,
            exp_volume: 0.0,
            exp_imbalance: 0.0,
            total_buy_volume: 0.0,
            total_sell_volume: 0.0,
            time_bar_counter: 0,
            wait_time_bars,
            builder,
            bars,
            volume_info: Sma::new(vol_halflife),
        }
    }

    /* constructor */
    pub fn trade_update(&mut self, timestamp: &u64, instrument_id: &u32, price: &f64, qty: &f64) -> Option<Vec<Bar>> {
        let mut res: Option<Vec<Bar>> = None;

        // update normalisation bar builder to calculate average bar information
        match self
            .builder
            .trade_update(timestamp, instrument_id, price, qty)
        {
            Some(bars) => {
                let mut volume: f64 = 0.0;

                for bar in &bars {
                    if bar.buy_volume > 0.0 || bar.sell_volume > 0.0 {
                        volume += bar.buy_volume - bar.sell_volume;
                    }
                }

                // update average volume
                self.volume_info.update(volume);

                // wait until we have a few updates before publishing
                if self.time_bar_counter >= self.wait_time_bars {
                    self.exp_volume = self.volume_info.value;
                    self.exp_imbalance = self.volume_info.value * self.imb_threshold;
                }

                self.time_bar_counter += 1
            }
            None => {}
        }

        // n.b sell_volume is -ve
        let total_volume = self.total_buy_volume - self.total_sell_volume;
        let volume_imb = (self.total_buy_volume + self.total_sell_volume).abs();
        if self.exp_volume > 0.0 &&      // check the the expected volume is set up
            (total_volume > self.exp_volume || volume_imb > self.exp_imbalance)
        {
            // new bar to publish so record results
            let mut bar_res: Vec<Bar> = Vec::new();
            for (iid, tbar) in &self.bars..iter_mut() {

                if iid == *instrument_id { tbar.close = *price; }
                tbar.close_ts = *timestamp;
                tbar.vwap_price /= (tbar.buy_volume - tbar.sell_volume);

                // record bar and reset the tracking values
                bar_res.push(tbar.clone());
                tbar.reset(price, timestamp);
            }

            self.total_buy_volume = 0.0;
            self.total_sell_volume = 0.0;

            res = Some(bar_res);
        }

        // update bars
        let mut bar = self.bars.get_mut(instrument_id).unwrap();

        bar.close = *price;

        // update high/low prices
        bar.high = f64::max(bar.high, *price);
        bar.low = f64::min(bar.low, *price);

        // update the vwap and quantity information
        bar.vwap_price += *price * (*qty).abs();
        if *qty > 0.0 {
            bar.buy_volume += *qty;
            total_buy_volume += *qty;
        }
        else {
            bar.sell_volume += *qty;
            self.total_sell_volume += *qty;
        }

        return res;
    }
}
```

It should be noted that if a simple multi volume bar builder is needed then the imb_threshold can be set to 1.0.
This will give the same behaviour as not having the volume imbalance check at all.

The performance of these implementations could be improved upon.
This is from my research code so for the sake of code re-use one bar builder is embedded in another.
Since only the time by volume needs to be tracked for the dynamic volume bars it would be possible to simply use an exponentially decaying sum of volume as the threshold.
This would be more performant than generating complete time bars only to throw away almost all the information.
Also since bars are cloned it would be better to have the builder store the bar and then return a boolean with true as there being a new bar.
This would avoid any over head from memory allocation.

### Conclusion

1. Using time bars to set the volume threshold is a neat way to avoid seasonality issues and also the complexity of defining multiple thresholds for different assets.
2. Using time bars to set a baseline for volume imbalance bars is a good way to avoid the problem if the threshold becoming degenerate and greatly simplifies the choosing of hyper parameters.
3. When multiple assets are concerned volume can be thought of as risk and its worth generating volume bars synchronously by looking at total traded risk.


I think these ideas are important specifically in cryptocurrency markets due to the kurtosis necessitating some for of informationally uniform sampling (volume or volume imbalance)
and the correlation structure meaning volume traded on one asset cannot be viewed independently of the rest.

good luck, have fun.