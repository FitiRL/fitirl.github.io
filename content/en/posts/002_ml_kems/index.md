---
title: "Embracing ML-KEMs in the Age of Post-Quantum Cryptography"
date: 2025-06-29
author: Fiti
description: Simple introduction to post quantum cryptography in shared secrets
tags:
  - cryptography
---

## Introduction

Diffie-Hellman, Elliptic Curve Diffie-Hellman (ECDH), RSA... ok, we all know them. They can be effectively used today to share a secret key between two parties where the channel of communication is public. Soon, these algorithms will be broken with the recent new quantum computers.

These algorithms are currently safe because they rely on hard mathematical problems that computers are currently unable to solve. Or better said: computers are not able to solve them in _reasonable_ time. There are algorithms that found the solution for them. The Shor Algorithm was proposed in 1994 and, in the case of it being correctly implemented in a quantum computer, it will be able to solve problems like the factoring problem, the discrete logarithm problem and period-finding problem.

>Example: Being $b^x = a$, then $x = \log_b(a)$. In RSA, arithmetic modulo is used in combination with this principle to ensure that if a message is ciphered by a key $k$, it is only possible to decipher it with another key $k^\prime$, taking into account that $k^\prime$ is the modular inverse of $k \mod \phi(n)$, where $\phi(n)$ is Euler's totient function.

Continuing with the example of the discrete logarithm, there is no algorithm able to solve this problem in polynomial time (for example, the Pohlig-Hellman algorithm _can_ be efficient if the group order has only small prime factors, but in the worst case it can be exponential). However, the quantum algorithm of Peter Shor would be able to solve it in polynomial time, compromising what we currently use.

## New age for key sharing: ML-KEMs to the rescue!

Post-quantum cryptography (PQC) proposes new algorithms that are based on mathematical problems that are hard to solve even for quantum computers. They are based on other new principles like lattices. In this article, I am going to talk about CRYSTALS-Kyber, which has evolved into the final standardized FIPS 203, Module Lattice-Based Key Encapsulation Mechanism (ML-KEM). This mechanism allows the sharing of a secret key between two participants in a similar way to what Diffie-Hellman does. It won the NIST competition (where many algorithms were presented) and has been accepted as secure (you know, for now). Let's see what is hiding behind this scheme.

> Note that even if ML-KEMS is for Key Encapsulations, ML-DSA is for Digital Signature and will not be covered in this article.
### ML-KEMs: How it works

Under the hood, ML-KEMs is composed of three sub-algorithms:

- Key generation
- Encapsulation
- Decapsulation

> Difference with Diffie-Hellman (DH): In the case of the Diffie-Hellman shared key algorithm, where the public and private keys were "merged" to obtain the shared key by both ends and only public keys were sent, the KEM mechanism sends the symmetric key encapsulated with the public key, and when it is received at the end, it is decapsulated with the private key.

The key generation is done by generating a Matrix $A$ of $k \times k$ being $k$ the dimension matrix. Then we have 
$$A\in (\mathbb{Z} _q ^{256})^{k \times k}$$
Now, compute the secret vector $s$:

$$s \in (\mathbb{Z}_q^{256})^k$$
And compute the error vector $e$:

$$e \in (\mathbb{Z}_q^{256})^k$$
Note that $s$ and $e$ are sampled from noised distribution (in other words, it consists in a pseudorandom sample of distribution $D_\eta (R_q)$).

Adding the calculated noise results in:

$$t = A_s + e$$And $s$ remains as the secret key.

Once parameters have been generated $(s, t)$ we have the key and we need to generate the encapsulation key:

First, we need to generate the random vector $r$ and two sample error vector $e_1$ and $e_2$:

$$r \in (\mathbb{Z}_q^{256})^k$$
$$e_1 \in (\mathbb{Z}_q^{256})^k$$
$$e_1 \in (\mathbb{Z}_q^{256})^k$$
> Note that these generations are done again using the centered binomial distribution and a pseudorandom value.

Then compute

$$u=A^t\cdot r + e_1$$
To finally encapsulate the shared secret:

$$v = t^t\cdot r + e_2 + encode(m)$$

As the final step, the **decapsulation** , we can compute:

$$w = v\cdot s^t u$$ To obtain m as: 
$$m = decode(w)$$

The algorithm's security is based on the Learning With Errors (LWE) problem. In essence, because of the lattice structure and added noise, it's computationally infeasible to distinguish the ciphertext from random data without knowledge of the secret $s$.

## What to do about this?

There's an emerging risk: attackers are already leveraging "Harvest now, decrypt later" strategies, waiting for this moment (Y2Q) to happen. Examples of these real risks are that Apple and Google have recently announced the use of hybrid encryption in applications like iMessage.

> Hybrid encryption combines post-quantum and non-post-quantum cryptographic schemes. This is done because there is a high probability that new algorithms may be discovered to be vulnerable, as they are quite recent. To mitigate this risk, the non-post-quantum scheme would protect the data even if the post-quantum algorithm is found to be vulnerable.

Adopting post-quantum cryptography (either fully or as part of a hybrid approach) is a smart move. Eventually, quantum advances will render RSA and ECC obsolete, and what's currently unbreakable could become trivially solvable.

### And in the embedded world?

Recently, some libraries like `WolfSSL` have announced that they support post-quantum cryptographic solutions for embedded devices. PQC requires more resources (for key storage, for example). This new age represents a new challenge for embedded devices, especially due to limited resources and computing power constraints.

## Conclusion

In this article, the motivation behind the ML-KEM key sharing scheme is described as an alternative to secure Diffie-Hellman when quantum computers arrive. A very summarized version of ML-KEM key generation, encapsulation, and decapsulation is shown. Some details have been omitted for simplicity. If you would like to know more details about this scheme, you can check the `FIPS 203` document.

