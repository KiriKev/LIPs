---
lip: 9
title: Vault
author: 
discussions-to: https://discord.gg/E2rJPP4
status: Draft
type: LSP
created: 2021-09-21
requires: ERC165, ERC725X, ERC725Y, LSP1, LSP2, LSP14
---


## Simple Summary

This standard describes a version of an [ERC725](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-725.md) smart contract, that represents a blockchain vault.
 
## Abstract

This standard defines a vault that can hold assets and interact with other contracts. It has the ability to **attach information** via [ERC725Y](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-725.md#erc725y) to itself, **execute, deploy or transfer value** to any other smart contract or EOA via [ERC725X](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-725.md#erc725x). It can be **notified of incoming assets** via the [LSP1-UniversalReceiver](./LSP-1-UniversalReceiver.md) function.


## Motivation


## Specification

[ERC165] interface id: `0xca86ec0f`

_This interface id can be used to detect Vault contracts._

_This `bytes4` interface id is calculated as the XOR of the function selectors from the following interface standards: ERC725Y, ERC725X, LSP1-UniversalReceiver and ClaimOwnership._

### ERC725Y Data Keys

Every contract that supports the LSP9 standard SHOULD implement:

#### SupportedStandards:LSP9Vault

The supported standard SHOULD be `LSP9Vault`

```json
{
    "name": "SupportedStandards:LSP9Vault",
    "key": "0xeafec4d89fa9619884b600007c0334a14085fefa8b51ae5a40895018882bdb90",
    "keyType": "Mapping",
    "valueType": "bytes4",
    "valueContent": "0x7c0334a1"
}
```

#### LSP1UniversalReceiverDelegate

If the contract delegates its universal receiver to another smart contract,
this smart contract address MUST be stored under the following data key:

```json
{
    "name": "LSP1UniversalReceiverDelegate",
    "key": "0x0cfc51aec37c55a4d0b1a65c6255c4bf2fbdf6277f3cc0730c45b828b6db8b47",
    "keyType": "Singleton",
    "valueContent": "Address",
    "valueType": "address"
}
```

Check [LSP1-UniversalReceiver] and [LSP2-ERC725YJSONSchema] for more information.

### Methods

See the [Interface Cheat Sheet](#interface-cheat-sheet) for details.

**Contains the methods from** [ERC725](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-725.md#specification) (General data key-value store, and general executor) with overriding:

#### execute

```solidity
function execute(uint256 operationType, address to, uint256 value, bytes memory data) public payable returns(bytes memory)
```

Executes a call on any other smart contracts, transfers the blockchains native token, or deploys a new smart contract.
MUST only be called by the current owner of the contract.

The following `operationType` MUST exist:

- `0` for `CALL`
- `1` for `CREATE`
- `2` for `CREATE2`
- `3` for `STATICCALL`

The following `operationType` COULD exist:

- `4` for `DELEGATECALL` - **NOTE** This is a potentially dangerous operation

Check [`execute(...)`](https://github.com/ERC725Alliance/ERC725/blob/develop/docs/ERC-725.md#execute) function of [ERC725X](https://github.com/ERC725Alliance/ERC725/blob/develop/docs/ERC-725.md#erc725x) for more details.

**Contains the methods from:**

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

### Events

#### ValueReceived

```solidity
event ValueReceived(address indexed sender, uint256 indexed value);
```

MUST be emitted when a native token transfer was received.

## Rationale

The ERC725Y general data key value store allows for the ability to add any kind of information to the contract, which allows future use cases. The general execution allows full interactability with any smart contract or address. And the universal receiver allows the reaction to any future asset.

## Implementation

An implementation can be found on the [lsp-universalprofile-smart-contracts](https://github.com/lukso-network/lsp-universalprofile-smart-contracts/tree/main/contracts/LSP9Vault) repository;

ERC725Y JSON Schema:

```json
[
    {
        "name": "SupportedStandards:LSP9Vault",
        "key": "0xeafec4d89fa9619884b600007c0334a14085fefa8b51ae5a40895018882bdb90",
        "keyType": "Mapping",
        "valueType": "bytes4",
        "valueContent": "0x7c0334a1"
    },
    {
        "name": "LSP1UniversalReceiverDelegate",
        "key": "0x0cfc51aec37c55a4d0b1a65c6255c4bf2fbdf6277f3cc0730c45b828b6db8b47",
        "keyType": "Singleton",
        "valueContent": "Address",
        "valueType": "address"
    }
]
```

## Interface Cheat Sheet

```solidity

interface ILSP9  /* is ERC165 */ {    

    
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
        

    // LSP1

    event UniversalReceiver(address indexed from, uint256 indexed value, bytes32 indexed typeId, bytes receivedData, bytes returnedValue);
    

    function universalReceiver(bytes32 typeId, bytes memory data) external payable returns (bytes memory);


    // LSP9 
      
    event ValueReceived(address indexed sender, uint256 indexed value);

    fallback() external payable;


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
