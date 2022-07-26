---
title: 'Ethereum Yellow Paper Mathematics Deciphered | Part 1: World State'
date: 2022-07-26 11:41:12
tags:
- ethereum
- yellow-paper
excerpt: 'This post is part of series of articles that try to decipher mathematics in ethereum yellow paper, so that it makes sense to the reader and convey their meaning'
mathjax: true
---


This part of the series will cover section 2 of the [Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf), which explains world state.

I assume you've already gone through conventions defined in [Part 0](https://blog.nvn.fyi/19/07/2022/ethereum-yellow-paper-mathematics-deciphered-part-0/). If you haven't yet, I highly recommend you to go check it out first.

## World State
Ethereum can be considered as a transaction based [state machine](https://statecharts.dev/what-is-a-state-machine.html). It begins with a genesis state and by executing transactions, it goes to a new state - a valid one enforced by a set of rules.  
The state of this state machine is termed - "world state" and denoted by $\boldsymbol{\sigma}$. The $\boldsymbol{\sigma}$ is, simply put, a mapping between addresses and corresponding account states as you'll see more later.

```
    | WORLD STATE | 

Address     AccountState
----------------------------------------------------------------
address1 -> accountState1(nonce, balance, storageRoot, codeHash)
address2 -> accountState2(nonce, balance, storageRoot, codeHash)
address3 -> accountState3(nonce, balance, storageRoot, codeHash)
        .
        .
```

That brings us to the first equivalence from paper -

$$
\boldsymbol{\sigma}_{t+1} \equiv \Upsilon(\boldsymbol{\sigma}_t, T) \tag{1}
$$

where,
- $\boldsymbol{\sigma}_t$ is the world state at some time $t$
- $T$ is a single transaction represented by a tuple
- $\Upsilon$ is the state transition function

When given the current state $\boldsymbol{\sigma}\_t$ and a transaction $T$, the state transition function $\Upsilon$ is defined to be a function that will generate a valid next state $\boldsymbol{\sigma}\_{t+1}$.

Keep in mind that $\Upsilon$ provides a new state given a **transaction** tuple, $T$, i.e. next state that'll result from executing a single transaction $T$.

Now, comes the **block**. Isn't it that Ethereum jumps to a new state, when a block is mined? Where does the block fit in the equation?

Well, yes a new state is indeed generated when a block in finalized by mining. In fact, the individual state transitions corresponding to each transaction in the block ($T_0, T_1, T_2...$) happen through $\Upsilon$ function.

$$
\boldsymbol{\sigma}\_{t+1} \equiv \Upsilon(\boldsymbol{\sigma}\_{t}, T_0) \\\\
\boldsymbol{\sigma}\_{t+2} \equiv \Upsilon(\boldsymbol{\sigma}\_{t+1},T_1) \\\\ 
\boldsymbol{\sigma}\_{t+3} \equiv \Upsilon(\boldsymbol{\sigma}\_{t+2},T_2) \\\\
$$


and so on...

Above three can be combined to be written as:  
$$
\boldsymbol{\sigma}_{t+3} \equiv \Upsilon(
    \Upsilon(
        \Upsilon(\boldsymbol{\sigma}_t, T_0)
        , T_1)
    , T_2) \tag{i}
$$

Now, we introduce another state-transition function $\Pi$ from paper,
$$
\boldsymbol{\sigma}_{t+1} \equiv \Pi(\boldsymbol{\sigma}_t, B) \tag{2}
$$

where,
- $\boldsymbol{\sigma}_t$ is the world state at time $t$
- $B$ is a single block, represented as a tuple
- $\Pi$ is the state transition function at block level

$\Pi$ may look similar to $\Upsilon$, but notice that $\Upsilon$ transitions state at transaction level. While $\Pi$ transitions state at block level. It transitions the world state, $\boldsymbol{\sigma}$, to a new state by including and finalizing all the transactions in the block $B$.

The block is a tuple with multiple components, one of which is a list of transactions (precisely, a list of transaction tuples),  $\mathbf{T} = (T_0, T_1. T_2, \ldots)$  

$$
B \equiv (\ldots, \mathbf{T}, \ldots) \\\ 
$$
which is same as-
$$
B \equiv (.., (T_0, T_1. T_2,...), ..)  \tag{3}
$$
_(**Note**: ellipsis before and after the list of transactions, $\mathbf{T}$ abbreviate other components of the block tuple. Other components of $B$ will be introduced later in series.)_

Paper defines another function $\Omega$ to be equivalent to $\Pi$ as,
$$
\Pi(\boldsymbol{\sigma}, B) \equiv \Omega(B, 
    \Upsilon(
        \Upsilon(\boldsymbol{\sigma}, T_0)
        , T_1) \ldots
    ) \tag{4}
$$

where,
- $\Omega$ is the block finalization state transition function
- $\Pi$ is the block level state transition function
- $B$ is a block tuple 

This might seem a bit confusing, but remember that all of the above are equivalences ($\equiv$) not equalities ($=$).

The RHS of $(4)$ seems to elaborate the $\Pi$ a bit more explicitly through $\Omega$. Notice the $(\Upsilon(\Upsilon(\boldsymbol{\sigma}, T_0), T_1) \ldots)$ here is same as described at $(i)$ above, which is equivalent to a state $\boldsymbol{\sigma}$. If you tried to substitute it in the $(4)$,

$$
\Pi(\boldsymbol{\sigma}, B) \equiv \Omega(B, \boldsymbol{\sigma}) \tag{ii}
$$

the equivalence seems more obvious. Additionally, $\Omega$ (and hence $\Pi$) is defined to reward the nominated miner of the block $B$ apart from finalization of $B$.

## Account State
As said before, the world state, $\boldsymbol{\sigma}$, is a mapping between addresses and their account states. The account state is a RLP-serialized data-structure. For an address $a$, the account state can be denoted by $\boldsymbol{\sigma}[a]$ - similar to accessing a value in a map by a key (here, $a$) in programming.

An account state $\boldsymbol{\sigma}[a]$ comprises of the following components -
- **nonce**, $\boldsymbol{\sigma}[a]_\mathrm{n}$: Nonce is number of transactions sent by this address, if it is an EOA, or it is number of contract-creations made by this address if it is a contract.
- **balance**, $\boldsymbol{\sigma}[a]_\mathrm{b}$: Number of Wei owned by this address.

- **storageRoot**, $\boldsymbol{\sigma}[a]_\mathrm{s}$: A 256-bit hash of root node of a [Merkle Patricia Trie](https://ethereum.org/en/developers/docs/data-structures-and-encoding/patricia-merkle-trie/) data-structure. This MPT structure holds contents of this account. The contents in the MPT are encoded mapping of 256-bit hash of 256-bit keys (aka storage slot) to RLP-encoded 256-bit integers as values (representing the account content). In simple terms, you might know this "content" as contract storage. As expected, only contract accounts can have non-empty storage. EOAs always have empty storage.

- **codeHash**, $\boldsymbol{\sigma}[a]_\mathrm{c}$: The hash of the EVM code (aka bytecode) of this account. Remember that only contracts have bytecode, EOAs have empty bytecode. Thus, if $\mathbf{b}$ is the bytecode then,  $\boldsymbol{\sigma}[a]_\mathrm{c} = \mathtt{KEC}(\mathbf{b})$. And $\mathbf{b} = ()$, always for an EOA account - meaning **codeHash** for an EOA is always hash of an empty string.

![A diagram showing the make up of an account](https://ethereum.org/static/19443ab40f108c985fb95b07bac29bcb/302a4/accounts.png "A diagram showing the make up of an account")

Paper defines a function $L_I$ that given a key, $k$ and a value, $v$ outputs suitable item for inserting into the account storage Merkle Patricia Trie. In this regard, $k$ must be keccak-256 hashed and $v$ must be RLP-encoded, as discussed earlier above in **storageRoot** component. Hence,

$$
L_I((k, v)) \equiv (\mathtt{KEC}(k), \mathtt{RLP}(v)) \tag{8}
$$

where $k$ must be 32-byte (256-bit) long **and** $v$ must be a natural number. Formally this is same as,

$$
k \in \mathbb{B_{32}} \land v \in \mathbb{N}  \tag{9}
$$

According to convention then, $L_T^*$ is a similar function to $L_I$ except $L_T^*$ takes as input a series of key-value pairs $((k_0, v_0), (k_1, v_1), (k_2, v_2)...)$ and performs same operation but element-wise. That is,

$$
L_T^*(( (k_0, v_0), (k_1, v_1),...) ) = (L_T((k_0, v_0)), L_T((k_1, v_1)), ...) \tag{iii}
$$

Finally, using all the above the paper defines a convenient equivalence -

$$
\mathtt{TRIE}( L_I^*( \boldsymbol{\sigma}[a]_\mathbf{s} ) ) \equiv \boldsymbol{\sigma}[a]\_{\mathrm{s}} \tag{7}
$$

_Note that in the equivalence above, the subscript $\mathbf{s}$ (bolder, on LHS) is different from subscript $\mathrm{s}$ (on RHS)_

We already know that $\boldsymbol{\sigma}[a]\_{\mathrm{s}}$ is the **storageRoot** defined earlier. If you look carefully, $\mathtt{TRIE}( L_I^*( \boldsymbol{\sigma}[a]_{\mathbf{s}} ) )$ is actually defines the same Merkle Patricia Trie whose root's hash is **storageRoot**. How?

$\boldsymbol{\sigma}[a]_{\mathbf{s}}$ (bolder $\mathbf{s}$) seems to represent a list/series of contract's storage key-value pairs (raw integer key and raw storage content) corresponding to an account, $a$. The $L_I^*$  then hashes each of the keys and RLP-encodes each of the values. Just as defined at $(iii)$. And with these encoded inputs the Merkle Patricia Trie is constructed with $\mathtt{TRIE}$ function.

The equivalence $(7)$ is made just for convenience since we typically refer to the trie's underlying set of key-value pairs and NOT just the hash of storage trie root $\boldsymbol{\sigma}[a]_{\mathrm{s}}$. It is assumed from now on that the latter is equivalent to former i.e. $\boldsymbol{\sigma}_\mathrm{s}$ maybe used to refer to the MPT and not just the simple hash value.

A _non-existent_ account is defined as:

$$
\boldsymbol{\sigma}[a] = \varnothing 
$$

Whereas an _empty_ account i.e. it has no code (bytecode), zero nonce and zero balance, given as:

$$
\mathtt{EMPTY}(\mathbf{\sigma}, a) \equiv \boldsymbol{\sigma}[a]_\mathrm{c} = \mathtt{KEC}(()) \land \boldsymbol{\sigma}[a]_\mathrm{n} = 0 \land \boldsymbol{\sigma}[a]_\mathrm{b} = 0  \tag{14}
$$

An account is _dead_ if it is either non-existent or empty. Mathematically it is same as,

$$
\mathtt{DEAD}(\boldsymbol{\sigma}, a) \equiv \boldsymbol{\sigma} = \varnothing    \lor \mathtt{EMPTY}(\boldsymbol{\sigma}, a) \tag{15}
$$

Now, paper defines the world-state collapse function $L_\mathrm{S}$ for existent accounts, $a$ as,

$$
L_\mathrm{S} \equiv \\{ p(a): \mathbf{\sigma}[a] \neq \varnothing \\}
$$

where,

$$
p(a) \equiv  \big(\mathtt{KEC}(a), \mathtt{RLP}\big( (\boldsymbol{\sigma}[a]_{\mathrm{n}}, \boldsymbol{\sigma}[a]\_{\mathrm{b}}, \boldsymbol{\sigma}[a]\_{\mathrm{s}}, \boldsymbol{\sigma}[a]\_{\mathrm{c}}) \big) \big) \tag{11}
$$

Remember, that the world-state is a mapping from account to account states. $L_\mathrm{S}$ function is basically squishing everything belonging to an account state and yields a set of these values for all accounts (i.e. $\forall a$) except non-existent accounts (i.e. $\mathbf{\sigma}[a] \neq \varnothing $). 

This set of items, yielded by $L_\mathrm{S}$ function, when put into the Merkle Patricia Trie (using $\mathtt{TRIE}$ function), provides a short identity for the world-state. This identity is nothing but the hash of root of this trie. Keep in mind that this MPT is not the same one as defined for **storageRoot** for a single account. This MPT encompasses all the accounts with it's contents. You might know this as "State Trie".

It is assumed that account state can either be non-existent or valid if existent. Mathematically,
$$
\forall a: \boldsymbol{\sigma}[a] = \varnothing  \vee  (a \in \mathbb{B}_{20} \wedge v(\boldsymbol{\sigma}[a])) \tag{12}
$$

where, $v$ is a validity function which just sets some restrictions on what values an account's components can take (e.g. nonce and balance must be natural number less than $2^{256}$):

$$
\quad v(x) \equiv x\_{\mathrm{n}} \in \mathbb{N}\_{256} \wedge x\_{\mathrm{b}} \in \mathbb{N}\_{256} \wedge x\_{\mathrm{s}} \in \mathbb{B}\_{32} \wedge x_{\mathrm{c}} \in \mathbb{B}\_{32} \tag{13}
$$

And that concludes this part of the series! Next part will be all about transactions. Stay tuned!

## Resources
- [State Machine](https://statecharts.dev/what-is-a-state-machine.html)
- [Ethereum Accounts](https://ethereum.org/en/developers/docs/accounts/)
- [Merkle Patricia Trie](https://ethereum.org/en/developers/docs/data-structures-and-encoding/patricia-merkle-trie/)
- [RLP encoding](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/) 

