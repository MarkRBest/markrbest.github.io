---
layout: post
title:  Book Reviews and Reading List
category:
- Books
---

How do you eat an elephant? I have wanted to write a reading list but I have been apprehensive since I didn't want to include too much and wanted also to explain why the books were in the list. 
If you want to trade crypto there is no point reading the Hull interest rate model book. This list likely will be a work in progress so keep that in mind.
As I write this I realise I have a deep problem with how many books I have.

These are not referal links, they are just Amazon links. Feel free to source the books from wherever you like. 

### General 

* [Advances in Financial Machine Learning](https://a.co/d/iatgtIp)

* [Trading Systems and Methods](https://a.co/d/4w2G4hk)

### Portfolio management

* [Quantitative Portfolio Management: The Art and Science of Statistical Arbitrage](https://a.co/d/cyOnB1B)

Gappy's book is an excellent guide to factor investing and also factor models for portfolio management. 

* [Advanced Portfolio Management: A Quant's Guide for Fundamental](https://a.co/d/dLmS1oY)

### Market Making / Trade Cost Analysis

This space is filled with books about Avellaneda & Stoikov models and stochastic control theory. A lot of it I don't even really understand and I had to learn stochastic calculus at university. 
In my opinion, there are no good books on market making that I know. The closest one is about trade impact modelling and trade cost analysis.
Market making is very similar to these ideas but in essence, you have a forecast model and you trade when cost < forecasts. So being efficient and being able to measure the costs is important.

* [Algorithmic Trading: A Practitioner's Guide](https://a.co/d/2ss0SuZ)

### Strategy Optimisation

This book is great. The issue with A/B testing is that you want to switch over to the best model as quickly as possible. The author is ex-getco and its very well written.

* [Experimentation for Engineers: From A/B testing to Bayesian optimization](https://a.co/d/gItrgAC)
* [Statistically Sound Indicators For Financial Market Prediction: Algorithms in C++](https://a.co/d/fi2g6TS)
* [Testing and Tuning Market Trading Systems: Algorithms in C++](https://amzn.eu/d/2T57Z9f)

### Volume Price Analysis

For discretionary trading Volume profiles are useful for understanding where to place stop losses and take profits. What happened last time the market was at a price often gives insight into what will happen next time the same price is reached.
Volume profiles tell you where a disproportionate amount of trading happened in the past and also if those orders are still there. 

* [A Complete Guide To Volume Price Analysis](https://a.co/d/1pTnmbg)

These books are mostly printed ebooks. They are cheap but are an introduction to how to read volume profiles for scalping.

* [ORDER FLOW: Trading Setups](https://amzn.eu/d/8n6kjsC)
* [VOLUME PROFILE: The insider's guide to trading](https://amzn.eu/d/jlOuwt8)

### Risk Management

This was a course book from my master's degree. It's an excellent foundational book on portfolio risk modelling.

* [Value at Risk: The New Benchmark for Managing Financial Risk, 3rd Edition](https://a.co/d/bzi1VpT)

# Regime Change

This isn't really that great a book. It is more of a paper that was converted into a book.
The ideas around direction change and having a time-invariant way to classify price changes are interesting.
The ultimate conclusion of the book is that their method doesn't outperform other strategies. The general idea is however useful if adapted.

* [Detecting Regime Change in Computational Finance](https://a.co/d/067yKkC)

### Books about traders

* [The New Market Wizards: Conversations with America's Top Traders](https://a.co/d/adiG2Ig)

This is a book about the life of someone who was a pit trader in the 80s. He was very successful and is just a very scrappy and likeable guy. 
There are a few interesting anecdotes to adapting to the world as it more from floor trading to screen trading.

* [Pit Bull: Lessons from Wall Street's Champion Day Trader](https://amzn.eu/d/0MkEmC9)

I have not read this but own it as it was a gift when I started at a hedge fund. It is mainly on here as people say you should read it. 
And they are correct, I really should get around to reading it. 

* [Reminiscences of a Stock Operator](https://amzn.eu/d/ixfco0z)

# Tail Risk Management

These two books are interesting but are likely not useful for 99% of people. The idea summarised is that options can be used to hedge downside risk. 
Options are also expensive due to theta and also in a normal market they have delta which you may not want as it interferes with the core strategy.
So by combining OTM puts and ATM calls, you can be long gamma and flat theta and delta. So you basically get free insurance.
The reason I don't think this is for most people is the risk of an options book rarely stays the same as exactly what you wanted so it needs to be actively managed.
If you don't know what you're doing, you might add more risk than the portfolio you're meant to be hedging.

* [Unperturbed By Volatility: A Practitioner's Guide To Risk](https://a.co/d/3v5cu7w)
* [The Second Leg Down: Strategies for Profiting after a Market Sell-Off](https://a.co/d/28xGfPx)

# Trading Platform and HPC

This book is not the best. That said, there are not many end-to-end books on the topic of HFT trading platforms. 
If you are new to trading platforms this is one of the few overviews I have seen.

* [Developing High-Frequency Trading Systems: Learn how to implement high-frequency trading](https://amzn.eu/d/8saVGdm)

This book is mainly about optimisation and instrumentation. What I liked about it most is that it says, "Don't assume measure". 
This is super important with building trading platforms as measuring end-to-end latency is not trivial. 
How do you know if it's fast if you can't measure it?

* [The Art of Writing Efficient Programs: An advanced programmer's guide to efficient hardware utilization and compiler optimizations using C++ examples](https://amzn.eu/d/8LlVD3W)

### General Programming

This book is a collection of Joel Spolsky's blog articles from around 2000. It's old but they are easy to read and some of the ideas like "eat your own dog food" or "cardboard Chewbacca" are timeless.

* [Joel on Software: And on Diverse and Occasionally Related Matters That Will Prove of Interest to Software Developers, Designers, and Managers](https://amzn.eu/d/beXU6rE)

### Refactoring

This book by Kent Beck covers a topic I often struggle with. When working on code you often want to clean up parts. 
This covers how to do that and how to organise PRs so that it is not a nightmare for other people to code review.

* [Tidy First](https://amzn.eu/d/6VzK0ml)

This book is about refactoring and although I don't agree with the core premise which is that functions should only be 5 lines. 
It is one of the few books that shows how to step by step re-organise code, extract sections and clean it up.

* [Five Lines of Code: How and When to Refactor](https://amzn.eu/d/bemylwD)

The two hardest problems in programming are overflow errors, cache invalidation and naming things.
This book is a small succinct discussion of the latter topic.

* [Naming Things: The Hardest Problem in Software Engineering](https://amzn.eu/d/cvmsIXP)
  
### Rust

This is only a small book and covers some of the weird ways that rust works that you might not expect. 
It is good for helping you to understand some of the lesser-covered topics.

* [Rust Brain Teasers: Exercise Your Mind](https://amzn.eu/d/dDHDIbj)

This is super specific to writing interprocess communication libraries. In general, synchronisation is something to be avoided in HPC since it is super slow.
It is important to deeply know about how atomics work since they are a lightweight way to achieve synchronisation if needed. 
Seq locks use atomics are are often used in trading platforms for L1 order book feeds.

* [Rust Atomics and Locks: Low-Level Concurrency in Practice](https://amzn.eu/d/jeSGmM6)

This is a rust reference book. So it is not really meant to be read cover to cover.

* [The Rust Programming Language](https://amzn.eu/d/5iQDjWL)



If there is anything that you feel is missing from the list then please let me know and ill try to read it and add it. 
