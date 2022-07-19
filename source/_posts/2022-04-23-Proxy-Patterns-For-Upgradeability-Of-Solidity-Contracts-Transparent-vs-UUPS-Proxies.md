---
title: 'Proxy Patterns For Upgradeability Of Solidity Contracts: Transparent vs UUPS Proxies'
date: 2022-04-23 16:43:35
tags:
- solidity
- ethereum
- smart-contract
---


Before you move further I assume you already of some Solidity experience and know how storage slots work in any contract.

## Why Proxies anyway?

If you have following blockchain and smart-contracts stuff you must've come across the word - "immutable". Well, the smart-contracts are immutable. Yes, you cannot tweak any functionality of the contract at that address. You can only interact with it. That's it and it is for the better of it! Otherwise, it wouldn't be so "trustable" if say one day someone in control suddenly dictates the rules in contract in their favor! This is a stark difference from traditional systems where fixes are pushed everyday.


## What could be done?
The drum rolls...proxies! Much has been [researched](https://blog.openzeppelin.com/proxy-patterns/) which led the proxy pattern is the current de-facto for upgrading smart contracts. 

_But wait didn't you just say that contracts are immutable and cannot be changed!_

Of course! And that immutability part still holds true. But proxies work around it.
Despite the much benefits of immutable nature of blockchains pushing bug-fixes and patches in multiple releases cannot be ignored and are much needed for patching bugs and security loopholes. The proxy pattern solves this. Let's see how.


## How Proxies Work

The fundamental idea here is having a Proxy contract. A bit of contextual terminology here before moving on:

**Proxy contract** - A contract that acts as a proxy, delegating all calls to the contract it is the proxy for. In this context, it will also be referred as **Storage Layer**.

**Implementation contract** - The contract that you want to upgrade or patch. This is the contract that Proxy contract will be acting as a proxy for. In this context, it is also the **Logic Layer**.

The Proxy contract stores the address the implementation contract or logic layer, as a state variable. Unlike normal contracts, the user doesn't actually send calls directly to the logic layer - which is your original contract. Instead, all calls go through the proxy and this proxy delegates the calls to this logic layer - the implementation contract at the address that proxy has stored, returning any data it received from logic layer to the caller or reverting for errors.

```
                         delegatecall
User ---------->  Proxy  -----------> Implementation
             (storage layer)          (logic layer)
```

The key thing to note here is that the proxy calls the logic contract through [`delegatecall`](https://eip2535diamonds.substack.com/p/understanding-delegatecall-and-how?s=r) function. Therefore, it is the proxy contract which actually stores state variables i.e. it is the storage layer.  It is like you only borrow the logic from implementation contract (logic layer!) and execute it in proxy's context affecting proxy's state variables in storage.

As an example consider a simple `Box` (implementation) contract, along with a `BoxProxy` (proxy) contract:

```solidity
contract Box {
    uint256 private _value;

    function store(uint256 value) public {
        _value = value;
    }

    function retrieve() public view returns (uint256) {
        return _value;
    }
}

contract BoxProxy {

     function _delegate(address implementation) internal virtual {
         // delegating logic call to boxImpl...
     }

     function getImplementationAddress() public view returns (address) {
         // Returns the address of the implementation contract
     }

     fallback() external {
         _delegate(getImplementationAddress());
     }
}
```

Although `Box` has defined a `uint256` state variable `_value`, it is the `BoxProxy` contract that actually stores the value associated with `_value` (in slot 0, acc. to [storage layout rules](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html)).

The [delegation code](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#proxy-forwarding) is usually put it a [`fallback`](https://solidity.readthedocs.io/en/v0.6.12/contracts.html#fallback-function) function of the Proxy.

The upgrading mechanism is nothing but authorized changing of the implementation contract address stored in proxy contract to point to a whole, newly deployed, upgraded implementation contract. And voila upgrade is complete! Proxy now delegates calls to this new contract. Although that older contract would stick around forever.

```
      upgrade call
Admin -----------> Proxy --x--> Implementation_v1
                     |
                      --------> Implementation_v2
```
Easy right? But there are a few gotchas like potential [storage collision](https://mixbytes.io/blog/collisions-solidity-storage-layouts) between proxy and implementation contract, arising from `delegatecall`.

### Storage collision between Proxy and Implementation contracts

One cannot just go around and simply declare `address implementation` in Proxy contract because that would cause storage collision with the storage of implementation which may have multiple variables in it at overlapping storage slots!

```
|Proxy                   |Implementation |
|------------------------|---------------|
|address implementation  |address var1   | <- collision!
|                        |mapping var2   |
|                        |uint256 var3   |
|                        |...            |
```
Any write to `var1` in implementation would actually write to `implementation` in `Proxy` (storage layer!).

Solution is to choose a pseudo-random slot and write the `implementation` address into that slot. That slot position should be sufficiently random so that having a variable in implementation contract at same slot is negligible.

```
|Proxy                   |Implementation |
|------------------------|---------------|
|    ..                  |address var1   |
|    ..                  |mapping var2   |
|    ..                  |uint256 var3   |
|    ..                  |    ..         |
|    ..                  |    ..         |
|address implementation  |    ..         | <- random slot
```

According to [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967) one such slot could be calculated as:
```solidity
bytes32 private constant implementationPosition = bytes32(uint256(
  keccak256('eip1967.proxy.implementation')) - 1
));
```
Every time implementation address needs to be accessed/modified this slot is read/written.

### Storage collision between different Implementation version contracts
Remember that Proxy is the storage layer. And because of this, when upgrading to a new implementation, if a new state variable is added to implementation contract, it MUST be appended in storage layout. The new contract **MUST extend the storage layout and NOT modify it**. Otherwise, collisions may occur.


**Wrong!** ❌
```
|ImplementationV1 |ImplementationV2|
|-----------------|----------------|
|address foo      |address baz     | <- collision!
|mapping bar      |address foo     | 
|                 |mapping bar     |
|                 |...             |
```

**Right!** ✅
```
|ImplementationV1 |ImplementationV2|
|-----------------|----------------|
|address foo      |address foo     | 
|mapping bar      |mapping bar     | 
|                 |address baz     | <- extended
|                 |...             |
```

### Initializing constructor code
Again, since the proxy is the storage layer, any initialization logic should run inside the proxy - like setting some initial values to state variables. But, you can't proxy call the constructor of the implementation contract. Because the constructor code is run only once - during deployment and it is not part of runtime bytecode. So, there is no way for proxy to directly access constructor bytecode and execute it in its context. 

The solution to this (as per [OpenZeppelin](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#the-constructor-caveat) contracts) is to move the constructor code to a `initializer` function in implementation contract. This is just like a normal function but MUST be ensured that it is called only once.
```solidity
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";

contract MyContract is Initializable {
    // `initializer` modifier makes sure it runs only once
    function initialize(
        address arg1,
        uint256 arg2,
        bytes memory arg3
    ) public payable initializer {
        // "constructor" code...
    }
}
```

### Function clashes between Proxy and Implementation
Since Proxy contract does exist, it is going to need functions of its own. Like a `upgradeTo(address impl)` function, for example. But, then it should decide whether to proxy/delegate the call to implementation or not. What if the implementation contract has a function with the same name i.e. `upgradeTo(address someAddr)`?

There must be a mechanism to determine whether to delegate the call to implementation or not. One such way ([OpenZeppelin](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#transparent-proxies-and-function-clashes) way) is by having an admin or owner address of the Proxy contract. Now, if the admin (i.e. `msg.sender` == `admin`) is making the calls to Proxy, it will not delegate the call but instead execute the function in Proxy itself, if it exists or reverts. For any other address it simply delegates the call to implementation. Thus only an admin address can call `upgradeTo(address impl)` of the Proxy to upgrade to new version of implementation contract.

Considering example of an Ownable ERC20 and and Ownable Proxy contract (`owner` is admin), this is how calls will go.
```
  msg.sender -> | proxy `owner`       | others
----------------|------------------------------------
`owner()`       | proxy.owner()       | erc20.owner()
`upgradeTo(..)` | proxy.upgradeTo(..) | reverts
`transfer(..)`  | reverts             | erc20.transfer(..)
```
All the calls in "other" column were delegated to implementation contract.

## Transparent vs UUPS Proxies
Transparent and UUPS are just different patterns of implementing the proxy pattern to support upgrading mechanism for implementation contracts. There is actually not a very big difference between these two different patterns, in the sense that these share the same interface for upgrades and delegation to implementation contract. 

The difference lies in where actually the upgrade logic resides - Proxy or the Implementation contract.

### Transparent Proxy
In the Transparent Proxy pattern, the upgrade logic resides in the Proxy contract - meaning upgrade is handled by Proxy. A function like `upgradeTo(address newImpl)` must be called to upgrade to a new implementation contract. However, since this logic resides at Proxy, it is expensive to deploy these kind of proxies.

Transparent proxies also require that admin mechanisms to determine whether to delegate the call to implementation or execute a Proxy contract's function. Taking on the `Box` example:

```solidity
contract Box {
    uint256 private _value;

    function store(uint256 value) public { /*..*/ }

    function retrieve() public view returns (uint256) { /*..*/ }
}

contract BoxProxy {

     function _delegate(address implementation) internal virtual { /*..*/ }

     function getImplementationAddress() public view returns (address) { /*..*/ }

     fallback() external { /*..*/ }

     // Upgrade logic in Proxy contract
     upgradeTo(address newImpl) external {
         // Changes stored address of implementation of contract
         // at its slot in storage
     }
}
```

### UUPS Proxy
The UUPS pattern was first documented in [EIP1822](https://eips.ethereum.org/EIPS/eip-1822). Unlike Transparent pattern, in UUPS the upgrade logic is handled by the implementation contract itself. It is the role of implementation to include method for upgrade logic, along with usual business logic. You can make the any implementation contract UUPS compliant by making it inherit a common standard interface that requires one to include the upgrade logic, like inheriting OpenZeppelin's [UUPSUpgradeable](https://docs.openzeppelin.com/contracts/4.x/api/proxy#UUPSUpgradeable1) interface.

```solidity
contract Box is UUPSUpgradeable {
    uint256 private _value;

    function store(uint256 value) public { /*..*/ }

    function retrieve() public view returns (uint256) { /*..*/ }

     // Upgrade logic in Implementation contract
     upgradeTo(address newImpl) external {
         // Changes stored address of implementation of contract
         // at its slot in storage
     }
}

contract BoxProxy {

     function _delegate(address implementation) internal virtual { /*..*/ }

     function getImplementationAddress() public view returns (address) { /*..*/ }

     fallback() external { /*..*/ }

}
```

It is **highly recommended** to inherit this interface to implementation contracts. Because failing to include upgrade logic in a new version implementation (non-UUPS compliant) and upgrading to it will lock the upgrade mechanism forever! Therefore it is recommended that you use libraries (like [UUPSUpgradeable](https://docs.openzeppelin.com/contracts/4.x/api/proxy#UUPSUpgradeable1)) that include measures to prevent this from happening.


## Resources
- [Proxy Upgrade Pattern](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies)
- [OpenZeppelin Proxies](https://docs.openzeppelin.com/contracts/4.x/api/proxy#ERC1967Proxy)
- [Exploring different proxy patterns](https://blog.openzeppelin.com/proxy-patterns/)
- [The Transparent Proxy Pattern](https://blog.openzeppelin.com/the-transparent-proxy-pattern/)
- [EIP-1967: Standard Proxy Storage Slots](https://eips.ethereum.org/EIPS/eip-1967)
- [EIP-1822: Universal Upgradeable Proxy Standard (UUPS)](https://eips.ethereum.org/EIPS/eip-1822)
- [Using OpenZeppeling Contracts with Upgrades](https://docs.openzeppelin.com/contracts/4.x/upgradeable)