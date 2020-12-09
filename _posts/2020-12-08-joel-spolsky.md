---
layout: post
title:  Joel On Software
category:
- Books

---

Joel Spolksy is a computer scientist and blogger that wrote a lot of books about 15-20 years ago.
Some of his observation about programming are timeless and worth knowing about.
Joel has light enjoyable writing style making his articles both educational as well as being easy to read.
His blog [joel on software][joel-on-software] was also compiled into a series of [books][jos-book].
Some of the blog posts are pertinent today and even explain my interest in maintaining this blog.
I have tried to include some of my favourites here.

## [The Law of Leaky Abstractions][abstraction]

Programming is full of abstractions and they are a major mechanism to make understanding complex topics easier.
Compared to C and C++, garbage collected languages such as Python and Java are far simpler to use because
you don't need to directly manage memory.
The user is abstracted from the memory by the allocator and garbage collector.
This however is not perfect and it's possible to still have memory leaks and other problems.
A leaky abstraction is one that forces you to still have to understand how the layer under the abstraction works.
The abstraction saves you time but relying on it too much can lead to other issues which are far harder to solve.

## Cardboard Chewie

A client entered into a software company and noticed one of the developers sitting at a table with explaining a complicated
issue they were having with a life sized cardboard cut out of chewbacca. The person asked a manager what was going on and it was
explained to him that 4 out of 5 times chewie is able to solve the issue for you.
This had the added benefit of reducing the number of times one developer would need to interupt another to ask questions.
The process of explaining and issue can often highlights something that was missed or allow a person to reconsider the context enough to be able to answer the question themselves.

## [Joels 10 Rules][10rules]

The article is a simple check list of things companies can do to make sure the quality of their code is as good as possible.
I am amazed that even today I have worked for companies that don't have continuous integration. It is also difficult since
nowadays for quantitative trading there is an expectation for quants to also have a high level of programming as well as dev ops.
Knowing how to use git properly is invaluable even as a quant.

1. Do you use source control?
2. Can you make a build in one step?
3. Do you make daily builds?
4. Do you have a bug database?
5. Do you fix bugs before writing new code?
6. Do you have an up-to-date schedule?
7. Do you have a spec?
8. Do programmers have quiet working conditions?
9. Do you use the best tools money can buy?
10. Do you have testers?
11. Do new candidates write code during their interview?
12. Do you do hallway usability testing?

## [It's Better to Refactor than Rewrite][netscape]

The temptation to start from writing a piece of software from scratch is always there but in almost all instances is the wrong decision.
Joel talks about how netscape was the market leader and then decided to rewrite everything from scratch. This was a massive
mistake since by the time they had completed the re-write they had completely lost their market share.

## Always Ship Your Code

It is better to get something close to correct into production since there is a huge amount to be learn from live trading.
It is a similar problem to analysis paralysis in quantitative research. There is always a temptation to do a little more tuning.
In the same regard with software there is always a tradeoff between getting an imperfect product to a client and never delivering anything.
I think one of the main advantages is that the faster the code ships, the faster issues with the foundation code are identified and fixed.
This saves a lot of waste time in having to retest (backtest) software once at a later data a bug is found.

## [Eat Your Own Dog Food][dog-food]

To write good software the company should use the software. If you won't use it, why would anyone else?


[jos-book]: https://www.amazon.co.uk/dp/1590593898/ref=cm_sw_em_r_mt_dp_SBo0Fb1K1DPQK#ace-3536363283
[joel-on-software]: https://www.joelonsoftware.com
[10rules]: https://www.joelonsoftware.com/2000/08/09/the-joel-test-12-steps-to-better-code/
[netscape]: https://www.joelonsoftware.com/2000/04/06/things-you-should-never-do-part-i/
[abstraction]: https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/
[dog-food]: https://www.joelonsoftware.com/2001/05/05/what-is-the-work-of-dogs-in-this-country/