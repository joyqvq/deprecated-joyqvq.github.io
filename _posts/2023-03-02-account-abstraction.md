---
layout: post
title:  "Account Abstraction: Alternatives to Private Key Wallets"
date:   2023-03-02 20:00:00
categories: crypto
image: /assets/article_images/2023-03-02/accounts.jpg
image2: /assets/article_images/2023-03-02/accounts.jpg
---
## A practical technical review on account abstractions and its implementation.

Account abstraction is a recently deployed feature on Ethereum network. Previously, funds can only be spent by providing a valid signature over the transaction data. Under the private key wallet model, a digital signature attests as a witness for a transaction to be valid, and it is considered non-repudiable and final.

However, such authentication methods can be considered as one of the barriers to entry when onboarding users given the key handling intricacies, and it is rather restrictive for new use cases of on-chain activities.

What if we allow _any_ authentication method for the user to define when executing a transaction? Account abstraction is proposed as a more generalized authentication model for transactions which allow for greater flexibility to evaluate transaction validity. In other words, we "abstracted" the authentication method for an account.

In this essay we will explore the goal of account abstraction, main use cases, the architecture of the smart contracts and the user flow to interact with it.

## Goal: Transaction Validity Extension

When a user interacts with a blockchain, they need to submit a transaction to alter the state machine on-chain. Before account abstraction, transaction validity is defined rigidly by the protocol: an ECDSA signature, a simple nonce, and a sufficient account balance that can pay for the operation. If all these validity checks pass, the transaction can be accepted to the mempool and subsequently executed by the VM immediately, which alters the balance or other states of the state machine.

To demonstrate a simple ERC-20 balance transfer from an Externally Owned Account (EOA), users must first package the call data, a valid nonce and into one transaction originating from the EOA, and provide a valid signature for execution. In other words, a transaction verification is tightly coupled with execution.

An early [essay](https://blog.ethereum.org/2015/07/05/on-abstraction) by Vitalik proposed the following idea:

> Every account should have a piece of "verification code" attached to it.


A transaction can be spent from an account as long as the verification code that the account defines passes, the execution can happen in the state machine. In other words, the verification check can be custom defined and be completely decoupled from execution. As a result, the verification logic including the signature verification, gas payment and replay detection is "abstract" away from the core protocol and it is now loaded and enforced by the EVM.

A very simple verification logic can be defined the following:

1. User generates a verification code offline and uses the hash of the verification code as an address.

2. When the user wants to spend funds from the address, the user provides the preimage.

3. The authentication method is evaluated to be true if the hash of the preimage matches with the address, the transaction is valid and the fund can be spent.

This somewhat resembles the Pay-To-Script-Hash model in Bitcoin: As long as the user provides a script matching the script hash and data, the script is evaluated to true by the state machine and will be executed right away.

## Architecture Overview

Now we understand the motivation for account abstraction (now we abbreviate AA), we can sketch out the necessary components to enable AA-enabled wallets to execute transactions with its customized verification logic.

- Define User Operations

Since now we do not want to constrain the transaction validity check to signature verification, we extend the definition of a transaction data. To differentiate, we call a transaction here a _user operation_ in the context of AA. A user operation is defined with some extra fields that are necessary for AA , and now let's compare it with transaction data.

| Fields | User Operation | Normal Transaction |
| -------| -------------- | ------------ |
| `sender` | The account making the operation. | The address of the sender that is signing the transaction has to be an EOA account. |
| `recipient` | N/A | The receiving address for transferring value if it is an EOA; the contract address that will execute the code. |
| `nonce` | A sequentially incrementing counter for every operation for replay attack. | Same effect, increment for each transaction. |
| `value` | N/A | Value transferred. |
| `initCode` | Code needed to create an account on-chain for the first time. | N/A |
| `callData` | The data to pass to the sender during the main execution. | Optional field for arbitrary data. |
| `callGasLimit` | The amount of gas to allocate during the main execution call. | Same. |
| `verificationGas` | The amount of gas to allocate for the verification step. | N/A |
| `preVerificationGas` | The amount of gas to allocate for the bundler for pre-verification. | N/A |
| `maxFeePerGas` | Max fee willing to be paid per gas. See [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559). | Same |
| `maxPriorityFeePerGas` | Max priority fee to be paid per gas. See [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559). | Same |
| `paymasterAndData` | Address of the paymaster sponsoring the transaction. | N/A |
| `signature` | Data passed along with nonce during the verification step. | A ECDSA signature committed to the transaction data by the sender's private key. |

- Define Alt Mempool for User Operations

Unlike a transaction, a user operation decouples its verification with its execution. Therefore, the evaluation flow to accept it to a mempool requires a slightly different treatment.

The first iteration of the account abstraction proposal defines a new OP_CODE to signify the VM for a special execution flow. An earlier [EIP-2938](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2938.md) proposes a new transaction type called AA_TX_TYPE.

Because this adds two additional OP_CODEs to the protocol, it is considered less desirable to implement compared to the accepted EIP-4337, which defines an alternative mempool that only accepts user operations and requires no consensus change.

![Mempool vs. Alt Mempool](/assets/article_images/2023-03-02/mempool.jpg)

- Define Bundler

Bundlers are introduced as a different actor interacting with the alt mempool. Its responsibility is to poll user operations, and call each AA-enabled wallet contract to run its custom verification logic with an verification cost, and decide whether the validity check passes. If the result is successful, the bundler organizes the user operations into a _bundled_ transaction and submits for the main mempool for immediate execution.

A bundler monitors the alt mempool for user operations and once a threshold is met, it signs a single transaction with its private key. A reference implementation and spec is provided [here](https://github.com/eth-infinitism/bundler).

The role of the bundlers is very similar to the block builders that operate along the main mempool to build blocks. Their economic incentives are aligned in the _verification_ fee market, rather than the _execution_ fee market. Similarly, bundlers are required to stake on the network to ensure they are rewarded for behaving honestly and are slashed otherwise.

### Smart Contract Interfaces

- User Wallet contract

Any AA-enabled wallet has to implement the following interface for it to be considered in the alt mempool.

`validateUserOp`: This call validates the signature field from user operation (if any), and validates and increments the nonce.

- Entrypoint contract

In order to interact with the alt mempool, the bundler invokes the entrypoint contract to perform corresponding verification checks for user operations it wants to bundle. An entrypoint contract is defined as a global singleton that is deployed once on the blockchain.

It has the following key interfaces:
- `handleOps`: This function defines two functionalities, one for verification and one for execution.
   - The verification loop first checks for `initData` in the user operation, if it is present, first creates the wallet. Otherwise, it invokes the wallet contract's `validateUserOp`, which runs the authentication logic and pays for the verification fee.
   - The execution loop invokes each AA-enabled wallet contract to execute `callData`.
- `simulateHandleOp`: This call simulates the user operation. 
![Architecture Overview](/assets/article_images/2023-03-02/tx.jpg)

### Transaction Lifecycle

Now we have defined all the players involved and a few on-chain components to make account abstract happen, let's put them together:

1. User defines its wallet contract according to the AA-enabled account interface.
2. User submits its user operation to the alt mempool by using the RPC endpoint `eth_sendUserOperation`.
3. The validation manager for the bundler manager calls `Entrypoint.simulateValidation` in the entrypoint contract, which would either create new AA wallet accounts or call each AA wallet account to validate user operation. If valid, the user operation is added to the alt mempool.
4. The bundler scans for user ops in alt mempool and invokes `Entrypoint.handleOps` which validates and executes the calldata of each user operation. If a call to handle user operations is successful, the transaction is submitted via RPC endpoint "eth_sendRawTransaction".
## Unlocking New Use Cases

Since account abstraction is a rather generalized idea for arbitrary verification logic, it provides a nice umbrella model defining several compelling use cases.

- Signature Abstraction

Now that the verification can be run according to any custom definition, as long as the signature verification can be compiled to a smart contract on-chain, we can invoke it as the verification logic. Instead of relying on the de-facto ECDSA signature scheme defined for an EOA, more signature schemes are unlocked for an AA-enabled account. For example, the BLS signature and post-quantum signature gated account can be quickly introduced to Ethereum without a protocol upgrade, since AA-based accounts can quickly replace its verification logic on-chain.

With the helper function defined by [EIP-1271](https://eips.ethereum.org/EIPS/eip-1271), we can verify whether a signature on behalf of any given contract is valid.

- Permission Abstraction

With account abstraction, defining more complex spending policy logic on-chain becomes more transparent and intuitive.

1. *Multisig*: For organizations that need a k-of-n threshold approval setup, we can implement a AA-enabled wallet contract with validateUserOp.

2. *Multi-Factor Authentication*: To take one step further from multisig, we do not even need multiple signatures from EOAs for authentication. Instead, other authentication factors such as Face ID computed from Apple device enclave, session tokens from OAuth flow and even other users' authentication data can be defined as part of the spending policy. For example, we can define a `validateUserOp` that needs to satisfy one of the policies to spend out of my AA-enabled account: A valid Apple enclave Secp256r1 signature generated by my Face ID is required; My friend Alice's Facebook login credential and my other friend Bob's ECDSA signature is required.
    
3. *Gas Abstraction*: Previously, any transaction spent from the EOA account is required to pay gas amount in ETH. This means a user needs to always pre-fund the account with sufficient ETH for any on-chain activities.

To relax such strenuous requirement for some transactions for a smoother user experience, account abstraction model enables (a) Sponsored transaction: Gas can be paid by a third party that is not the spending user; (b) Any token gas: Gas can be paid in any ERC-20 tokens instead of Ethereum.

- Atomic Multi-Operation

Another use case for account abstraction is to have multiple user operations to require multiple user operations to validate. Here we introduce a new smart contract called an Aggregator contract that implements the following methods:

1. `validateSignatures(userOps, signature)`: This takes in a list of user ops, reverts if the aggregated signature is invalid.

2. `validateUserOpSignature(userOp)`: This is a _helper_ function that takes in a single user operation, returns a signature to be supplied in the User Operation `signature` field.

3. `aggregateSignatures(userOps)`: This is a _helper_ function that returns the aggregated signature from individual signatures from each user operation.

We then extend the Entrypoint contract to also include `EntryPoint.handleAggregatedOps` that calls `Aggregator.validateSignatures`. If an account returns an aggregator during `Entrypoint.simulateValidation`, `handleAggregatedOps` is used instead of `handleOps`

![AA Account with Aggregator](/assets/article_images/2023-03-02/aggregator.jpg)
## Concluding Thoughts

In the author's opinion, account abstraction is an exciting direction for Ethereum and many other blockchains to reach mass adoption and unleash its full potential for generalized decentralized computing. It leverages the on-chain programs to ensure verification integrity in addition to execution integrity.

Fundamentally speaking, this simplifies the responsibility of wallet softwares and would increase the participation of on-chain activities.

There are many ideas and flavors of account abstractions being worked on, and here are some I find interesting:
1. [Squad Protocol](https://squads.so/blog/what-is-account-abstraction-ethereum-vs-solana) highlights the account model on Solana that enables programs to manage other accounts using a Program derived addresses.  
2. [Starknet](https://community.starknet.io/t/starknet-account-abstraction-model-part-1/781) implements abstract accounts in StarkOS, very similar to EIP-4337. 
4. [ZkSync](https://era.zksync.io/docs/dev/developer-guides/aa.html#prerequisites) implements 
account abstraction with few extensions. 

Hope you enjoy the read! I can be reached at @joyqvq on Twitter.

## References
1. [On Abstraction](https://blog.ethereum.org/2015/07/05/on-abstraction)
2. [EIP-4337](https://eips.ethereum.org/EIPS/eip-4337)
3. [EIP-2938](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2938.md)
4. [EIP-4337 discussion](https://ethereum-magicians.org/t/implementing-account-abstraction-as-part-of-eth1-x/4020)