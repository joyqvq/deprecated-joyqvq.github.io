---
layout: post
title:  "Annoations and Discussions on Key Derivation"
date:   2022-12-29 20:00:00
categories: crypto
image: /assets/article_images/2022-12-29/keys.jpg
image2: /assets/article_images/2022-12-29/keys.jpg
---

# Annoations and Discussions on Key Derivation

This post is a more general discussion on key derivation applying to broader cryptocurrency usage. You can find my writing on the Sui Network specifically [here](https://tech.mystenlabs.com/cryptography-in-sui-wallet-specifications/). 

[BIP-32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) and [SLIP-0010](https://github.com/satoshilabs/slips/blob/master/slip-0010.md) as clear and excellent standards on *how* key management works, and they both have extensive implementations across many languages. However, I struggle to find a good reading on *why* some constructions are needed and *how* they relate to the RFCs on different curves. This post aims to discuss these two standards 

## BIP-32 Revisited, With Heavy Annotations

BIP-32 defines the Hierarchical Deterministic Wallet structure to logically associate a set of keys together. This is primarily motivated by the need to reduce the overhead of keeping track of a large number of private keys. BIP-32 is a standard that defines a tree structure such that a bunch of private keys can be *derived* from one single master private key. 

Why is this useful? A tree structure provides domain separations for a set keys. This way instead of securing multiple private keys, the user just needs to secure one master key. 

A common use case is for a custodian that need to associate distinct managed addresses for all of the users. By deriving a number addresses from the same master private key, each user can receive funds to his own address. Moreover, since the custodian has the master private key that can derive all the private keys, the custodian has the ability to move the funds that are deposited in any of the derived addresses. 

Specifically, BIP-32 defines three functions: 
1. **Master seed $\rightarrow$ master private key**: This defines the first master private key. By multiplying the base point in the field, the public key can be derived.
2. **Parent private key $\rightarrow$ Child private key**: This defines the induction step for the private key derivation. 
3. **Parent public key $\rightarrow$ Child public key**: This defines the induction step for the public key derivation. 

![image tooltip here](/assets/article_images/2022-12-29/bip32.png)

Now let's take a look at the inner workings of each functions. 

## Function 1: Master Seed $\rightarrow$ Master Private Key
$gen(seed) -> (sk_0, cc_0)$

1. Compute $I$.
`I = HMAC-SHA512(Key = "Bitcoin seed", Data = seed)`
2. Compute master private key.
`sk_0 = I_L` where $I_L$ is the first 32 bytes of $I$. 
3. Compute master chaincode.
`cc_0= I_R` where $I_R$ is the last 32 bytes of $I$. 

## Function 2: Parent Private Key $\rightarrow$ Child Private Key
$CKD_{sk}((sk_{parent}, cc_{parent}), i) \rightarrow (sk_i, cc_i)$

1. Compute $I$. 
- If $i$ is hardened ($i \geq 2^{31}$): `I = HMAC-SHA512(Key = cc_i, Data = 0x00 || sk_i || i)`. 
- If $i$ is normal: `I = HMAC-SHA512(Key = cc_i, Data = sk_i * G || i)`.

2. Compute child private key. 

    $sk_i = I_L + sk_{parent} \mod n$ where $I_L =$ the first 32 bytes of $I$. 

3. Compute child chaincode. 

    $cc_i = I_R$ where $I_R =$ the last 32 bytes of $I$

### Notes
- Why do we need chaincode? 

    Chaincode is used as the `key` for the HMAC function. This is to make the key derivation not only depend on the key itself but with extra entropy. We call a private key along with chaincode as extended private key `xpriv = (sk, cc)` and same for extended public key `xpub = (pk, cc)`. 

- What are the differences between "hardened" and "normal"?

    The hardened $I$ is calculated with the private key whereas the normal one is calculated with the public key (the result of $sk_i * G$). 

    The latter is not "hard" is becasue a parent extended private key can be derived using a parent extended public key and any non-hardened private key. (TODO add more)

- Why 0x00 padding when computing $I$? 

    This is an implementation detail to make the `Data` to hash the same length no matter $i$ is hardened or not. 
    
    $sk_i * G$ is a point on the elliptic curve $(x, y)$ where $x$ and $y$ are both 32 bytes. The compact serialization form of this point is 33 bytes where $y$ can be compressed to `0x02` or `0x03`. $x$ and 1-byte indicating $y$ can uniquely identify a point because there are only two $y$ corresponding to a $x$ that lies on the curve. 

## Parent public key $\rightarrow$ Child public key

$CKD_{pk}((pk_{parent}, c_{parent}), i) \rightarrow (pk_i, cc_i)$

1. Compute $I$. 
- If $i$ is hardened ($i \geq 2^{31}$), fail.
- If $i$ is normal, `I = HMAC-SHA512(Key = cc_{parent}, Data = pk_{parent} || i)`

2. Compute child public key. 
$pk_i = I_L * G + pk_{parent}$

3. Compute child chaincode. 
$cc_i = I_R$

### Notes

1. Why the derivation fails if $i$ is hardened? 

2. Why this is correct for non-hardened public key derivation? 

    Accoring to the key derivation for private and public key functions, we have: 

    $sk_i = I_L + sk_{parent}$

    $pk_i = I_L * G + pk_{parent}$

    The (+) here is an elliptic curve addition of two points. We can then derive:

    $pk_i = I_L * G + pk_{parent} = I_L * G + sk_{parent} * G = (I_L + sk_{parent}) * G = sk_i * G$

    Now we have $pk_i = sk_i * G$ for each child private/public key pairs. 

2. Watch-Only Wallet

    The ability to do public key derivation enables the use case of watch-only wallets. A common set up is to have an online service that can derive addresses for receiving payments and can view balances. The offline service keeps the private key and it only will be used to create transactions moving the funds out of any of the addresses since it can derive all the private keys that controls the funds.

## Key Generation for ECDSA vs Ed25519

Here, let's quickly recap how a public key is generated from a private key for two particular cryptosystems: Secp256k1 and Ed25519. This will help us understand the impact of key derivations for them. 

### Secp256k1 

The private key is a randomly generated scalar represented in 32 bytes. The public key is a point on the elliptic curve, using the scalar point multiplication of the private key. 

$pk = sk * G$ where $G$ is the generator point. 

### Ed25519

The private key is slightly different than just a randomly generated scalar. It first involves a preprocessing step to "hash then prune" a private key. We denote the private key before "hash and prune" as $sk$ and after as $sk_{prune}$. 

The seed is a randomly generated scalar represented in 32 bytes. 

The private key is pruned as follows:

1. Compute the SHA512 hash of 32-byte seed. The hash is 64 bytes. 

2. Take the first 32-bytes of the hash and prune by clearing the first three bit, the last bit is set. 

```
sk[0] &= 248  // 248 == 0b11111000
sk[31] &= 127 // 127 == 0b01111111
sk[31] |= 64  // 64  == 0b01000000
```

3. The public key is the scalar multiplication of the pruned private key. 

$pk = sk_{prune} * G$

See more on this in the latest Ed25519 [RFC](https://www.rfc-editor.org/rfc/rfc8032#section-5.1.5). 

### Notes
1. Why the private key for EdDSA needs to be pruned but not ECDSA?

    Unfornately there is no short answer here without explaining few concepts. 

    Secp256k1 is defined in a prime-order group. 

    For performance reasons, edwards25519 doesn't provide a prime-order group and provides a group of order $h * l$ instead, where h is a small co-factor (8) and $l$ is a 252-bit prime.

    This is to guarantee that the private key will always belong to the same subgroup of EC points on the curve and that the private key always have similar bit length. This results in a slight bias towards non-uniformity at one spectrum of the range of valid keys.

    TODO: Order of the Group, Cofactor

2. What does the pruning actually do? 

    (A paragraph for the curious mind, feel free to skip.)

    `scalar[0] &= 248  // 248 == 0b11111000
`
    This is making the number a multiple of 8, presumably due to the cofactor. Clearing the lowest three bits = clearing the cofactor.

    TODO

3. How does this impact key derivation defined in BIP-32?

    Now that we know there is a special requirement on the Ed25519 private key: it must be hash and pruned before usage. BIP-32 does not really enforce such requirement for the master private key specified in Function 1 (recall this is the first 32 bytes of `I = HMAC-SHA512(Key = "Bitcoin seed", Data = seed)`). 

    In addition, there is also no checks on the child private key derived from Function 2 $CKD_{sk}$ in BIP-32 would maintain this requirment. 

Now let's take a look at SLIP-0010, a generalized version of BIP-32 that is fully compatible with the existing BIP-32, but also provides a generalized specification to make Ed25519 key derivation work.

## A Generalized form of BIP-32: SLIP-0010

Similar to BIP-32, SLIP-0010 also defines the same three functions, with modifications to each one of them. 

**Function 1: Master seed $\rightarrow$ master private key**

First calculating $I$: `I = HMAC-SHA512(Key = Curve, Data = seed)`. Where Curve here is a domain separator string that can be "Bitcoin seed" (same as BIP-32) or "ed25519 seed". 

Same as BIP-32, the master private key $sk_0$ is the first 32 bytes of $I$ and the master chaincode $cc_0$ is the last 32 bytes. Here $sk$ is assumed to be before "hash and prune" to be always valid. 

**Function2: Parent private key $\rightarrow$ Child private key**

While BIP-32 allows for normal and hardened child private key derivation, SLIP-0010 only allows hardened $i$ here. This is because $pk_i = sk_i * G$ no long holds for a non-hardened $i$. We would have to "hash and prune" $sk_i$ first to be able to compute $pk_i = sk_{prune} * G$.

1. Compute $I$. 
- If $i$ is hardened ($i \geq 2^{31}$): `I = HMAC-SHA512(Key = cc_i, Data = 0x00 || sk_i || i)`. 
- If $i$ is normal: Fail. 

2. (Unchanged) Compute child private key and child chaincode. 

    $sk_i = I_L + sk_{parent} \mod n$ where $I_L =I[..32]$

    $cc_i = I_R$ where $I_R =I[32..]$

**Function 3: Parent public key $\rightarrow$ Child public key**

This step is disallowed. Because the derived public key would not match with the derived private key which assumes it is not "hash and prune"-ed. 