---
layout: post
title:  The Rust Programming Language
category:
- HFT
- Programming
- Rust
---

I love programming! There is something really satisfying about solving a complicated problem concisely.
That said I see programming languages as a tool to solve a problem rather than purely coding for coding sake.
I have used a lot of programming languages over the last 20 years namely Java, R, Matlab, Python, C++ and now Rust.
It is pretty common to read articles about language wars and which one is best. Would you write a website in assembler?
Try embedding javascript in a washing machine. Most of these discussions in my opinion are asinine and often miss the context of the problem,
and the resource constraints associated with programmer time.

I have not learnt a new language in a long time (since I learnt python). Learning a new programming language is a large time investment.
If you cannot use it professionally then the amount of time you can dedicate to it is limited. If a language cannot solve practical problems well, then there is also a question as to if it is a good language.
This, from what I can work out seem to be the conflict between imperative and functional languages.
I do however need to spend some time looking at Haskel since I have heard good things about its type system, and I know Jane Street makes use of OCaml so there is clearly some merit to that.

### Problem space

A lot of cryptocurrency trading platforms are written in python. There are some great libraries such as [cryptofeed](https://github.com/bmoscon/cryptofeed) that make getting data simple.
Python is also a very productive programming language which is why I think is one of the reasons that there is so much great library support for it.
In saying that the performance of python is pretty poor and python's approach to multithreading/multiprocessing is a big issue. There are ways around it and there is shared memory support but in general, it is not great.
A crypto strategy might listen to upwards of 10-20 feeds from multiple exchanges. When markets are busy this might involve getting 10s of thousands of messages per second.
This is just beyond the ability of python to deal with efficiently. This is not a direct criticism of python any more than saying a hammer is shit at unplugging drains, it is just not what it is for.
An alternative is to use C++ since it is a complied language and has much better performance. I have known people to try this and invariably the issue is the time to market.
That is not to say C++ is not a good language, some things can be done in C++ that I don't know how to do well in other languages (intrusive collections for fast order book building).
It is just that it can take a much longer time to write the same amount of business functionality. Take the last statement with a pinch of salt This is also dependent on the skill level and the experience of the programmer.

Rust is touted as a C++ killer which put me off looking at it for a long time since I think this does it a disservice.
Many languages have claimed this, and it is inevitably unlikely to be true. Say what you like about C++ but there is a reason that it has stood the test of time.
Someone I worked with said ```"C++ is your best friend and your worst enemy"``` and I think this is a good summary of using it.
That said, I think that alternative languages can carve out niches and that is how I feel about Rust and HFT.
I have been hearing more and more about Rust from different sources, that it warranted a deeper dive. What I found I am very impressed with.

### So why Rust?

Just as a caveat there are a lot of articles about the good things about Rust. This is not supposed to be a comprehensive analysis, and I am still learning the language myself.
It is however the list of the specific features of Rust that mean it is possible to code highly performant highly concurrent systems without having to spend a huge amount of time
fixing memory leaks and dealing with a range of problems associate with multi-threading.

Rust is a compiled language with no runtime. This alone gives it a lot of advantages in terms of performance.
The main advantage in my opinion specifically to HFT is:

- It is very fast.
- The lack of runtime and gc means the latency distribution is much more predictable.
- The Cargo build system makes using libraries easy.
- Concurrency and inter-process communication is really simple.
- String functionality makes working with json and string parsing almost as easy as python.
- It is designed with unit testing in mind and this is easy to do with Cargo.

The main two advantages in my mind are lack of run time and what Rust calls ```"fearless concurrency" ```
The reason some languages have a run time is partly portability and partly for garbage collection. Rust solves both the issues relating to memory management and concurrency using the same idea.
The way Rust does this is using the concept of ownership and borrowing. When an object is passed to a function it is moved, and the ownership transferred. The ownership then controls the lifetime of the objects and the responsibility for de-allocation.
This is a big pain in other languages since there have to be strict rules about if memory is allocated within a function or outside and who is responsible for de-allocation. This is easy to mess up and is problematic when using libraries that have different conventions.
Ownership also solves a lot of the concurrency problems since if objects have one owner then the problem of race conditions are greatly simplified.

The bonus of not having a runtime specifically in regards to HFT is to do with the predictability of latency. The issue with garbage collectors is they kick in when the program is busy and producing a lot of garbage.
Benchmark statistics that only show the meantime taken can conceal the fact that when things are busy the program takes many times longer to do the same task. For languages such as C++ and Rust, this is not the case since there is no garbage collector.
The variance in performance is then mainly a question of the program's ability to use the memory system efficiently and to make sure as much data as possible is in the cache.
It is also trivial in Rust to set up CPU core affinity to make sure the CPU cache is not flushed with data that doesn't relate to the trading strategy.
Long story short the distribution of the tick to trade latency for trading platform written in Rust will have a much smaller difference between the 50th percentile and the 99.9th percentile.
Most of the interesting activity such as fills etc happen in the 99.9th percentile hence the often masked issue of the tick to trade tail latency.

I think Rust is designed to have most if not all the performance advantages of C++.
This is also dependent on programmer skill and also the amount of time you have for tuning. Rust however solves another important practical problem.
I think it is far more productive. Given a certain amount of programmer hours it is possible to write a lot more business functionality with Rust.
Given infinite time it is likely possible to make something more performant in C++, however, with Rust you can say with more confidence that there will not be memory leaks or concurrency issues and spend more time thinking about business logic.


### Code Example

There are also a lot of other great things about Rust but as I am mainly making the case for why I see Rust being useful for HFT.

This is a simple example of a producer/consumer that can be used to send market data between two threads that are running on assigned cores using a lock-free ring buffer.
It is nice how different data types can be wrapped in enums that makes IPC easy. Rust also supports various forms of channels that are single producer single consumer (spsc),
multi producer single consumer (mpsc) and multi producer multi-consumer (mpmc). The match statement in Rust is similar to switch statements and makes dealing with enums easy.

n.b. this won't compile so it just an example.

```rust

pub enum Side { Bid, Ask }

pub struct L2Level {
    pub price: f64,
    pub qty: f64,
    pub count: u32,
}

pub struct L2BookUpdates{
    pub sequence_id: u64,
    pub timestamp: u64,
    pub instrument_id: u32,

    pub bids : Vec<L2Level>,
    pub asks : Vec<L2Level>,
}

pub struct TradeUpdate {
    pub sequence_id: u64,
    pub timestamp: u64,
    pub instrument_id: u32,

    pub side: Side,
    pub price : f64,
    pub qty: f64,
}

pub enum MarketDataMessage {
    Quote(L2BookUpdates),
    Trade(TradeUpdate),
    Close
}

fn main(){
    let rb = RingBuffer::<MarketDataMessage>::new(100);
    let (mut prod, mut cons) = rb.split();

    // producer
    let pjh = thread::spawn(move || {
        // set which core the producer should run on
        let core_ids = core_affinity::get_core_ids().unwrap();
        core_affinity::set_for_current(core_ids[0]);

        // push some data
        for i in 0..10 {
            let ts = SystemTime::now().duration_since(SystemTime::UNIX_EPOCH).unwrap();
            let ts_mics = u64::try_from(ts.as_micros()).unwrap();
            let update = TradeUpdate::new(i, ts_mics, 0, Side::Bid, 1000.0, 10);

            prod.push(MarketDataMessage::Trade(update)).unwrap();
        }

        // tell the consumer it can exit
        prod.push(MarketDataMessage::Close).unwrap();
    });

    // consumer
    let cjh = thread::spawn(move || {
        // set which core the consumer should run on
        let core_ids = core_affinity::get_core_ids().unwrap();
        core_affinity::set_for_current(core_ids[1]);

        loop {
            if !cons.is_empty() {  // this is basically using a spin lock
                match cons.pop().unwrap() {
                    MarketDataMessage::Quote(book_update) => {
                        println!("{:?}", book_update);
                    }
                    MarketDataMessage::Trade(trade_update) => {
                        println!("{:?}", trade_update);
                    }
                    MarketDataMessage::Close => {
                        break;
                    }
                }
            }
        }
    });

    // wait for threads to complete
    pjh.join.unwrap();
    cjh.join.unwrap();
}

```

### Performance

I have not had the time to do anything more formal but from the simple market data collector I have been working on the processing time per quote message is about 12 microseconds including string parsing
and for trade messages it is about 6 microseconds. This can likely be improved by pre-allocation of strings and making assumptions about message structure. For python systems, these numbers were more in the 250-500 microsecond range.
I am looking forward to seeing what the latency distribution will be for a full trading system. I expect it will be in line with C++ numbers of single-digit micros for tick to trade with about 4-5x 99.9th percentile.

### The Issues

Rust has a steep learning curve. This however can be explained in terms that are easier to understand.
If you have not worked with compiled languages or have not had the fun of working with heavily templated C++ code before then you might not be used to dealing with a compiler.
The normal process for python is, write some code, run it, see if it did what was expected and then make some modifications.
I'm not saying it is a good process if anything it is an incredibly lazy way to program but this is the price of flexibility.
The Rust compiler is very strict. The development process with Rust when you are learning involves,
writing some code, getting compiler errors, googling for 30 minutes about what RefCells are, then fixing it and then getting more errors.
This is not a criticism as such, the good thing about a strict compiler is that you can have much more confidence that the program will run correctly if it compiles.
It just means getting your first non-trivial example to compile can take a little while.


Rust is a relatively new language so there is limited functionality in the std library. There are multiple libraries for things like json, logging etc but in my experience,
especially while learning, the documentation support for it is not great. For most libraries, there is a single trivial example and google doesn't find much on stack exchange other than method definitions.
This will be something that improves over time as things mature, but it adds to the steepness of the learning curve.

There are others, but these are the main ones that I found.

### Conclusion

It seems to me that it is only a matter of time before Rust is the dominant language for writing HFT trading platforms.
There is a huge amount of inertia in the form of porting code bases and retraining people so this migration will take time.
As we hit the physical limits of transistor density and CPU counts grow it would seem logical to use languages that allow this power to be unlocked efficiently.
I have been very impressed just from writing some simple things for the crypto markets.
Going forward I am planning to only use Rust for data collection, trading platform and simulator and then use python/pandas/numpy for signal research of pre-processed datasets produced by the rust code.
