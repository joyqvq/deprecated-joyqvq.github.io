---
layout: post
title:  "All Things Commitments"
date:   2022-12-05 20:00:00
categories: crypto
image: /assets/article_images/2022-12-08/commitment.jpg
image2: /assets/article_images/2022-12-08/commitment.jpg
---

# Examining vector and polynomial commitments.

"A promise is made, a promise is kept." How do we make commitments? 

Making commitments is a two-step process. First, we make a promise: a committer publishes a value and it binds to the message without revealing it.

How do we know a promise is kept? Second, he can choose to reveal the message to a verifier, who can check if the message is indeed what he previously published. 

An obvious commitment scheme that may come to mind: Given a collision-free hash function, the commitment can be $Com(m) = H(m)$. Later we can verify the hash is indeed what the message hashed to.

Alternatively, some of you may recall the security of the discrete log problem: It is computationally impossible to find $m$ given $g^m$ where $g$ is a generator of a group $G$ of prime order $p$. Here we've got another commitment scheme: $Com(m) = g^m$. The verifier can later check if the commitment is indeed $g^m$. Let's call this $Com_{DL}$.

These two are both commitment schemes that satisfy the following important properties:

1. It is **hiding**: The probability to learn $m$ from $Com(m)$ is negligible. 
2. It is **binding**: The probability that the commitment on two different messages being the same is negligible. That is, $Com(m) = Com(m')$.[^1]

## Pedersen Commitment

Sometimes the $m$ we are committed to can be a small value in a range, which makes brutal force $Com_DL = g^m$ to find $m$ not too hard. Here we can improve $Com_{DL}$ to Pedersen commitment a randomization: 
1. The committer publishes $Com(m) = g^m h^r$. Here, $r$ is a random value (or it is called a blinding factor) . $g$ and $h$ are two random generators of group $G$. 
2. Later, the committer reveals $m$ and $r$, and the verifier simply needs to check if $C$ equals to $g^m h^r$. 

### Example: Confidential Transaction

This commitment scheme is simple and powerful with its notable use case in confidential UTXO transactions. Here the message the commitment binds to is the transaction amount. Without revealing the amount, the committer just needs to show commitments to each inputs and output amount. This is due to these properties:

For an UTXO transaction, we have: 

$\sum inputs = \sum outputs$

For two commitments, we have additive homomorphism:

$Com(m_1, r_1) \cdot Com (m_2, r_2) = g^{m_1} \cdot g^{r_1} \cdot g^{m_2} \cdot g^{r_2} = Com(m_1 + m_2, r_1 + r_2)$

Here we see that without revealing the amount that the user committed to, anyone can verify the validity of the transaction by checking the commitment to input amounts equals the commitment to output amounts.

## From One to Many Commitments

Now we generalize that we have more than one value to commit to, say $m_0, ..., m_n$. Using the previous commitment schemes, we make two observations:
1. A commitment to a vector of messages now has size $O(n)$ where $n$ is the number of the messages.
2. To open the commitment, the committer needs to reveal all messages of size $O(n)$.

## Merkle Hash Tree

A naive improvement to the size of the commitment is to take the hash of all commitments: 

$H(Com(m_1) \parallel Com(m_2)... \parallel Com(m_n))$

This allows multiple value committed to one value, but to verify the commitment, the committer needs to reveal $O(n)$ messages.

A Merkle hash tree is a nice trick to reduce the number of messages to be revealed from $O(n)$ to $O(\log n)$. For a vector of messages, we construct a tree such that the leaf nodes are stored with $m_1...m_n$, the intermediate node is stored with the hash of the left and right intermediate value, $H(L \parallel R)$ [^2].

A commitment for $m_i$ is defined as a path consisting of any node that shares the same parent with the node on the path till $m_i$ is reached. Note that the depth of the tree is the length of the path, therefore the commitment size is $O(\log n)$. 

To formalize what we just said in algorithms, a Merkle tree commitment scheme is defined as:

1. $Construct(m_1...m_n) \rightarrow tree$: This constructs the Merkle tree with all leaves as messages. Alternatively, this can be done incrementally with $Add(m_i, tree) \rightarrow tree$. 
2. $Com(m_i, tree) \rightarrow \pi_i$: The proof which is the path by all the sister nodes of the parent node, represents the commitment that the message is indeed included in the root. 
3. $Verify(root, m_i, \pi_i) \rightarrow \{0, 1\}$: Given the root of the Merkle tree, the revealed value $m_i$ and the proof $\pi_i$, the verifier can check if the value is indeed committed to the tree. 

### Example: Blockchain Light Client
For each transaction in the Bitcoin blockchain, it is a commitment to a block. A block is in fact a Merkle tree where each leaf node stores the transaction ID and the root is a block hash. Instead of downloading all transaction IDs in all blocks to verify a transaction indeed exists, a light client just needs to download all block hashes. 

For a light client to verify if the transaction exists in the blockchain chain, it can query a full node, and gets back the block hash and it's the commitment (represented as the path from the transaction ID to the Merkle root) and run $Verify(root, m_i, \pi_i)$. 

This construction gives its name "light" client, since the proof of membership can be verified against a much smaller amount of data (block headers). 

## From Vector to Polynomial Commitment
Merkle tree provides some improvements to commitment size, how about the hiding property? Do we need to reveal _all_ messages $m_1...m_n$ to reveal the commitment? 

Here we want to introduce polynomial commitment, which similar to the vector commitment we discussed earlier, are binding and hiding. But unlike verifying a vector commitment that requires the entire vector to be revealed, verifying a polynomial commitment turns out to be less demanding.

We first transform a vector $(m_1...m_n)$ to a polynomial $f(x) = k_0 + k_1x + k_2x^2 +...+k_nx^n$ such that $f(i) = m_i$ using [Lagrange interpolation](https://en.wikipedia.org/wiki/Polynomial_interpolation). While the mathematical detail is out of scope here, a result of this transformation is: a commitment to $m_i$ is really a commitment to the evaluation of $f(x)$ at $x=i$, $com(m_i) = com(f(i))$. 

Since now the vector has turned into a polynomial, we now show a polynomial commitment scheme where the verification step does not require revealing the polynomial itself. 

## KZG Polynomial Commitment

#### Step 0: Trusted Setup

$Setup \rightarrow PK$ where $PK = (g, g^\tau, g^{\tau^2}...g^{\tau^n})$

We first set up a list of public values that are available both for the committer and the verifier. We call these values **common reference string**. They are a bunch of precomputed elliptic curve points $g, g^\tau, g^{\tau^2}...g^{\tau^n}$ based on a secret value $\tau$ that is only known at this step. We call $\tau$ a toxic waste that neither the committer nor the verifier should know later on.

#### Step 1: Commit to a polynomial

$Com(f(x)) \rightarrow C$

For a given polynomial $f(X) = \sum_{i=0}^n k_i x^i$, and these public values $g, g^\tau, g^{\tau^2}...g^{\tau^n}$. We show that the commitment can be calculated as $g^{f(x)}$:

$C = g^{f(x)} = g^{k_0 + k_1 \cdot x + ... + k_n \cdot x^n}$

$C = g^{k_0} \cdot g^{k_1 \cdot x} ... \cdot g^{k_n \cdot x^n}$

$C = g^{k_0} \cdot (g^x)^{k_1}...\cdot (g^{x^n})^{k_n} = \prod_{i=0}^{n} (g^{x^i})^{k_i}$

Now we plug in $x = \tau$, we have the commitment to a secret value $\tau$ without using $\tau$: $C = \prod_{i=0}^{n} (g^{\tau^i})^{k_i}$. Since $\tau$ is not known to anyone, we simply call this the commitment to the polynomial itself.

### Step 2: Commit to $f(m_i)$
$Com(m_i) \rightarrow (i, f(i), w_i)$

Recall the transformation from vector commitment, we just need to show the commitment to the evaluation of $f(x)$ at $i$ instead of $m_i$.

First, let's define $\psi_i(x)$ as a quotient polynomial:

$\psi_i(x) = \dfrac{f(x) - z}{x-i}$

A significance of this polynomial is that it only exists if and only if $f(i) = z$. In other word, the committer indeed knows the evaulation at $i$ to be $f(i)=z$ to be able to construct such polynomial.

Then we construct the witness $w_i$ similarly by raising it to the power. With the "if you know you know" polynomial $\psi_i(x)$ and the public values from Step 0 (Again, without know $\tau$ itself), we can compute easily:

$w_i = g^{\psi_i(\tau)}$

Now the committer only needs to publish three values, note that the polynomial itself $f(x)$ is not revealed:

- $i$: The value that the polynomial is evaluated at.
- $z$: What the committer claims the evaluation of the polynomial at $i$ aka $f(i)$ equals to.
- $w_i$: A value can only be "witnessed" because $f(i)$ exists!

#### Step 3: Verify the commitment

$Verify(PK, C, i, f(i), w_i) \rightarrow \{0, 1\}$

The verification is quite simple, we just need to check if $e(C, g) == e(w_i, g^{\tau} / g^{i}) \cdot e(g, g)^{f(i)}$. Why does this hold? 

First, we introduce the notation $e(\cdot, \cdot)$ as a bilinear mapping. Pairing is a non-trivial topic to cover ([here](https://static1.squarespace.com/static/5fdbb09f31d71c1227082339/t/5ff394720493bd28278889c6/1609798774687/PairingsForBeginners.pdf) is an excellent post in detail), but for the sake of this post, we will only make use of its property:

$e(g^a, g^b) = e(g, g)^{ab}$

Now we can derive the verification condition: 

$RHS = e(w_i, g^{\tau} / g^{i}) \cdot e(g, g)^{f(i)} = e(g^{\psi_i(\tau)}, g^{(\tau-i)}) \cdot e(g, g)^{f(i)}$

$RHS = e(g^{\dfrac{f(\tau) - f(i)}{\tau-i}}, g^{(\tau-i)}) \cdot e(g, g)^{f(i)}$

With the raise to the power for each $g$ in the pairing can be moved outside and cancels out: 

$RHS = e(g, g)^{f(\tau) - f(i)} \cdot e(g, g)^{f(i)} = e(g^{f(\tau)}, g) = e(C, g)$

Here we arrive at the $RHS = e(C, g) = LHS$. With the magic of bilinear mapping, the secret $\tau$ is never used. 

### Result

Compared to the vector commitment is as large as the vector size, the KZG commitment size is constant as shown in Step 1. It is in fact just a group element $C = g^{f(\tau)}$. 

Compared to the vector commitment that needs to reveal the entire vector to prove the commitment or Merkle tree that needs to reveal the path as the depth of the tree, the KZG committer just needs to reveal two values $i, f(i)$ and one group element $w_i$ as shown in Step 2. 

To verify the proof, there are two pairings and two group multiplications. While the computation cost of pairings is beyond discussion here, we can see it is independent from the size of the polynomial.

### Example: Verifiable Secret Sharing

With KZG polynomial commitment, we now have a way to create a commitment against a vector of messages. For each individual message, we can verify the commitment to its value without needing to know the rest of other messages. 

We can apply this to the secret sharing scenario: a dealer needs to create a list of secret shares. Each node that gets the secret share wants to verify it is indeed in the secret. 

Using the KZG commitment scheme, the dealer now needs commits to a chosen polynomial $C = Com(f(x))$ where the secret is $f(0)$. 

For each secret share $f(i)$ the dealer creates, he needs to send each node $(i, f(i), w_i)$ securely. 

Each node can verify the share it receives $Verify(PK, C, i, f(i), w_i)$. The secret share algorithm concludes if each node verifies their corresponding share successfully. 

## Resources
1. [https://vitalik.ca/general/2021/01/26/snarks.html](https://vitalik.ca/general/2021/01/26/snarks.html)
3. [https://dankradfeist.de/ethereum/2020/06/16/kate-polynomial-commitments.html](https://dankradfeist.de/ethereum/2020/06/16/kate-polynomial-commitments.html)
4. [https://www.iacr.org/archive/asiacrypt2010/6477178/6477178.pdf](https://www.iacr.org/archive/asiacrypt2010/6477178/6477178.pdf)

## Notes 
[^1]: For simplicity, we do not discuss the difference between computationally hiding/binding and unconditionally hiding/binding. 

[^2]: Ethereum uses a Merkle Patricia tree that allows each node to have more than two children, here we use the simplied version. 