---
title: 'Ethereum Yellow Paper Mathematics Deciphered | Part 0: Conventions'
date: 2022-07-19 19:19:54
tags:
- ethereum
- yellow-paper
excerpt: 'This post is part of series of articles that try to decipher mathematics in ethereum yellow paper, so that it makes sense to the reader and convey their meaning'
---


# Introduction
Ethereum is, by far, the most popular blockchain, which it owes not only to its technical attributes but also to its economical and most decentralized attributes. This beautiful machinery boils to numerous mathematical artifacts, laid out in the [Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf), each moving part pertaining to some equation.

My search for explanations of all things mathematical from the Yellow Paper on the internet always resulted articles that dodged the very equations/equivalences and mathematical symbols revolving around them - obscuring the meanings they convey. And I'm not really in enough proximity (physically or digitally) of a Mathematician chad to ask for stuff here and there. Therefore, I'm writing these series of writings to solidify what I already know. And hopefully, along the way, to help you too, anon.

Before you move on, this is not an article focused on describing what Ethereum or EVM is or their constituent mean. Instead, this series is focused on the mathematical artifacts governing the blockchain. Therefore I assume, you already have some knowledge of Ethereum's structure. I assume the terms like - _transaction_, _gas_, _block_, _world state_, _fork_, _nonce_, _contract_ etc. makes sense to you and you have at least some basic knowledge of high-school level set theory.

All said, this will deepen your Ethereum knowledge to the bare bones, so do read anyway :). Alright - let's start with some conventions used in the paper.

# Symbols & Conventions
This section might be a great reference for further, subsequent parts of this series. Also, I might probably update it as I proceed.

Following are some mathematical symbols - some of which you might know already but I'm going to put some here for completeness:

- $\equiv$ denotes equivalence i.e. value on LHS and RHS are equivalent.
- $\in$ denotes that an element/variable belongs to a particular set.
- $\lor$,$\land$ denote logical OR and AND operator respectively  
These are usually used to constraint the values that can be assigned to components of a tuple. For example, the expression,  
  $a \in X \land b \in Y$  
conveys that "$a$ must be in set $X$ **and** $b$ must be in set $Y$".
- $\forall$ denotes universal quantification i.e. for all elements in a set.

Note that the paper, almost everywhere uses **equivalence** (using $\equiv$), which is not same as equality (using $=$). In this context, $\equiv$ is used to define an expression or something being identical to other thing.

Following are the conventions used for choosing a particular symbol and example below:

## Top level state values are bold and lowercase Greek letters:  
$\boldsymbol{\sigma}$: World state (sequence of account tuples)  
$\boldsymbol{\mu}$: Machine state (aka EVM state)

## Function operating on these (top-level) state are uppercase Greek letter:  
$\Upsilon$: State transition function  (transaction level)  
$\Pi$: State transition function  (block level)  
$C$: Gas const function (can be subscripted like $C_{SSTORE}$)

## Specialized and externally defined functions in typewriter text:
$\tt{KECC}$/$\tt{KECC512}$: keccak256/keccak512 hash function  
$\tt{RLP}$: [RLP](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/) encoding function

## Tuples denoted by uppercase letter:  
$T$: Tuple representing transaction tuple/object.  
It can be letter-subscripted to denote an individual component of tuple e.g. $T_n$ denotes nonce of transaction. However when it is number-subscripted (e.g. $T_0, T_1, T_2 ...$) it denotes a transaction tuple at a particular index in list of transactions.

## Scalars and fixed size bytes denote by small case letters:
$n$: Nonce  
$\beta$: Chain id  
$\delta$: No. of items required on stack for an operation

## Dynamic length sequences denoted by bold lower case letters:
$\bf{o}$: output data of a message call  
Last item of a given sequence, e.g. a byte sequence, $\mathbf{x}$ is $\ell(\mathbf{x})$:
$$
\ell(\mathbf{x}) \equiv \mathbf{x}[\lVert \mathbf{x} \rVert - 1]
$$  
where, $\lVert \mathbf{x} \rVert$ denotes length of the sequence $\mathbf{x}$.

## Square brackets to index into and reference individual items or subsequences of a sequence:
$\boldsymbol{\mu}_{\bf{s}}[i]$: i-th item of the stack  

$\boldsymbol{\mu}_{\bf{m}}[i..j]$: items ranging from i to j of machine's (EVM) memory  

## Other Assumptions:
- All scalars are non-negative integers. That is, all scalars belong to set $\mathbb{N}$. Similarly set of all byte-sequences is $\mathbb{B}$.  
If the set is constrained to particular subset, it is denoted by subscript:  
  - $\mathbb{N_{256}}$ is all non-negative integers less that $2^{256}$  
  - $\mathbb{B_{32}}$ is set of all byte-sequences of fixed length $32$  

- If unmodified input is denoted by $\square$. Then:
  - $\square'$: modified/utilizable value
  - $\square^*$, $\square^{**}$: intermediate values

- When considering the use of existing functions, given a
function $f$, the $f^*$ denotes similar element-wise version of $f$. Therefore:  
$f^*((x_0, x_1, x_2,...))$ = $(f(x_0), f(x_1), f(x_2)...)$

## Miscellaneous:
  - A particular set of functions in the paper are denoted by $L$ (e.g. $L_I, L_T, L_S$). These $L$ functions are referred as "collapse-functions" in the paper. Because it, sort of, collapses/squishes multiple components of a complex entity (like transaction tuple) to yield an output.  
  This output (seemingly, as in paper) is suitable for RLP encoding (i.e.suitable to be fed into $\tt{RLP}$). For example, given a transaction tuple, $T$, it is made suitable for RLP encoding by $L_T(T)$ function. $L_T$ is needed rather than directly feeding $T$ to $\tt{RLP}$ because, it is $L_T$ that dictates what components of $T$ (_type_, _nonce_, _gasPrice_, _to_ etc.) should be part encoding and in what order they should be encoded. This is so that encoding scheme remains uniform throughout.


  - The **first block number** of a particular fork is specially denoted by $F$ with a subscript name of fork. Below are forks with their first block number:  
$F_{Homestead} = 1150000$  
$F_{TangerineWhistle} =  2463000$   
$F_{Spurious Dragon} =   2675000$  
$F_{Byzantium} = 4370000$  
$F_{Constantinople} = 7280000$  
$F_{Petersburg} = 7280000$  
$F_{Istanbul} = 9069000$  
$F_{Muir Glacier} = 9200000$  
$F_{Berlin} = 12244000$  
$F_{London} = 12965000$  
$F_{Arrow Glacier} = 13773000$  
$F_{Gray Glacier} = 15050000$  

And this is it for the pilot of this series. I'm not expecting you to understand exhaustive meaning of everything above, it will be uncovered later. This will however, serve as great reference for the rest of the series.