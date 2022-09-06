---
title: 'Ethereum Yellow Paper Mathematics Deciphered | Part 2: Transaction'
date: 2022-08-05 12:44:12
tags:
- ethereum
- yellow-paper
excerpt: 'This post is part of series of articles that try to decipher mathematics in ethereum yellow paper, so that it makes sense to the reader and convey their meaning'
mathjax: true
---

This part of the series ventures into section 4 of the [Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf), to describe transactions.

Be sure to check out [Part 0](https://blog.nvn.fyi/19/07/2022/ethereum-yellow-paper-mathematics-deciphered-part-0/), if you haven't yet to get conventions right.

## Transaction

As said in previous part, transactions are what that drive the ethereum state machine from one state to another. Consider a transaction, $T$, as tuple containing multiple fields for the paper. It is equivalently represented as a cryptographically signed instruction - which is basically encoding all of its fields together and signing it with a private key.

These are cryptographically signed instructions are signed by the sender which can only be an EOA, not a contract. This may seem confusing to Solidity developers. Since a contract can also call other contract & be a sender (`msg.sender`) in context of the called contract - right? Well, yes but the originator of the transaction (`tx.origin`) i.e. the entity which initialized the the transaction in the first place is always an EOA. A transaction can either result in a message call to some existing contract or creation of a new contract.

### Types
The Berlin version of the protocol introduced two distinct transaction types (see [EIP2718](https://eips.ethereum.org/EIPS/eip-2718) and [EIP2930](https://eips.ethereum.org/EIPS/eip-2930)):
- **0** (legacy): The legacy transactions, before the introduction of the new transaction types.
- **1** ([EIP2930](https://eips.ethereum.org/EIPS/eip-2930)): These transactions specify additional fields in the transaction tuple (see below).

## Transaction Anatomy

### Common Fields
All transactions regardless of type/action have a number of common fields:
- $\mathbf{type}$ ($T_{\mathrm{x}}$): EIP-2718 transaction type.
- $\mathbf{nonce}$ ($T_\mathrm{n}$): Nonce of sender i.e. no. of transactions sent by sender
- $\mathbf{gasPrice}$ ($T_\mathrm{p}$): Gas price of this transaction i.e wei per unit of gas paid for this transaction
- $\mathbf{gasLimit}$ ($T_\mathrm{g}$): Max amount of gas this transaction can use. Paid up-front by sender
- $\mathbf{to}$ ($T_\mathrm{t}$): Address of receiver. If null ($T_\mathrm{t} = \varnothing$), it is a contract creation transaction.
- $\mathbf{value}$ ($T_\mathrm{v}$): Amount of wei to be sent to receiver or in case of contract creation, a value to newly created contract.
- $\mathbf{r}$, $\mathbf{s}$ ($T_\mathrm{r}, T_\mathrm{s}$): Values that correspond to signature of the transaction and used to verify, cryptographically, the sender of the transaction.

### Type-1 Tx
A type-1 transaction (i.e. $T_{\mathrm{x}} = 1$) has some additional fields:

- $\mathbf{accessList}$ ($T_\mathbf{A}$): List of [access entries](https://www.evm.codes/about#accesssets) to warm-up. Each list item is a tuple $E$. Tuple $E$ has two items - and account address and a list of storage keys, i.e. $E \equiv (E_{\mathrm{a}}, E_{\mathbf{s}})$.  
  
_Notice the subscript $\mathbf{s}$ in $E\_{\mathbf{s}}$ is bold lowercase - meaning it's a sequence not some scalar value according to convention. You an specify these access lists using [ethers](https://docs.ethers.io/v5/api/providers/types/#providers-AccessList), for example, while interacting with a contract._

- $\mathbf{chainId}$ ($T_\mathrm{c}$): Chain id. Must be equal to network chain id, $\beta$.
- $\mathbf{yParity}$ ($T_\mathrm{y}$): Y-parity of the transaction. Used to compute the $\mathbf{v}$ value of the signature. See [this](https://ethereum.stackexchange.com/questions/15766/what-does-v-r-s-in-eth-gettransactionbyhash-mean).

### Type-0 (Legacy) Tx
The legacy ($T_{\mathrm{x}} = 0$) do not have $\mathbf{accessList}$ (i.e. $T_{\mathbf{A}} = ()$). However, $\mathbf{chainId}$ and $\mathbf{yParity}$ are combined into a single value $\mathbf{w}$:
- $\mathbf{w}$ ($T_\mathrm{w}$): A scalar value that encodes $\mathbf{yParity}$ ($T_\mathrm{y}$) and, if present, the $\mathbf{chainId}$ ($\beta$).  So,  

$$
T_{\mathrm{w}} = \begin{cases}
27 + T_{\mathrm{y}} \\\\
2\beta + 35 + T_{\mathrm{y}} & \text{if } \beta  \text{ is known}
\end{cases}
$$

See [EIP-155](https://eips.ethereum.org/EIPS/eip-155).

### Contract Creation Tx
If the transaction is a contract creation transaction, it also contains: 
- $\mathbf{init}$ ($T_\mathbf{i}$): The byte array specifying initialization process of contract. You might know it as the **creation bytecode** fragment of the contract bytecode.

_Keep in mind that in contract creation transactions $\mathbf{to}$ is set to null i.e. $T_\mathrm{t} = \varnothing$_

$\mathbf{init}$ fragment of the bytecode returns the $\mathbf{body}$, aka **runtime bytecode**, fragment of the bytecode to EVM, after which $\mathbf{init}$ part is discarded and $\mathbf{body}$ is what lives on chain & executed in response to calls. The $\mathbf{init}$ is always concatenated with the $\mathbf{body}$ in a contract creation transaction to be sent.

### Message Call Tx
Now, if transaction is a message call transaction instead of contract creation, it has:

$\mathbf{data}$ ($T_\mathbf{d}$): A byte array that is input data to the called contract.

_Let me disambiguate something for folks who've used ethereum libraries like ethers or web3.js. While creating a custom transaction object using these, $\mathbf{data}$ and ($\mathbf{init}$ + $\mathbf{body}$) is generally specified at same key - **data**. Library interprets the **data** as $\mathbf{data}$ or ($\mathbf{init}$ + $\mathbf{body}$) depending on **to** key in transaction object. If **to** is null  it's ($\mathbf{init}$ + $\mathbf{body}$), i.e. contract creation, otherwise $\mathbf{data}$, a message call._

The paper defines the transaction collapse function, $L_T(T)$ as:

$$
L_T(T) \equiv \begin{cases}
(T_{\mathrm{n}}, T_{\mathrm{p}}, T_{\mathrm{g}}, T_{\mathrm{t}}, T_{\mathrm{v}}, \mathbf{p}, T_{\mathrm{w}}, T_{\mathrm{r}}, T_{\mathrm{s}}) &  \text{if} \ T_{\mathrm{x}} = 0 \\\\
(T_{\mathrm{c}}, T_{\mathrm{n}}, T_{\mathrm{p}}, T_{\mathrm{g}}, T_{\mathrm{t}}, T_{\mathrm{v}}, \mathbf{p}, T_{\mathbf{A}}, T_{\mathrm{y}}, T_{\mathrm{r}}, T_{\mathrm{s}}) & \text{if} \ T_{\mathrm{x}} = 1 
\end{cases} \tag{16}
$$

where,
$$
\mathbf{p} \equiv \begin{cases}
T_{\mathbf{i}} & \text{if}\ T_{\mathrm{t}} = \varnothing \\\\
T_{\mathbf{d}} & \text{otherwise}
\end{cases} \tag{17}
$$

The $L_T(T)$ basically covers all possible values of a value received by concatenating the components of tuple $T$ and components depend on what transaction type is and if it's a contract creation transaction or message call. The $(16)$ can hence be equivalently expanded to:
$$
L_T(T) \equiv \begin{cases}
(T_{\mathrm{n}}, T_{\mathrm{p}}, T_{\mathrm{g}}, T_{\mathrm{t}}, T_{\mathrm{v}}, T_{\mathbf{i}}, T_{\mathrm{w}}, T_{\mathrm{r}}, T_{\mathrm{s}}) & \text{if} \ T_{\mathrm{x}} = 0 \ \text{and} \ T_{\mathrm{t}} = \varnothing \\\\
(T_{\mathrm{n}}, T_{\mathrm{p}}, T_{\mathrm{g}}, T_{\mathrm{t}}, T_{\mathrm{v}}, T_{\mathbf{d}}, T_{\mathrm{w}}, T_{\mathrm{r}}, T_{\mathrm{s}}) & \text{if} \ T_{\mathrm{x}} = 0 \ \text{and} \ T_{\mathrm{t}} \neq \varnothing \\\\
(T_{\mathrm{c}}, T_{\mathrm{n}}, T_{\mathrm{p}}, T_{\mathrm{g}}, T_{\mathrm{t}}, T_{\mathrm{v}}, T_{\mathbf{i}}, T_{\mathbf{A}}, T_{\mathrm{y}}, T_{\mathrm{r}}, T_{\mathrm{s}}) & \text{if} \ T_{\mathrm{x}} = 1 \ \text{and} \ T_{\mathrm{t}} = \varnothing \\\\
(T_{\mathrm{c}}, T_{\mathrm{n}}, T_{\mathrm{p}}, T_{\mathrm{g}}, T_{\mathrm{t}}, T_{\mathrm{v}}, T_{\mathbf{d}}, T_{\mathbf{A}}, T_{\mathrm{y}}, T_{\mathrm{r}}, T_{\mathrm{s}}) & \text{if} \ T_{\mathrm{x}} = 1 \ \text{and} \ T_{\mathrm{t}} \neq \varnothing
\end{cases}
$$

Remember from [conventions](https://blog.nvn.fyi/19/07/2022/ethereum-yellow-paper-mathematics-deciphered-part-0/) that $L_T$ function squishes the transaction tuple through RLP encoding into a sequence of bytes.

There are only certain possible values each constituent of $T$ can take which is laid out in the paper as:

$$
T_{\mathrm{x}} \in \\{0, 1\\} \ \wedge \ T_{\mathrm{c}} = \beta \ \wedge \ T_{\mathrm{n}} \in \mathbb{N}_{256} \ \wedge \\\\
T\_{\mathrm{p}} \in \mathbb{N}\_{256} \ \wedge \ T\_{\mathrm{g}} \in \mathbb{N}\_{256} \ \wedge \ T\_{\mathrm{v}} \in \mathbb{N}\_{256} \ \wedge \\\\
T\_{\mathrm{w}} \in \mathbb{N}\_{256} \ \wedge \ T\_{\mathrm{r}} \in \mathbb{N}\_{256} \ \wedge \ T\_{\mathrm{s}} \in \mathbb{N}\_{256} \ \wedge \\\\
T\_{\mathrm{y}} \in \mathbb{N}\_{1} \ \wedge \ T\_{\mathbf{d}} \in \mathbb{B} \ \wedge \ T\_{\mathbf{i}} \in \mathbb{B}
$$  

where $\mathbb{N}\_{\mathrm{n}}$ is a set of all natural numbers less that $2^{\mathrm{n}}$. Same as,
$$
\mathbb{N}_{\mathrm{n}} = \\{ P: P \in \mathbb{N} \wedge P \lt 2^n \\} \tag{19}
$$

The case with $T_{\mathbf{t}}$ is a bit difference because (as you might've guessed) depends on whether transaction is contract creation or message call. In case of former ($T_{\mathrm{t}} = \varnothing$) it is simply RLP empty byte sequence, otherwise normal 20-byte public address:

$$
T\_{\mathbf{t}} \in 
\begin{cases} \mathbb{B}\_{20} & \text{if} \quad T\_{\mathrm{t}} \neq \varnothing \\\\
\mathbb{B}\_{0} & \text{otherwise}\end{cases} \tag{20}
$$

And this concludes this part! Next up is the section about **Block** stay tuned!