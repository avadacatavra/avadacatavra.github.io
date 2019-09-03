---
layout: post
title: RWC-Privacy Preserving Machine Learning
date: 2018-01-24 2:00:00
categories: rwc crypto levchin-prize cloud machine-learning data-oblivious
short_description: preparing for the day when the machines take over
image_preview: assets/cyberman.jpg
---

Welcome back to part 2/n in my 'hey look at the cool things that happened at RWC' series.


# Levchin Prize

Since 2015, the [Levchin Prize](levchinprize.com) has been awarded at RWC. So far, each year there's been one individual recipient and one team recipient. This year, it went to Hugo Krawczyk (he created HMAC!) and the OpenSSL team (for dramatic improvements to code quality). 

I can attest to the improvements to the quality in OpenSSL. A few years ago, part of a final in school was to trace some OpenSSL code. I love tracing code, and it had me jumping all over the place until I couldn't open any more vim buffers (only a slight exaggeration). It's much less awful today.

More importantly, the [trophies](http://www.bennettawards.com/custom-awards-gallery/levchin-prize) are awesome. They're based on a 'cryptex' and a wheel cipher. There are 7 rotating disks, and when they're aligned correctly, the trophy opens to reveal an inner chamber. No clue what's in it.

# Side-channel mitigation -- Olya Ohrimenko

## Azure

Azure is Microsoft's cloud computing platform. They've [integrated confidential computing](https://azure.microsoft.com/en-us/blog/introducing-azure-confidential-computing/
) so that when data is being processed, it's in a Trusted Execution Environment (TEE). When the data is at rest, it's encrypted (as far as I can tell), but processing encrypted data is really inefficient (and not always possible). This presents a problem for cloud users with sensitive data--they have to protect the data, but they can't simultaneously use it and ensure privacy protections.

Microsoft currently supports both software (Virtual Secure Mode) and hardware (Intel SGX) secure enclaves, and is working to make more available.

In particular, this talk addressed information leakage through memory accesses (she explicitly was not addressing Spectre/Meltdown). Consider a binary decision tree where all of the data is encrypted. Although the data is encrypted, you can still observe when memory is accessed and where. If you then do multiple traces, you could possibly reverse engineer the tree content.

That's...not good--imagine that decision tree is a diagnostic tool, where the nodes contain your medical information. Now anyone who saw those memory accesses knows that you have Lupus (do you not spend your Saturday nights looking at cloud memory accesses?)! As with other side-channel flavors, you need some sort of 'secret-independence.' 

## Memory side-channel mitigations

A naive approach is to insert dummy accesses into the code, but this is ad hoc, and still might leak data. Instead, you want to use a traditional, game-based security proof. This resulted in an idea called 'data-obliviousness.' Informally, this means that any algorithms shouldn't branch on secret data. More precisely

> We specify *public parameters* that are allowed to be disclosed (such as the input sizes and number of iterations to perform) and we treat all other inputs as private. We then say that an algorithm is data-oblivious if an attacker that interacts with it and observes its interaction with memory, disk and network learns nothing except possibly those public parameters. - [USENIX paper](https://www.usenix.org/system/files/conference/usenixsecurity16/sec16_paper_ohrimenko.pdf)

The talk mentioned four mitigation techniques:
1. Data obliviousness
  * isolate computations in private memory
  * attacker can see addresses that go in/out, but not how the content is manipulated
  * limited private memory (for example, max 2kb in registers<sup>[1](#sgxnote)</sup>)
2. Explicit private memory
  * operate on registers directly in assembly
  * IMO, assembly should not be the goto option for side-channel mitigation
3. Intel TSX
  * initially developed for concurrent data access
  * side effect: if data is evicted from the cache, the transaction aborts
  * can be used to turn caches into private memory <sup>[2](#tsxnote)</sup>
      * data becomes part of the read set of the transactions; therefore, if the attacker tries to evict it from the cache, the transaction aborts
      * cache misses can't happen, so caches become private memory
  * size limitations
3. Oblivious RAM (ORAM)
  * A more general solution to the problem, can be thought of as a proxy
  * hides which addresses are accessed and how often, but not the number of requests
  * data solution only, not code

But, back to the point--the talk was about data oblivious algorithms. They wrote data-oblivious algorithms for SVMs, matrix factorization, decision trees, neural networks and k-means clustering. Then, they ran oblivious and non-oblivious versions using SGX enclaves for decryption and processing. They validated the data-obliviousness by collecting traces at cache-line with Intel's Pin tool.

## A quick summary

Performance results

The following table (from the USENIX paper) shows the overhead when compared to a baseline the processes the data in plaintext without SGX. Both the oblivious and non-oblivious versions use SGX protection and encryption.

Algorithm| non-oblivious | data-oblivious
--- | --- | --- 
K-Means | 1.91 | 2.99
CNN | 1.01 | 1.03
SVM | 1.07 | 1.08
Matrix factorization | 1.07 | 115.00
Decision trees | 1.22 | 31.10

There are a few quick conclusions from this chart:
* using SGX and encryption doesn't add significant performance overhead
* some algorithms are already predominantly data-oblivious (CNN, SVM, K-Means)

One thing to note is that these are not state-of-the-art implementations, which might use data-dependent optimizations. But it's neat to see that you can harden machine learning against data leakage with much more reasonable overheads than previous work.

Microsoft is calling this their 'Always Encrypted capability.' I take issue with this, because it's not always encrypted--it's just that it's only decrypted and processed in an enclave. That's not to say I have any better suggestions...but I'm opposed to misleading nomenclature like this. I've heard *far* too many talks that claim end-to-end encryption via TLS to think that a phrase like 'Always Encrypted' is a good idea when decryption is called at some point. While secure enclaves are great tools, they [aren't](https://www.blackhat.com/docs/us-16/materials/us-16-Aumasson-SGX-Secure-Enclaves-In-Practice-Security-And-Crypto-Review.pdf) [infallable](https://arxiv.org/abs/1702.08719).

I admit that The Cloud isn't my favorite thing to talk about, mostly from dealing with people who think that it's a magical way to make everything work, but I really enjoyed this talk. I was intrigued by the machine learning examples cited and noticed that they've applied for a [patent](http://www.freepatentsonline.com/y2017/0372226.html) on the technique described in the talk. I foresee more similar papers joining my TODO stack

<a name="sgxnote">1</a>: SGX private memory is limited to registers

<a name="tsx">2</a>: in very brief pseudocode: `xbegin(); load_sensitive_code; load_sensitive_data; run_algorithm; xend();`