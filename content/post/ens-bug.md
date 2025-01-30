+++
author = "Hugo Authors"
title = "Integer overflow vulnerability in ENS"
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

![Image alt](/images/ens-bug/ens.png)

This is achieved by the ETH registrar contract. It's a contract that stores the mapping of the `.eth` subdomains to Ethereum addresses. [The ETH registrar contract](https://etherscan.io/address/0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85) is deployed on the Ethereum mainnet and is the only contract that can register new `.eth` subdomains. This is an oversimplified explanation, but it's good enough for this post.


The actual flow of registering a new `.eth` subdomain is something like this:
![Image alt](/images/ens-bug/ens-flow.png)

From the diagram above, it can be seen that EOAs cannot directly interact with the ETH Base Registrar contract. They have to use a controller contract to do so. Controllers handle the fees and the registration process. Also, controllers are trusted contracts that are deployed by the ENS team. Lets keep this in mind for now.

#### Finding the gem in the documentation

The ENS documentation has been updated since the bug was found. I'll link the archived version of the docs instead. While reading [the old ENS documentation](https://web.archive.org/web/20230331002457/https://docs.ens.domains/contract-api-reference/.eth-permanent-registrar), something caught my attention under the System Architecture section:

>Controllers may register new domains and extend the expiry of (renew) existing domains. **They can not change the ownership or reduce the expiration time of existing domains.**

This is a very important point. It ensures the censorship resistance of the ENS protocol. If controllers could change the ownership or reduce the expiration time of existing domains, the ENS protocol could be easily censored by the ENS team or ENS DAO.

With that in mind, I started reading the code of the ETH Base Registrar contract.

#### The overflow

Lets take a look at the `renew` function of [the ETH Base Registrar contract](https://github.com/ensdomains/ens-contracts/blob/master/contracts/ethregistrar/BaseRegistrarImplementation.sol#L171):

```solidity
    function renew(
        uint256 id,
        uint256 duration
    ) external override live onlyController returns (uint256) {
        require(expiries[id] + GRACE_PERIOD >= block.timestamp); // Name must be registered here or in grace period
        require(
            expiries[id] + duration + GRACE_PERIOD > duration + GRACE_PERIOD
        ); // Prevent future overflow

        expiries[id] += duration;
        emit NameRenewed(id, expiries[id]);
        return expiries[id];
    }
```











