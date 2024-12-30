+++
author = "Hugo Authors"
title = "A funny little overflow bug in ENS (Ethereum Name Service) that landed me $100,000"
date = "2024-12-17"
description = "A funny little overflow bug in ENS that landed me $100,000"
tags = [
    "ENS",
    "Bug Bounty",
	"Reward",
]
categories = [
    "Bug Bounty",
]
+++

Lets start with an arbitrary quote from Sun Tzu because why not and I like random quotes:
>know yourself and you will win all battles \
> \- Sun Tzu 

This is my story about a funny little overflow bug I found in [ENS](https://ens.domains/) that landed me $100,000. The "funny" part is that I wasn't even looking for a bug in the ENS protocol. I was just trying to understand how ENS works.

It all started when I saw a new ENS audit contest was starting on [Code4rena](https://code4rena.com/audits/2023-04-ens-contest). 

{{< box info >}}
If you aren't familiar with Code4rena, it's a platform where you can participate in audit contests for various web3 protocols. The contests run for a certain period of time, have a fixed reward pool and the winners share the pool. [Check out the docs for more info.](https://docs.code4rena.com/)
{{< /box >}}

When I saw the ENS contest, I thought it would be a good opportunity to learn about the ENS protocol. I started reading the documentation before jumping into the code. 

If you aren't familiar with the ENS protocol, I suggest you to check out <https://docs.ens.domains/learn/protocol>. But I'll provide a short summary from the docs:

>ENS maps human-readable names like 'alice.eth' to machine-readable identifiers such as Ethereum addresses, other cryptocurrency addresses, content hashes, metadata, and more. 

So imagine how DNS works, but instead of mapping human-readable domains to IP addresses, it maps domain names to Ethereum addresses. 

