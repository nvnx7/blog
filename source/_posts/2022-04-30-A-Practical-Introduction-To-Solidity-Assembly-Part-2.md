---
title: 'A Practical Introduction To Solidity Assembly: Part 2'
date: 2022-04-30 16:47:56
tags:
- solidity
- ethereum
- smart-contract
---
In the previous part ([Part 1](#)) we understood some facts about solidity compiler and wrote the `Box` contract's functions using inline assembly i.e. in Yul. This part would guide you to writing the `Box` contract in pure assembly. As a reminder here is the `Box` contract with inline assembly from previous part:

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

## Prelude

You've already had a taste of assembly with inline assembly. We're going to use the body of the functions similarly using opcodes to perform operations. Note that when `Box` will be purely written in Yul, it will no longer be a Solidity contract. The solidity compiler only recognizes the inline assembly with the `assembly { }` blocks. So, if you're writing it on Remix IDE, be sure to select Yul compiler.

Before I move into assembly let me explain how exactly the contract is "initiated" by the EVM. Our `Box` contract doesn't have a constructor but that doesn't mean there won't be any initial setup execution going on. The compiled bytecode of a contract has two sections:

### Initialization bytecode
This part of bytecode contains the instruction for setting up any initial state (contract constructor logic) and handing over the copy of **runtime bytecode** to EVM, during the deployment of the contract. After EVM receives **runtime bytecode** it saves it on blockchain storage and associates with an address. Thus, the final bytecode deployed on-chain consists only of the runtime bytecode. Initialization bytecode is executed only once during the deployment of the contract and is not stored in the storage.

For example in the below bytecode of some very tiny contract:
```
600a600c600039600a6000f3602a60505260206050f3
```
The leading `600a600c600039600a6000f3` part is initialization bytecode and rest is runtime bytecode.

Since, we will be writing in pure assembly, any initialization operation will need to be performed manually now. `Box` does not have any constructor, so no initial state variable will be set up. Only thing left to do is to copy the runtime code and return it to EVM. Rest is automatically handled by the EVM itself.

### Runtime bytecode
This comprises of anything (methods, events, constants, etc.) in contract except constructor code. It consists of all execution logic when the contract is interacted with. 

When a contract is interacted through calling a method on it - an encoded calldata is sent to the contract. The contract extracts the first 4 bytes of this calldata, which is the function selector and basically run a series of `if` / `else` statements to decide which function matches it. And execution is then carried out according to matched function (in runtime bytecode).

For example, consider the following calldata:
```
0x6057361d000000000000000000000000000000000000000000000000000000000000000a
```
where `6057361d` is the function selector - which will match `Box`'s `store()` function - which will be invoked after match.

### Basic Yul Structure
A Yul code consists of "blocks" and "sub-blocks" defined by `object` keyword. An `object`. These `object`s are used to group `code` and `data` nodes within them. The `data` node is some static data. And `code` node is executable code of `object`. Within `code` node you write logic using loops, if-else statements, opcodes, etc.

```yul
object "ExampleObject" {
    code {
        // some logic code
    }

    data "ExampleData" hex"123"

    object "ExampleSubObject" {
        code {
            // some other logic code
        }
    }
}
```

See an example [here](https://docs.soliditylang.org/en/v0.8.13/yul.html#specification-of-yul-object).

## Pure Assembly `Box` Contract

A contract consists of a single `object` with sub-`object`s which represent the (runtime) code to be deployed. The `code` nodes are executable code of a `object`. For example we defined/structure `Box` contract as follows:

```yul
object "Box" {
    code {
        // initialization code
    }

    object "runtime" {
        code {
            // runtime code within sub-object
        }
    }
}
```

Let's focus on `initialization` part first and then move on to `runtime` part. 

### Initialization code
As discussed earlier, the `initialization` code must return the runtime code. For this we're required to determine the `offset` (aka position) of the runtime code in the contract's bytecode and its `size`. Then copy this bytecode to the memory so that it can be returned.

The Yul dialect actually provides three specific functions to do this: [`datasize(x)`, `dataoffset(x)` and `datacopy(t, f, l)`](https://docs.soliditylang.org/en/v0.8.13/yul.html#datasize-dataoffset-datacopy) which are used to access other parts of a Yul object.

The `datacopy` of Yul is similar to that of [`codecopy`](https://www.evm.codes/#39) of EVM. It takes the destination offset (in memory) to copy code to, the source offset to copy code from and the length of the code to copy. Then we simply return it with `return` opcode.

`dataoffset` returns offset of the runtime section and `datasize` returns the size of it.

```yul
object "Box" {
    code {
        let runtime_size := datasize("runtime")
        let runtime_offset := dataoffset("runtime")
        datacopy(0, runtime_offset, runtime_size)
        return(0, runtime_size)
    }

    object "runtime" {
        code {
            // runtime code within sub-object
        }
    }
}
```

### Runtime code
Okay, time to sort out the runtime part. Fortunately, with Yul we can write functions just like in we did with inline assembly before. The only tricky part is determining the which function to execute, upon receiving the calldata. Remember from previously discussed that this is done behind the scenes by extracting the function selector (first 4 bytes of calldata) and matching it with the functions defined in the contract. How do we do that? Let's see!

Before solving how to go about function selector thing, first let's define the `retrieve` and `store` functions.

Time to fill-in runtime part - which is almost similar to the functions inline assembly part.

```yul
object "Box" {
    code {
        // ..
    }

    object "runtime" {
        code {
            function retrieve() -> memloc {
                let val := sload(0)
                memloc := 0x80
                mstore(memloc, val)
            }

            function store(val) {
                sstore(0, val)
                mstore(0x80, val)
                log1(0x80, 0x20,0xac3e966f295f2d5312f973dc6d42f30a6dc1c1f76ab8ee91cc8ca5dad1fa60fd)
            }
        }
    }
}
```

Above, the `store()` function is basically the same as before in inline assembly. `retrieve()` however, instead of returning the value upfront, stores it in memory (`mstore(memloc, val)`) returns the memory location at which the value is stored. Reason will become clear later.

It might seem we're done, but not so fast! It's pure assembly. Remember the runtime code part where function selector from calldata is extracted to match the function to be executed? Well, we need to manually do that too. So, let's go!

First, let's calculate the selector from calldata that was sent to it. The calldata may contain encoded arguments of call, apart from the function selector. But we need to extract the first 4 bytes (selector) part of the calldata for time being. The calldata can be read using the [`calldataload`](https://www.evm.codes/#35) opcode which takes only 1 input - the position/offset to read the calldata from. But it returns fixed 32 byte length of data starting from the given position. 

For example, if we invoke `store(10)` (10 = `0xa`) on `Box`, the calldata will look like this:
```
0x6057361d000000000000000000000000000000000000000000000000000000000000000a
```

Now, if you do `calldataload(0)` on above calldata it returns 32 byte value starting from 0:
```
6057361d00000000000000000000000000000000000000000000000000000000
```

How do we extract only first 4 bytes (`6057361d`) from this (`calldataload(0)`) now? Actually, we can use division using `div` opcode, like following:
```
div(
    calldataload(0),
    0x100000000000000000000000000000000000000000000000000000000
)

// gives first four bytes of calldataload(0) i.e. 6057361d
```

But why does it work? Let's take an example with decimal numbers instead of hexadecimal numbers. If `9876543` was calldata and we wanted to extract just first 4 digits using division operation, what would you divide it by (ignore decimals)? Correct, by 1000!
```
9876543 / 1000 = 9876 // considering only integer part
```

Similarly - to get first 4 bytes of 32 byte length (total 64 hex characters) of hex - `calldataload(0)`, we  divide it by `0x100000000000000000000000000000000000000000000000000000000` (56 `0`s).

Based on this let's write a helper function `selector` that extracts function selector from calldata.

```yul
object "Box" {
    code { .. }

    object "runtime" {
        code {
            function retrieve() -> memloc { .. }

            function store(val) { .. }

            function selector() -> s {
                    s := div(calldataload(0), 
                    0x100000000000000000000000000000000000000000000000000000000)
            }
        }
    }
}
```

Only thing left now is to switch to the appropriate function based of selector we just extracted. For this purpose we can utilize Yul's `switch` statement.

When contract is compiled to bytecode every function's selector is also pre-calculated and appended in the bytecode itself. So, that it can be compared against extracted selector from calldata. Hence, we'll also need to pre-calculate selectors of `store()` and `retrieve()` functions. One way to do this is using the `keccak256` hashing of function signature:

```
// retrieve function
bytes4(keccak256("retrieve()")); // 0x2e64cec1

// store function
bytes4(keccak256("store(uint256)")); // 0x6057361d
```

Now with function selectors calculated, we can write the `switch` statement:

```yul
object "Box" {
    code { .. }

    object "runtime" {
        code {
            switch selector() 

            // retrieve function match
            case 0x2e64cec1 {
                let memloc := retrieve()
                return(memloc, 32)
            }

            // store function match
            case 0x6057361d {
                store(calldataload(4))
            }

            // revert if no match
            default {
                revert(0, 0)
            }   

            function retrieve() -> memloc { .. }

            function store(val) { .. }

            function selector() -> s {
                    s := div(calldataload(0), 
                    0x100000000000000000000000000000000000000000000000000000000)
            }
        }
    }
}
```

Remember we returned the memory location from `retrieve()` function instead of value? This is so, because we can use it with `return` opcode!

Another to note is the argument passed to `store()` function above i.e. `calldataload(4)`. `calldataload(4)` would equate to the value argument passed along in the calldata. Since first 4 bytes is selector we skip first 4 bytes and read the next 32 bytes. From the previous calldata example of `store(10)` call:
```
0x6057361d000000000000000000000000000000000000000000000000000000000000000a
```

`calldataload(4)` would return:
```
000000000000000000000000000000000000000000000000000000000000000a
```
which is nothing by 10 in hex.

Finally we have a simple pure assembly (Yul) `Box` contract ready:

```yul
object "Box" {
    code { 
        let runtime_size := datasize("runtime")
        let runtime_offset := dataoffset("runtime")
        datacopy(0, runtime_offset, runtime_size)
        return(0, runtime_size)
    }

    object "runtime" {
        code {
            switch selector() 

            // retrieve function match
            case 0x2e64cec1 {
                let memloc := retrieve()
                return(memloc, 32)
            }

            // store function match
            case 0x6057361d {
                store(calldataload(4))
            }

            // revert if no match
            default {
                revert(0, 0)
            }   

            function retrieve() -> memloc {
                let val := sload(0)
                memloc := 0
                mstore(memloc, val)
            }

            function store(val) {
                sstore(0, val)
                mstore(0, val)
                log1(0x80, 0x20,0xac3e966f295f2d5312f973dc6d42f30a6dc1c1f76ab8ee91cc8ca5dad1fa60fd)
            }

            function selector() -> s {
                    s := div(calldataload(0), 
                    0x100000000000000000000000000000000000000000000000000000000)
            }
        }
    }
}
```

And this is it! Congrats, you made it!

## Resources
- [Yul](https://docs.soliditylang.org/en/v0.8.13/yul.html#yul)
- [EVM dialect]([docs](https://docs.soliditylang.org/en/v0.8.13/yul.html#evm-dialect))
- [Specification of Yul Object](https://docs.soliditylang.org/en/v0.8.13/yul.html#specification-of-yul-object)
- [datasize, dataoffset, datacopy](https://docs.soliditylang.org/en/v0.8.13/yul.html#datasize-dataoffset-datacopy)
