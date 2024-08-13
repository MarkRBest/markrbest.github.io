---
layout: post
title:  Even Faster Logging in Rust! 
category:
- HFT
- Rust
- Algo Trading
---

I was re-reading some older posts, and I realised I owed some readers a follow up. Hopefully I will be forgiven that this took 2 years.
This post will be short and the core of the ideas are a follow up to the [original article](https://markrbest.github.io/fast-logging-in-rust/) here.

The key takeaways from the original article are: 

1. IO and Logging should not be done on the strategy hot path.
2. Formatting is expensive, and it is better to log variables and to format them on the logging core.

This article introduces a couple of improvements that could be made to the original code. 

1. Pushing a closure onto the ring buffer means a Box<T> is created on the heap.
2. The format string needs to be pushed onto the ring buffer each time which can be larger than the data itself.
3. Performance can be better optimised when an object's size is known at compile time. 

The solution to the above issues is pretty simple. 
The RawFunc is replaced with a Logging Enum and a set of Logging Messages that define sets of data to be logged. 
Then these enums can be pushed on to the logging queue as if they were any other data. This avoids the need for both the Box and the formatting string. 
Using the rust type system, it is then easy to associate the data logged with any formatting process and logging string and make sure it is logged correctly.  

```rust
#[derive(Debug)]
struct QuoteLogData {
    instrument_id: u32,
    bid: f64,
    ask: f64,
}

impl QuoteLogData {
    fn new(instrument_id: u32, bid: f64, ask: f64) -> Self {
        QuoteLogData { instrument_id, bid, ask }
    }
}

impl Display for QuoteLogData {
    fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
        write!(f, "Quote: instrument_id: {} bid: {} ask: {}", self.instrument_id, self.bid, self.ask)
    }
}

#[derive(Debug)]
struct AlphaLogData {
    instrument_id: u32,
    ema_fast: f64,
    ema_slow: f64,
    vola: f64,
}

impl AlphaLogData {
    fn new(instrument_id: u32, ema_fast: f64, ema_slow: f64, vola: f64) -> Self {
        AlphaLogData { instrument_id, ema_fast, ema_slow , vola}
    }
}

impl Display for AlphaLogData {
    fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
        write!(f, "Alphas: instrument_id: {} ema_fast: {} ema_slow: {} vola: {}", self.instrument_id, self.ema_fast, self.ema_slow, self.vola)
    }
}

#[derive(Debug)]
enum LogMessage {
    Quote(QuoteLogData),
    Alphas(AlphaLogData)
}

impl Display for LogMessage {
    fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
        match self{
            LogMessage::Quote(msg) => { write!(f, "{}", msg) }
            LogMessage::Alphas(msg) => { write!(f, "{}", msg) }
        }
    }
}
```

The nice thing about using logging enums is the formatting and format string is defined alongside the data. 
Over time as the log messages evolve, it is trivial to change the formatting as well and keep it up to date.

Setting up the sender and receiver is as simple as creating a lockfree queue to receive the LogMessage enums and print them out or send them to trace.

```rust
use lockfree::channel::spsc::{create, Sender, Receiver};

let (mut sx, mut rx) = create::<LogMessage>();  // lock free channel

let guard = thread::spawn(move || {
    let core_ids = core_affinity::get_core_ids().unwrap();
    core_affinity::set_for_current(*core_ids.last().unwrap());

    match (rx.recv()) {
        Ok(msg) => { println ! ("{}", msg) }
        Err(e) => { panic!("well this is not good"); }
    }
});

// send a log messages
sx.send(LogMessage::Quote(QuoteLogData::new(2008, 6000.25, 60000.26))).expect("Error sending log message");
sx.send(LogMessage::Alphas(AlphaLogData::new(2008, 10.0, 20.0, 1.0))).expect("Error sending log message");

```

The new enum based code is also significantly more performant compared to the old code.
These are the criterion timings from a very simple benchmark. 
Please note, the benchmark only measures the time taken to put a log message onto the respective queue.

```
raw_func_logging        time:   [172.23 ns 173.09 ns 173.97 ns]
enum_logging            time:   [127.78 ns 128.37 ns 128.96 ns]
```

Hopefully it is clear why this new code is an improvement over the old version. I still like the idea that rust closures are easy to send over a channel.
In the case of logging, it is just more efficient to send simple data objects and use a type system to define the formatting to be applied later.