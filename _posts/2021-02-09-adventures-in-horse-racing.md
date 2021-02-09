---
layout: post
title:  Adventures in Horse Racing
category:
- Betting Markets
- Horse Racing

---


Back in 2008 when I worked at Deutsche Bank I became interested in algorithmic trading. At the time we were working on some automated pricing models for trading Euro Gov fixed income which at the time was quite novel. 
The issue then was that professional markets were inaccessible and there was little in the way of retail offerings. I have never heard of Interactive Brokers and cryptocurrency was not even on the horizon. So I turned to Betfair.
The thing that was interesting about Betfair is that it was free. You could connect to the API and which was built on a SOAP interface and check races, manage risk, place bets automatically. 
Now it's far simpler to do since there are libraries that handle the connectivity but back then it was quite complicated and required implementing the whole connectivity layer yourself. After learning about TCP connection pooling and co-location I managed to get the tick to trade latency down to about 14 ms from 300ms using a laptop at home. It was lighting quick considering it was public internet and an http transport. 

Getting to this point was a total nightmare and also introduced me to the world of leaky abstractions. The idea of soap was to abstract the complexity of transmitting complex objects over HTML. This is something that has now been effectively solved by json. It turns out that the specifications for SOAP were different between .net and java. The connectivity layer I had written was a java implementation on top of apache. The Betfair backend however was written in .net. 
this caused no end of issues when trying to make everything work and required hand-coding the .wsdl object definition file. The only reason I bring this up was that what now is about 3-4 hours of work to write a bot from scratch using an existing python library I think took over a month to just be able to get a list of all tradable markets and to be able to manage orders.

There was another betting exchange based in Ireland called Betdaq which I think has since been acquired by Ladbrokes. 
For a while, I was writing arbitrage bots between races on the same meet.
It was a great learning experience and the data I gathered during all of this work allowed me to write my [Masters Thesis](/assets/2021-02-09/MarkBestThesis.pdf). 
Betting markets are incredibly interesting since they are one of the few markets which are pretty much isolated. In trading, most assets are priced on a relative value to everything else.   
In a horse race, the price of a horse is related to the other horses and a few other factors but not much else. 
In this case, you would expect pretty much efficient markets. This, however, is very far from the truth. 
I think the best explanation of this is by Joesph Stiglitz who argues that if information is costly there is no incentive for agents to
participate in markets unless the profits of doing so at least cover the outlay of information costs.

I eventually moved away from betting markets since Betfair has a monopoly on market access and after going public it became clear that anyone who profits from their site would be charged accordingly. I never like the idea of bookies and casinos for the same reason that if you start to make money they will kick you out. 
