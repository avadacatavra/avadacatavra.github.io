---
layout: post
title: RWC-Post Quantum signatures are a Picnic
date: 2018-01-24 2:00:00
categories: rwc crypto post-quantum zero-knowledge
short_description: Pepe and Veronica play with boolean circuits
image_preview: assets/picnic.jpg
---

Welcome back to part 3/n where I try to figure out what hapens after the quapocolypse! Quantum-pocalypse?

# Post-quantum signatures  - Melissa Chase

If you don't already know, a sufficiently powerful quantum computer will be able to efficiently factor numbers and compute discrete logarithms. This breaks...everything (or at least public-key-everything). Currently, quantum computers can only handle a few q-bits, so we don't have to worry about this, right? Wrong. Crypto is hard. And it takes a while.

Since the underlying assumptions we've relied on in the past will no longer be hard, what will be hard? Some proposed quantum hard problems:
* lattice-based problems
* supersingular isogeny Diffie-Hellman
* code-based problems
* multivariate polynomial problems
* symmetric key primitives (or at least, we haven't thought of a way to attack better with a quantum computer)

[This talk](https://www.youtube.com/watch?v=_J9ESIy8D2o) is concerned with designing a PQ signature scheme using these symmetric key primitives. Wouldn't it be nice if we could reuse something we already have?

Also, I get super bored with Prover/Verifier so I'm going to call them Pepe and Veronica<sup>[1](#peggy)</sup>.

# A signature refresher

Pepe has a secret key, sk, and a public key, pk. Let's also assume that pk is a function of sk, say pk = F(sk) where F is hard to invert. He needs to prove to Veronica that he actually knows sk, without letting her know anything about sk. If this sounds suspiciously like a zero-knowledge proof, then give yourself a gold star.

There are a few desireable qualities:
* small keys
* small signatures
* quick signing/verification

The basics of a signature protocol are generally fairly consistent, so let's chat about Pepe and Veronica!

Pepe needs to prove that he knows sk such that pk = F(sk) (i.e. the public key is some hard function of a secret key). He could just send sk that over to Veronica, who would compute F(sk) to check, but that's not very zero-knowledge.

Seems like it would be a good idea to add some randomness in. Pepe grabs a random number r out of his magic hat and tells Veronica c = F(r). Veronica isn't quite sure about this whole thing, so she asks Pepe to tell her either the value of r or the value (sk + r)-- this is the challenge.

* Veronica requests r --> computes c1 = F(r) and verifies c1 = c
* Veronica requests (sk + r) -> computes c1 = F(sk+r) and verifies c1 = c * F(sk)

I've glossed over some (a lot) of details, but let's assume that's how you combine c and pk to check equivalence <sup>[2](discretelog)</sup>.

Pause here. Have we leaked any information about sk? Nope. The challenge requests two totally random values (assuming the magic hat is working properly). Can we tell if Pepe actually knows sk? If he actually knows it, he can respond correctly to either challenge. Can Pepe cheat? Maybe.

If Pepe knows that Veronica will request r, then everything is unicorns and rainbows. He doesn't even have to know sk in this case. If Pepe knows that she'll ask for (sk + r), then he'll pick a random value r' and compute c' = F(r')/F(sk). Now he's cheated and Veronica is sad with 0.5 probability. Cheating is only possible if Pepe anticipates the challenge:

* If Pepe thinks Veronica will pick r, but she asks for (sk + r), he'll fail. He doesn't know sk.
* If Pepe thinks Veronica will pick (sk + r), but she asks for r, he would have to invert the hard problem F if he cheated as described.

Ok, so if Pepe and Veronica only run this protocol once, then he can cheat half the time. So, let's just do it over and over until that probability becomes acceptably negligible.


## Identification versus signature

The above can easily be made non-interactive using hashes, but I'm going to just state that and leave it.

There's a slight nuance between identification and signing. The interactive protocol above is an identification protocol, so a non-interactive version directly translates to a digital signature. But beyond that, a digital signature implies both identity/authentication and message integrity, while identification gives no guarantees about the latter.

Also, not all interactive protocols are identification protocols. For a non-trivial identification protocol, you need a public key that corresponds in some (difficult to invert) way to a private key.


# Hand-waving over, lets talk Picnic

[Picnic](https://eprint.iacr.org/2017/279) is the proposed PQ signature scheme. It requires a hash function and a block cipher--yay I don't have to learn a new math in order to figure this one out (I hope). And--this is where it gets real--Picnic doesn't use a [Merkle tree](https://en.wikipedia.org/wiki/Merkle_tree)! Nothing against them, but she makes a great point that there's some benefit to a diversity of approaches.

Spoilers: Picnic achieves small keys, large signatures, moderate signing and verification time. They suggest [SHAKE](https://en.wikipedia.org/wiki/SHA-3#Instances)<sup>[3](#shake)</sup><sup>[4](#shake2)</sup> and [LowMC]() as the hash function and block cipher.

## ZKBoo

In addition to the hash function and block cipher, we need some way to write the proofs required by the scheme. Enter [ZKBoo](https://eprint.iacr.org/2016/163.pdf), a ZK proof system for boolean circuits.

* secret key corresponds to inputs x1...xn
* public key corresponds to outputs y1...ym<sup>[5](#numbering)</sup>
* the boolean circuit is the hard-to-invert function

![Picnic Interactive Protocol]({{ "assets/pqsig.png" | absolute_url}})

Great, now you're a PQ expert!

Is it ZK? a1, b1 don't leak anything about a or b. c1, c2 don't reveal anything abut a or b either. Looks good to me. Thanks magic hat of randomness!

Just like current signature schemes, the cheating probability is 0.5, so you have to run multiple rounds with fresh random numbers to reduce this.

There are two ways they've transformed the interactive protocol into a non-interactive signature scheme:

1. Fish Signatures: uses the [FS transform](https://en.wikipedia.org/wiki/Fiat%E2%80%93Shamir_heuristic)
  * secure under the [Random Oracle Model](https://blog.cryptographyengineering.com/2011/09/29/what-is-random-oracle-model-and-why-3/)

2. Picnic Signatures: uses [Unruh's transform](https://eprint.iacr.org/2014/587.pdf)
  * secure under the [Quantum Random Oracle Model](https://eprint.iacr.org/2010/428.pdf)

On the way to proposing these two signatures, they optimized ZKBoo to form ZKB++, resulting in smaller signature sizes. They've proven each secure in their respective random oracle models, and they've written a [reference implementation](https://github.com/Microsoft/Picnic). Using this implementation, they've done some initial TLS experiments that look promising (1.4-1.7x performance hit).



# Certificate Transparency

My only note from this talk reads

> In summary, CAs are a bargain with the <redacted> devil

---

<a name="peggy">1</a>: I'm aware that Peggy/Victor occurs in the literature, but I like my version more.

<a name="discretelog">2</a> I'm basically leaving out all of the discrete log details. In this case, the challenge would be (sk + r) mod (p - 1), where pk = g^sk mod p. It's then just some arithmetic to see that g^(x+r) mod (p-1) = c * pk mod p, where c = g^r mod p as long as I typed that up correctly without LaTeX.

<a name="shake">3</a>: Protip - if using a monitored computer, maybe wait til you get home before typing 'shake hash' into Google

<a name="shake2">4</a>: Also, technically SHAKE is an extendable-output function (XOF), but you can use them in similar ways to hash functions

<a name="numbering">5</a>: Yes, I'm morally opposed to 1-indexing, but that's how they did it