---
layout: "post"
title:  "Intuiting noise"
date:   "2025-12-20 11:23:35 +0530"
categories: cryptography
--- 

How do you speak privately in a public setting? Think about it.

The internet's age-old problem still applies: anyone between you and the website you visit can see (and even modify) all communication flowing between them. If you're not just browsing the web, but instead building a peer-to-peer protocol or a custom network service, you still need a way to communicate privately. This is what the `Noise Protocol Framework` is designed to address.

But let's step back for a moment. How would you do it yourself?

## Encryption

The most straightforward idea is to speak in a code (cipher) known only to you and the peer. One side scrambles the message into something unreadable (encryption), and the other unscrambles it(decryption). This is a good starting point, but it quickly runs into issues. First, the code(cipher) must be sufficiently complex or it will eventually be broken (“use one letter over” becomes painfully obvious after a while). Second, the same code cannot be reused everywhere, or it stops being secret.

So instead, we generate a fresh code randomly and uniquely for each connection.

We can model this using a random number generator(RNG) construct that can be created with a starting point (seed). Each connection has its own seed. Encryption takes bytes from RNG and combines it with our message:
```
                    Enc=M⊕RNG(seed)
```

Decryption simply applies the same operation again:
```
                    M=Enc⊕RNG(seed)
```
The value `seed` acts as the lock and the key.If both sides know it, communication is private.

Wait... How do you know the other peer's key? If it's sent over the network, other's would see it too & create the same `RNG(key)` to decrypt our messages. Maybe it could distributed before-hand? But then what about new connections?

# Key Exchange

Fortunately, there's a neat algorithm to solve this: `Diffie-Hellman Key Exchange`. It relies on an operation (op) that is both "one-way" (`A op B = C` is easy, but the inverse `A = B ?? C` is very hard) and "commutative" ( `A op B = C` or `B op A = C`, order doesn't matter). Here's how it works:

Each side generates a random private key that is never shared (`k1` and `k2` respectively). From this private key, a public key is derived using a known one-way operation (`(k1∘G)` and `(k2∘G)` respectively). The public keys are exchanged openly. Each side then combines its own private key with the other side's public key to compute a :shared secret" key that's known only to both the sides.

One side computes `k1∘(k2∘G)`, the other computes `k2∘(k1∘G)`. Because the operation is commutative, both arrive at the same result. Because it is one-way, an observer cannot recover `k1` or `k2` from the public key alone.

This shared secret can now be used as the key for our random number generator. A private channel has been established without ever transmitting the key itself.

So far, so good.

All of this only prevents someone from reading the messages. But what if they can edit them too?

# Authentication

Encryption prevents eavesdropping, but it does not prevent interference.

1. When sending over the public keys, someone could swap our view of "the other's public key" to their own, resulting in us "secretly" talking to the middle-man instead of to other peer. How do peers trust that the other side is actually the other peer? (Key Authenticity)

2. When decrypting messages, someone could just replace the encrypted text with some that decrypts into an undesired message/response. Imagine bank sends an encrypted message asking to move funds. Someone making response decrypt to “yes” instead of “no” is pretty bad. How do the peer trust the other side is saying what they mean, even with a valid shared secret key? (Data Authenticity)

Noise addresses both problems explicitly.

# Data Authenticity

To prevent message tampering, encryption must also authenticate the data.

First, we'll need to introduce a hash function `H(x) -> y`. This is an operation that takes an input `x` and produces an output `y` that's resistant to "pre-images" (finding `x` from only `y` is very hard).

Then, we hash the encrypted message along with the secret RNG key `H(Enc ++ key)`, sending both `Enc` and the result together. The hash result acts as a way the  decryption side can use to double-check the message is right before accepting.

if we only hashed the encrypted message, the middle-man could also just hash their own malicious encrypted message. However, this hash includes the RNG key that's kept secret from the middle-man. Meaning they can't realistically compute a hash for their `MaliciousEnc` that satisfies `H(MaliciousEnc ++ key)` to pass decryption.

Such a scheme is called Authenticated Encryption (optionally hashing some Additional Data). Or `AEAD` for short.

# Key Authenticity

Alright, but how do we the shared secret key from being compromised?

Honestly? Not much you can do besides agreeing to something beforehand. A bit disappointing that there's no clever solution here, but think about it: you can't know you're talking to someone if you don't already know who they are.

So then the straight-foward approach is Pre-Shared-keys (PSK) that both sides would either put directly into AEAD (skipping key exchange), and use randomize Diffie-Hellman shared secret result.

In practise, Noise usually avoids shared secrets and instead relies on identities. Rather than agreeing on a secret ahead of time, one side can know the other's public key in advance or learn it once and remember it for future connections. This still requires prior agreement, but only on public information, which scales far better and does not need to be kept confidential.

# Closing

That, at a high level, is how Noise establishes private communication over an untrusted network.

Rather than bundling identity systems or policy decisions into the protocol, Noise focuses narrowly on constructing secure channels from simple, well-defined cryptographic steps. Everything is explicit, and nothing is hidden behind abstraction layers.

In the upcoming parts, we’ll take a closer look at how Noise is used within the SRI (Stratum Reference Implementation).