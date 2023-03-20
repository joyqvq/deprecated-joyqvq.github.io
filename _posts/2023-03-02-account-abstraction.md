---
layout: post
title:  "Account Abstraction: Alternatives to Private Key Wallets"
date:   2023-03-02 20:00:00
categories: crypto
image: /assets/article_images/2023-01-19/threshold.jpg
image2: /assets/article_images/2023-01-19/threshold.jpg
---
## A practical technical review on account abstractions and its implementation. 

Account abstraction is a recently deloyed feature on Ethereum network. Previously, funds can only be spent by providing a valid signature over the transaction data. Under the private key wallet model, a digital signature attests as a witness for a transaction to be valid, and it is considered nonrepudiable and final. 

However, such authentication method can considered as one of the barriers to enter when onboarding user given the key handling intracacies, and it is rather restrictive for new use cases of on-chain activities. 

What if we allow _any_ authentication method for the user to define when executing a transaction? Account abstraction is proposed as a more generalized authentication model for transactions which allow for greater flexibility to evaulate transaction validity. In other word, we "abstracted" the authentication method for an account. 

In this essay we will explore the goal of account abstraction, main use cases, the architecture of the smart contracts and the user flow to interact with it. 

## Goal: Transaction Validity Extention

When a user interacts with a blockchain, they need to submit a transaction to alter the state machine on chain. Before account abstraction, transaction validity is defined rigidly by the protocol: an ECDSA signature, a simple nonce, and an sufficient account balance that can pay for the operation. If all these validity checks pass, the transaction can be accepted to the mempool and subsequently executed by the VM immediately, which alters the balance or other states of the state machine.

To demonstrate a simple ERC-20 balance transafer from an Externally Owned Account (EOA), user must first package the call data, a valid nonce and into one transaction originating from the EOA, and provide a valid signature for execution. In other word, a transaction verification is tightly coupled with execution.

```shell
curl eth_sendRawTransaction
assert!(owner == sender);
```

An early [essay](https://blog.ethereum.org/2015/07/05/on-abstraction) by Vitalik proposed the following idea:

> Every account should have a piece of "verification code" attached to it.

A transaction can be spent from an account as long as the verification code that the account defines passes, the execution can happen in the state machine. In other word, the verification check can be custom defined and be completely decoupled from execution. The verification logic is "abstract" away. 

A very simple verification logic can be defined the following: 

1. User generates a verification code offline and uses the hash of the verification code as an address.

2. When the user wants to spend funds from the address, user provides the preimage. 

3. The authentication method is evaluated to true if hash of the preimage matches with the address, the transation is valid and the fund can be spent. 

This somewhat ressembles to the Pay-To-Script-Hash model in Bitcoin: As long as the user provides a script matching the script hash and data, the script is evaluated to true by the state machine and will be executed right away. 

## Architecture Overview

Now we understand the motivation for account abstraction (now we abbreviated to AA), we can sketch out the necessary components to enable AA-enabled wallet to execute transactions with its customized verification logic. 

1. Define User Operations

Since now we do not want to constrain the transaction validity check to signature verification, we extend the definition of a transaction data. To differentiate, we call a transaction here a _user operation_ in the context of AA. A user operation is defined with some extra fields that are necessary for AA , and now let's compare it with transaction data. 

```
| Fields | User Operation | Transaction | Usage | 
| -------| -------------- | ------------ | ------ | 
| sender | x| | | 
| nonce | x| | | 
| initCode | x| | | 
| callData | x| | | 
| callGasLimit | x| | | 
| verificationGas | x| | | 
| preVerificationGas | x| | | 
| maxFeePerGas | x| | | 
| maxPriorityFeePerGas | x| | | 
| paymasterAndData | x| | | 
| signature | x| | | 
```

2. Define Alt Mempool for User Operations

Unlike a transaction, a user operation decouples its verification with its execution. Therefore, the evaluation flow to accept it to a mempool requires a slightly different treatment. 

The first iteration of the account abstraction proposal defines a new OP_CODE to signify the VM for a special execution flow. An earlier prposal for AA called [EIP-2938](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2938.md) proposes for a new transaction type called AA_TX_TYPE. 

Because of its requirment for a soft fork upgrade, it is considered less desirable to implement. On the other hand, the accepted EIP-4337 defined an alternative mempool that only accepts user operations and requires no consensus change.
 
3. Define Bundler 
Bundlers are introduced as a different actor interacting with the alt mempool. Its responsibility is to poll user operations, and call each AA-enabled wallet contract to run its custom verification logic with an verification cost, and decide whethere the validity check passes. If the result is successful, the bundler organizes the user operations into a _bundled_ transaction and submits for the main mempool for immediate execution.

On a high level, the role of the bundlers is very similar to the block builders that operates along the main mempool to build blocks. Their economic incentives are aligned in the _verification_ fee market, rather than the _execution_ fee market. Similarly, bundlers stake `todo`

A reference implementation and spec is provided [here](). 

### Smart Contract Interfaces

1. User Wallet contract
Any AA-enabled wallet has to implement the following interface for it to be considered in the alt mempool. 

- `validateUserOp`: This call validates the signature field from user operation (if any), and validate and increment the nonce.

2. Entrypoint contract

In order to interact with the alt mempool, the bundler invokes the entrypoint contract to perform corresponding verification checks for user opertations it wants to bundle. An entrypoint contract is defined as a global singleton that is deployed once on the blockchain. 

It has the following key interfaces: 

- `handleOps`: This function defines two functionalities, one for verification and one for execution.
    - The verification loop first checks for `initData` in the user operation, if it is present, first creates the wallet. Otherwise, it invokes the wallet contract's `validateUserOp`, which runs the authentication logic and pay for the verification fee.
    - The execution loop invokes each AA-enabled wallet contract to execute `callData`. 
- `simulateHandleOp`: This call simulates the user operation.  

### Transaction Lifecycle

Now we have defined all the players involved and a few on-chain components to make account abstract happen, let's put them together:

1. User defines its wallet contract according to the AA-enabled account interface. 
2. User submits its user operation to the alt mempool with using RPC endpoint "eth_sendUserOperation".
3. Bundler calls handleops, which calls each AA wallet account to validate user operation. 
4. 

eth_calluserOperation

## Unlocking New Use Cases

Since account abstraction is a rather generalized idea for arbitrary verification logic, it provides a nice umbrella model defining several compelling use cases. 

1. Signature Abstraction

Now that the verification can be ran according to any custom definition, as long as the signature verification can be compiled to a smart contract onchain, we can invoke it as the verification logic. Instead of relying on the de-facto ECDSA signature scheme defined for an EOA, more signature schemes are unlocked for an AA-enabled account. For example, the BLS signature and post-quantum signature gated account can be quickly introduced to Ethereum without a protocol upgrade, since AA-based accounts can quickly replace its verification logic on-chain.

With the helper function defined by [EIP-1271] (https://eips.ethereum.org/EIPS/eip-1271), we can verify whether a signature on a behalf of any given contract is valid.

2. Permission Abstraction

With account abstraction, defining more complex spending policy logic on-chain becomes more transparent and intuitive.

- Multisig: For organizations that need a `k-of-n` threshold approval setup, we can implement a AA-enabled wallet contract with `validateUserOp` implemented as follows:

```
todo
```

- Multi-Factor Authentication: To take one step further from multisig, we do not even need multiple signatures from EOAs for authentication. Instead, other authentication factors such as Face ID computed from Apple device enclave, session tokens from OAuth flow and even other users' authentication data can defined as part of the spending policy. 

For example, we can define a `validateUserOp` that needs to satisfy one of the policies to spend out of my AA-enabled account. 

1. A valid Apple enclave Secp256r1 signature generated by my Face ID is required. 
2. My friend Alice's Facebook login credential and my other friend Bob's BLS signature is required. 

What `validateUserOp` looks like:

```
todo
```

What spending path can the user take: 

```
todo
```

3. Gas Abstraction
Previously, any transaction spent from the EOA account is required to pay gas amount in ETH. This means a user needs always prefund the account with sufficient ETH for any on-chain activities. 

To relax such strenous requirement for some transactions for a smoother user experience, account abstraction model enables the following gas payment scenarios: 

1. Sponsored transaction: Gas can be paid by a third party that is not the spending user.

However, this introduces a slightly different transaction lifecycle.

2. Any token gas: Gas can be paid in any ERC-20 tokens instead of Ethereum

## Deep Dive on Multisig Account Abstraction

As we covered briefly earlier, multisig is one of the compelling use cases enabled by account abstraction. As many of you may be used the existing multisig application e.g. Gnosis Safe on Ethereum, what more account abstraction brings to the table? First, a quick recap on how Gnosis Safe works on chain right now. 

How does the transaction lifecycle look like if we initialize Gnosis Safe as an AA-enabled wallet contract? 
- create2
- 

## Safety Considerations

DoS
## Concluding Thoughts

In the author's opinion, account abstraction is an exciting direction for Ethereum and many other blockchains to reach mass adpotion and unleash its full potential for generalized decentralized computing. It leverages the on-chain programs to ensure verification integrity in addition to execution integrity.

Fundamentally speaking, this simiplies the responsibility of wallet softwares and would increase the participation of on-chain activities. 

There are many ideas and flavors of account abstractions being worked on, and here are some I find interesting:
1. [Lit Protocol](https://developer.litprotocol.com/coreConcepts/accessControl/intro)
2. Squad Protocol on Solana
3. Starknet: https://community.starknet.io/t/starknet-account-abstraction-model-part-1/781
4. ZkSync: https://matterlabs.medium.com/introducing-account-abstraction-l2-l1-messaging-and-more-760282cb31a7

Hope you enjoy the read! I can be reached at @joyqvq on Twitter. 

## References
1. On Abstraction: https://blog.ethereum.org/2015/07/05/on-abstraction
2. EIP-2938: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2938.md
3. https://ethereum-magicians.org/t/implementing-account-abstraction-as-part-of-eth1-x/4020
