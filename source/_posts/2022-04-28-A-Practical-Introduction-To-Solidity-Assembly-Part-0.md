---
title: 'A Practical Introduction To Solidity Assembly: Part 0'
date: 2022-04-28 16:47:46
tags:
- solidity
- ethereum
- smart-contract
---

Solidity is by far the most used language for smart contracts on Ethereum blockchain. It is a high-level language that abstracts away various underlying nitty-gritty details (like several important safety checks) - and that is for good! 

However, sometimes however you may want to go deep down to fine-grain controls of EVM whether it is for gas golfing, writing optimized libraries, or maybe some other witch-craft, who knows. Luckily, you can actually write part of your contract (or whole contract!) assembly language.

The language used for assembly here is called [Yul](https://docs.soliditylang.org/en/v0.8.13/yul.html#yul) which is your portal to access EVM's fine-grain controls. Be warned - **writing assembly bypasses many important checks of Solidity. You should only use it when needed and if you know what the heck is going on!**

Before you proceed, I assume you know basics of Ethereum and Solidity language, already. Alright let's go!

## What to Expect
We're going to take at the classic `Box` contract below and progressively convert this contract to pure assembly. Apart from learning about Solidity assembly, you'll also learn how opcodes work, what goes under the hood when you deploy a contract or make an external function call and different locations where data is stored and when.

## Prelude
In this part we are going to briefly focus on how Solidity handles and store variables in different places during execution. This will be key to understanding how Solidity assembly works. We are going to have intro to 4 data locations: **storage**, **memory**, **call stack**, and **calldata**.

### Storage
Storage is where state variables are stored and persisted **permanently**. Structure of this storage can be visualized as simple key-value pairs. Both key and value are 32 bytes long. And since keys are 32 bytes or 256 bits long, there can be at most 2^256 different key-value pairs. 

```
Slot# (key)      Value
-----------------------
0      ----->     123
1      ----->     456
      .
      .

```

The key here can also be considered as slot number - which ranges from 0 to 2^256 - 1. Hence, knowing the slot number, you can read data stored in that slot. This is sufficient basic overview of storage structure for our scope - but feel free to geek out on this [Program The Blockchain](https://programtheblockchain.com/posts/2018/03/09/understanding-ethereum-smart-contract-storage/) post to dive into more details.

### Memory
Memory is where **temporary** variables are stored during a function execution in Solidity code. Memory is usually used to store complex types such as arrays. You must've encountered `memory` keyword in Solidity code when defining an array, like this:
```solidity
function f() public {
    uint256[] memory arr = new uint256[](10);
    // ..
}
```

 Memory can be visualized as a byte array where data can be written in 32 byte or 1 byte chunks and read in 32 byte chunks.

```
-----------------------------
 1B |   32B   |   32B  |  ..
-----------------------------
```

### Call Stack
Like memory call stack is also a volatile place for storing some data. While writing solidity you can manipulate Storage and Memory, the call stack is automatically utilized behind the scenes. It cannot be accessed using plain solidity code, but through assembly. Call stack is used to store values that are used immediately and don't need to be stored for later use.

So, how it differs from memory? The Solidity code is actually compiled to a list opcodes. You can consider these opcodes like low-level functions which can take inputs and give output. The way these opcodes receive argument/inputs (if any) is by popping off the same number of values from the call stack as the number of parameters it accepts. And, if it outputs anything it pushes the returned value to call stack.

For example - when you write to a state variable, the `SSTORE` opcode is executed. Now, `SSTORE` requires two inputs - the slot number to write to and the value to write. So, what `SSTORE` does is it pops off two values from the call stack and uses them as inputs. `PUSH1` opcode can be used to push these inputs to stack first.

```
opcode      |   call stack
-----------------------------
PUSH1 0x05  |   0x5
            |
-----------------------------
PUSH1 0x00  |   0
            |   0x5
-----------------------------
SSTORE      |    -
```

The above example shows the state of call stack after corresponding opcode is executed. `SSTORE` store value `0x05` (5 in decimal) to slot `0x00` (0 in decimal) of Storage - by popping off two values from the call stack and using them as inputs.

This is why EVM is sometimes referred to as a stack machine!

You may ask why not use memory itself instead of call stack? Well, because using stack is more efficient and cheaper than memory for primitive/simple types. EVM is not the first to do this actually - many fast statically-typed languages like Rust also use stack.

### Calldata
Calldata is similar to memory but is a special data location where function arguments are stored, only available to external function calls. According to docs:

_Calldata is a non-modifiable, non-persistent area where function arguments are stored, and behaves mostly like memory_

The calldata may look like this:
```
0x6057361d000000000000000000000000000000000000000000000000000000000000000a
```
where first 4 bytes (`6057361d`) are the function selector (identifier for target function) and rest are the input parameters (padded) passed to function.

The way EVM decides which function to execute seeing the received calldata is by matching the function selector from bytecode with pre-calculated function selectors of all the functions defined in the contract. We'll later implement this manually when writing a pure assembly contract.

So, you can consider an external function call is nothing but a long hex data (calldata) sent to a particular contract address.

## Solidity Contract
Okay, we have all the required basic knowledge to better understand Solidity assembly (Yul) now. Let's pick the classic `Box` contract for our example.

```solidity
pragma solidity ^0.8.7;

contract Box {
    uint256 private _value;

    event NewValue(uint256 newValue);

    function store(uint256 newValue) public {
        _value = newValue;
        emit NewValue(newValue);
    }

    function retrieve() public view returns (uint256) {
        return _value;
    }
}
```

`Box` is a simple contract that with one state variable `_value` which according to [State Variable Layout Rules](https://docs.soliditylang.org/en/v0.8.13/internals/layout_in_storage.html) is stored in slot 0 of Storage. Moreover, it has two functions to get/set that value. Nothing complicated here.

For the next part, we'll be writing the following simple `Box` contract partially in assembly i.e. using inline assembly. Stay tuned 😉.

## Resources
- [Layout of State Variables in Storage](https://docs.soliditylang.org/en/v0.8.13/internals/layout_in_storage.html)
- [Understanding Ethereum Smart Contract Storage](https://programtheblockchain.com/posts/2018/03/09/understanding-ethereum-smart-contract-storage/)
- [Layout in Memory](https://docs.soliditylang.org/en/v0.8.13/internals/layout_in_memory.html)
- [Layout of Call Data](https://docs.soliditylang.org/en/v0.8.13/internals/layout_in_calldata.html)
- [Function Selector Deep Dive](https://noxx.substack.com/p/evm-deep-dives-the-path-to-shadowy?s=r)
