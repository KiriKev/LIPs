---
lip: 0
title: ERC725Account
author: Fabian Vogelsteller <fabian@lukso.network> 
discussions-to: https://discord.gg/E2rJPP4
status: Draft
type: LSP
created: 2021-09-21
requires: ERC165, ERC1271, ERC725X, ERC725Y, LSP1, LSP2, LSP14
---


## Simple Summary

This standard describes a version of an [ERC725](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-725.md) smart contract, that represents a blockchain account.
 
## Abstract

This standard, defines a blockchain account system to be used by humans, machines, or other smart contracts. It has the ability to **attach information** via [ERC725Y](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-725.md#erc725y) to itself, **execute, deploy or transfer value** to any other smart contract or EOA via [ERC725X](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-725.md#erc725x), is able to **be notified of incoming assets** via the [LSP1-UniversalReceiver](./LSP-1-UniversalReceiver.md) function, and can **verify signatures** via [ERC1271](https://eips.ethereum.org/EIPS/eip-1271).


## Motivation

Using EOAs as accounts makes it hard to reason about the actor behind an address. Using EOAs have multiple disadvantages:
- The public key is the address that mostly holds assets, meaning if the private key leaks or get lost, all asstes are lost
- No information can be easily attached to the address thats readable by interfaces or smart contracts
- Security is not changeable, so proper precautions of securing the private key has to be taken from the generation of the EOA.
- Recevied assets can not be tracked in the state of the account, but can only be retrieved to external block explorers.

To make the usage of Blockchain infrastructures easier we need to use a smart contract account, rather that EOAs directly as account system.
This allows us to:

- Make security upgradeable via a key manager smart contract (e.g. [LSP6 KeyManager](./LSP-6-KeyManager.md))
- Allow any action that an EOA can do, and even add the ability to use `create2` through [ERC725X](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-725.md#erc725x)
- Allow the account to be informed and react to receiving assets through [LSP1 UniversalReceiver](./LSP-1-UniversalReceiver.md)
- Define a number of data key-values pairs to attach profile and other information through additional standards like [LSP3 UniversalProfile-Metadata](./LSP-3-UniversalProfile-Metadata.md)
- Allow signature verification through [ERC1271](https://eips.ethereum.org/EIPS/eip-1271)
- can execute any smart contract and deploy smart contracts
- is highly extensible though additional standardisation of the key/value data stored.


## Specification

[ERC165] interface id: `0xdca05671`

_This interface id can be used to detect ERC725Account contracts._

_This `bytes4` interface id is calculated as the XOR of the function selectors from the following interface standards: ERC725Y, ERC725X, LSP1-UniversalReceiver, ERC1271-isValidSignature and ClaimOwnership._

Every contract that supports the LSP0 standard (ERC725Account) SHOULD implement:

### ERC725Y Data Keys


#### LSP1UniversalReceiverDelegate

If the account delegates its universal receiver to another smart contract,
this smart contract address MUST be stored under the following data key:

```json
{
    "name": "LSP1UniversalReceiverDelegate",
    "key": "0x0cfc51aec37c55a4d0b1a65c6255c4bf2fbdf6277f3cc0730c45b828b6db8b47",
    "keyType": "Singleton",
    "valueType": "address",
    "valueContent": "Address"
}
```

Check [LSP1-UniversalReceiver] and [LSP2-ERC725YJSONSchema] for more information.

### Methods

See the [Interface Cheat Sheet](#interface-cheat-sheet) for details.

Contains the methods from:
- [ERC725](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-725.md#specification) (General data key-value store, and general executor)
- [ERC1271](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1271.md#specification)
- [LSP1](./LSP-1-UniversalReceiver.md#specification)
- [LSP14](./LSP-14-Ownable2Step.md#specification)


#### universalReceiver

```solidity
function universalReceiver(bytes32 typeId, bytes memory data) public payable returns (bytes memory)
```

This function is part of the [LSP1-UniversalReceiver] Specification, and do the following:

- Emits [ValueReceived](#events) event when receiving native tokens.

- Forwards the call to the `universalReceiverDelegate(..)` function in the **UniversalReceiverDelegate** contract which address is stored under the data key attached below, if it supports [LSP1UniversalReceiverDelegate InterfaceId](../smart-contracts/interface-ids.md). If there is no address stored under the data key, execution continue normally.

```json
{
  "name": "LSP1UniversalReceiverDelegate",
  "key": "0x0cfc51aec37c55a4d0b1a65c6255c4bf2fbdf6277f3cc0730c45b828b6db8b47",
  "keyType": "Singleton",
  "valueType": "address",
  "valueContent": "Address"
}
```

- Forwards the call to the `universalReceiverDelegate(..)` function in the **TypeIdDelegate** contract which address is stored under the data key attached below, if it supports [LSP1UniversalReceiverDelegate InterfaceId](../smart-contracts/interface-ids.md). If there is no address stored under the data key, execution continue normally.

```json
{
  "name": "LSP1UniversalReceiverDelegate:<bytes32>",
  "key": "0x0cfc51aec37c55a4d0b10000<bytes32>",
  "keyType": "Mapping",
  "valueType": "address",
  "valueContent": "Address"
}
```

> <bytes32\> is the `typeId` passed to the `universalReceiver(..)` function. 

- Emits the [UniversalReceiver](./LSP-1-UniversalReceiver.md#events) event with the typeId and data passed to it, as well as additional parameters such as the amount sent to the function, the caller of the function, and the return value of the delegate contracts abi-encoded.

#### transferOwnership

```solidity
function transferOwnership(address newOwner) external;
```

This function is part of the [LSP14]((./LSP-14-Ownable2Step.md#transferownership)) Specification, with additional requirements as follows:

**Additional requirements:**

- The `newOwner` MUST NOT be the contract itself `address(this)`.

#### fallback

```solidity
fallback() external payable;
```

The fallback function allows for receiving native tokens, as well as arbitrary calldata. The reasoning is that it allows for Graffiti with transactions, or protocols to be built for offchain parsing.

MUST emit a [`ValueReceived`](#valuereceived) event if value was present.

### Events

#### ValueReceived

```solidity
event ValueReceived(address indexed sender, uint256 indexed value);
```

MUST be emitted when a native token transfer was received.

## Rationale

The ERC725Y general data key-value store allows for the ability to add any kind of information to the the account contract, which allows future use cases. The general executor allows full interactability with any smart contract or address. And the universal receiver allows reacting to any future asset received.

## Implementation

An implementation can be found in the [lukso-network/lsp-smart-contracts](https://github.com/lukso-network/lsp-smart-contracts/blob/develop/contracts/LSP0ERC725Account/LSP0ERC725AccountCore.sol) repository.

ERC725Y JSON Schema `ERC725Account`:

```json
[
    {
        "name": "LSP1UniversalReceiverDelegate",
        "key": "0x0cfc51aec37c55a4d0b1a65c6255c4bf2fbdf6277f3cc0730c45b828b6db8b47",
        "keyType": "Singleton",
        "valueType": "address",
        "valueContent": "Address"
    }
]
```

## Interface Cheat Sheet

```solidity
interface ILSP0  /* is ERC165 */ {
         
        
    // ERC725X

    event Executed(uint256 indexed operation, address indexed to, uint256 indexed  value, bytes4 selector);

    event ContractCreated(uint256 indexed operation, address indexed contractAddress, uint256 indexed value);
    
    
    function execute(uint256 operationType, address to, uint256 value, bytes memory data) external payable returns (bytes memory); // onlyOwner
    
    
    // ERC725Y

    event DataChanged(bytes32 indexed dataKey, bytes dataValue);


    function getData(bytes32 dataKey) external view returns (bytes memory dataValue);
    
    function setData(bytes32 dataKey, bytes memory dataValue) external; // onlyOwner

    function getData(bytes32[] memory dataKeys) external view returns (bytes[] memory dataValues);

    function setData(bytes32[] memory dataKeys, bytes[] memory dataValues) external; // onlyOwner

        
    // ERC1271
    
    function isValidSignature(bytes32 hash, bytes memory signature) external view returns (bytes4 magicValue);
    
    
    // LSP0 (ERC725Account)
      
    event ValueReceived(address indexed sender, uint256 indexed value);

    fallback() external payable;
    

    // LSP1

    event UniversalReceiver(address indexed from, uint256 indexed value, bytes32 indexed typeId, bytes receivedData, bytes returnedValue);
    

    function universalReceiver(bytes32 typeId, bytes memory data) external payable returns (bytes memory);

    
    // LSP14

    event OwnershipTransferStarted(address indexed previousOwner, address indexed newOwner);

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    event RenounceOwnershipInitiated();

    event OwnershipRenounced();


    function owner() external view returns (address);
    
    function pendingOwner() external view returns (address);

    function transferOwnership(address newOwner) external; // onlyOwner

    function acceptOwnership() external;
    
    function renounceOwnership() external; // onlyOwner

}


```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

[ERC165]: <https://eips.ethereum.org/EIPS/eip-165>
[LSP1-UniversalReceiver]: <./LSP-1-UniversalReceiver.md>
[LSP2-ERC725YJSONSchema]: <./LSP-2-ERC725YJSONSchema.md>
