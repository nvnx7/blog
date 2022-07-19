---
title: 'A Practical Introduction To Solidity Assembly: Part 1'
date: 2022-04-29 16:47:51
tags:
- solidity
- ethereum
- smart-contract
---

In the previous part ([Part 0](#)) we covered some pre-requisite knowledge of EVM. In this part, we will write the `Box` contract partially in assembly - the function bodies of `retrieve()` and `store()`. Here is our `Box` contract in Solidity:

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

You can interleave Solidity statements with `assembly { }` blocks and can write inline assembly within that code block. As mentioned previously language used for assembly for EVM is [Yul](https://docs.soliditylang.org/en/v0.8.13/yul.html#yul).

## Prelude
Before you move further, I want to clear up some possible confusion. When you compile a contract it is compiled to bytecode. Bytecode is nothing but a long sequence of bytes, something like:

```
608060405234801561001....36f6c63430008070033
```
What does it represent? Well, it is nothing but list of tiny, 1 byte, instructions appended together. These tiny instructions are called **opcodes**. EVM understands/interprets these opcodes during execution of any operation in the contract.

In the above bytecode, for example, the first instruction is `60` (1 byte), which is code for opcode `PUSH1`. You can look up all of the EVM opcodes on [evm.codes](https://www.evm.codes). These opcodes comprise of "EVM dialect".

While writing assembly code, in Yul, you'll notice that Yul provides functions like `add`, `sload`, `store`, `mstore` etc. which are similar to opcodes of EVM dialect - `ADD`, `SLOAD`, `SSTORE`, `MLOAD` respectively. But, EVM dialect is not exactly the same as Yul's. Even though most opcodes from EVM dialect are available in Yul dialect, but not all. E.g. `PUSH1` opcode has no corresponding Yul function. You can check all such Yul functions on the [docs](https://docs.soliditylang.org/en/v0.8.13/yul.html#evm-dialect).

Even though while writing Yul you get fine-grain control, Yul still manages some of the even more low level things like local variables and control flow. So, the order of level of control would be like:

**Solidity < Yul (assembly) < bytecode (opcodes)**

If you would want to go even beyond Yul, you'd be writing raw bytecode for the EVM and won't need the compiler 🤖.

_Note: From now on I may use the term "opcode" to refer to Yul function that corresponds to EVM opcode in the context, since all common EVM opcodes are provided by Yul with similar function signature._

## Inline Assembly

Now, let's use inline assembly to write `retrieve()` and `store()` function bodies of `Box` in assembly. We simply use `assembly { }` blocks. These blocks have access to outside local variables too.

First, focus on `retrieve()` function body. All it does is:
- Read `_value` state variable from storage
- Return the read value

You want opcodes to perform this. [evm.codes](https://www.evm.codes/) is a great reference to search for EVM opcodes. Have a look there and search for our first opcode: [`sload`](https://www.evm.codes/#54).

### `sload`
`sload` takes one input:
- `key` which is Storage slot number (see [Layout of Storage](https://docs.soliditylang.org/en/v0.8.13/internals/layout_in_storage.html)) to be read.

And returns the read value to call stack. 

Go ahead read the slot 0 (`_value` is in slot 0):
```solidity
assembly {
    let v := sload(0) // _value is at slot #0
}
```
Now how do we return it? Right! The [`return`](https://www.evm.codes/#f3) opcode. 

### `return`
It takes two inputs:
- `offset` which is the location of where the value starts from in memory
- `size` which is the number of bytes starting from `offset`.

But `v` returned by `sload` is in call-stack not memory for `return` to work with! So we need to move it to memory first. Enter [`mstore`](https://www.evm.codes/#52).

### `mstore`
It also takes two inputs:
- `offset` which is location (in memory array) where the value should be stored 
- `value` which is bytes to store (which is `v` for us).

Proceeding:
```solidity
assembly {
    let v := sload(0) // read from slot #0
    mstore(0x80, v) // store v at position 0x80 in memory
    return(0x80, 32) // v is 32 bytes (uint256)
}
```

That's it we converted `retrieve()` body to assembly!

_Note: If you're curious as to why I specifically chose **0x80** position in memory to store to store the value, it's because Solidity reserves four 32 byte slots (i.e. 0x00 to 0x7f) for special purposes. So free memory starts from 0x80 initially. While it's ok to just put 0x80 to store new variable in our very simple case, any more complex operation requires you to keep track of pointer to free memory and manage it. Read more in the [docs](https://docs.soliditylang.org/en/v0.8.13/internals/layout_in_memory.html)._

```solidity
function retrieve() public view returns(uint256) {
    assembly {
        // load value at slot 0 of storage
        let v := sload(0) 

        // store into memory at 0x80
        mstore(0x80, v)

        // return value at 0 memory address of size 32 bytes
        return(0x80, 32) 
    }
}
```

The  `store()` function body can be written on the similar lines. I encourage you to reference the opcodes and try to write it on your own before moving further!

Alright, here is the `store()` function body in assembly:

```solidity
function store(uint256 newValue) public {
    assembly {
        // store value at slot 0 of storage
        sstore(0, newValue)

        // emit event
        mstore(0x80, newValue)
        log1(0x80, 0x20,0xac3e966f295f2d5312f973dc6d42f30a6dc1c1f76ab8ee91cc8ca5dad1fa60fd)
    }
}
```

This time we used [`sstore`](https://www.evm.codes/#55) opcode to store new value - the `newValue` param, at the same storage slot (slot 0) where our `_value` state variable is stored. Hence overwriting the old value with new value.

To emit the event (`NewValue`) with new value data, we used [`log1`](https://www.evm.codes/#a1) opcode. First two parameters to it are `offset` in memory and `size` of the data like other opcodes. 

So, we first moved `newValue` to memory at `0x80` position with `mstore`. Then, passed `0x80` as `offset` and `0x20` (32 in decimal) as `size` to `log1` opcode. Now, you might be wondering what is that third argument we passed in to `log1`. It is `topic` - sort of a label for event, like the name - `NewValue`. The passed in argument is nothing but the hash of the event signature:

```solidity
bytes32(keccak256("NewValue(uint256)"))

// 0xac3e966f295f2d5312f973dc6d42f30a6dc1c1f76ab8ee91cc8ca5dad1fa60fd
```

Finally, our `Box` contract looks like this now:


```solidity
pragma solidity ^0.8.7;

contract Box {
    uint256 public value;

    function retrieve() public view returns(uint256) {
        assembly {
            // load value at slot 0 of storage
            let v := sload(0) 

            // store into memory at 0
            mstore(0x80, v)

            // return value at 0 memory address of size 32 bytes
            return(0x80, 32) 
        }
    }

    function store(uint256 newValue) public {
        assembly {
            // store value at slot 0 of storage
            sstore(0, newValue)

            // emit event
            mstore(0x80, newValue)
            log1(0x80, 0x20,0xac3e966f295f2d5312f973dc6d42f30a6dc1c1f76ab8ee91cc8ca5dad1fa60fd)
        }
    }
}
```

In the next, part we'll be writing the `Box` in pure assembly. Buckle up! 😎

## Resources

- [evm.codes](https://www.evm.codes/)
- [Inline Assembly](https://docs.soliditylang.org/en/v0.8.13/assembly.html)
- [Yul](https://docs.soliditylang.org/en/v0.8.13/yul.html#yul)
- [EVM dialect]([docs](https://docs.soliditylang.org/en/v0.8.13/yul.html#evm-dialect))
- [Layout in Memory](https://docs.soliditylang.org/en/v0.8.13/internals/layout_in_memory.html)
- [A trip down Memory Lane](https://noxx.substack.com/p/evm-deep-dives-the-path-to-shadowy-d6b?s=r)