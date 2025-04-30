+++
author = "Hugo Authors"
title = "Overflowing the ENS Registrar"
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

It takes two arguments: the id of the domain and the duration of the renewal. The duration is added to the expiry of the domain. The `GRACE_PERIOD` is 90 days and it's constant.

The first `require` statement ensures that the domain is registered or in the grace period. The second `require` statement ensures that the duration does not overflow the expiry of the domain.

But do you see the problem here? I recommend you to pause for a moment and think about it.

...

...

...

At first glance, everything looks fine. The second `require` is supposed to prevent an integer overflow when adding duration. But let's look closely.

The duration is user-supplied. So what happens if the user sets duration to `2^256 - GRACE_PERIOD`?

```Solidity
expiries[id] + duration + GRACE_PERIOD > duration + GRACE_PERIOD
```

Both sides of this expression will overflow and wrap around, making the condition always true. The check is completely bypassed.

And then this line executes:

```Solidity
expiries[id] += duration;
```

This causes `expiries[id]` itself to overflow. If `expiries[id]` was larger than `GRACE_PERIOD`, then adding duration (which is effectively `-GRACE_PERIOD`) reduces `expiries[id]` instead of increasing it!

By repeatedly calling `renew` with this specially crafted duration, an attacker can actually force an ENS domain's expiration time to go down, eventually expiring it. That’s a direct violation of the protocol's design guarantee from the documentation.

#### Impact

If exploited, this vulnerability could allow an attacker (or a malicious DAO controller) to force the expiration of any `.eth` domain. Once expired, the attacker could quickly register and claim the domain as their own.

Now, practically speaking, this attack has some big caveats:

* Calling `renew` costs ETH proportional to the `duration`. So using this to expire a valuable domain like `vitalik.eth` would require an astronomical amount of ETH—more than even exists.

* But if ENS ever introduces a fixed-price, unlimited-renewal system (like a subscription model), this bug becomes immediately exploitable.

So while it's not exploitable today in the real world, it’s a serious design flaw that could become critical in future changes.


#### The fix

Since the ENS ETH registrar contract is not a proxy contract, it's immutable. So, this issue cannot be fixed easily. ENS team introduced a patch in [this commit](https://github.com/ensdomains/ens-contracts/commit/e6b136e979084de3761c125142620304173990ca) that could in theory fix the issue however they decided to not deploy the fix as of now. The [ENS discussion board](https://discuss.ens.domains/t/security-advisory-a-malicious-dao-update-could-reduce-the-registration-duration-of-registered-eth-2lds/17576/8) has comments on explaining the reason behind their decision. The fix is too complicated to go over in this blogpost. So I'll leave it to the readers to go and check it out.















