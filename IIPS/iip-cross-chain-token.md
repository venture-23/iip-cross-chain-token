--- 
iip: <to be assigned>
title: Cross Chain Token Standard
author: Sudeep Simkhada (sudeep.simkhada@ibriz.ai), Dan Brehmer (dan@venture23.io), Sudeep Bhandari (sudeep@ibriz.ai)
discussions-to: <Discussion link>
status: Draft
type: Standards Track
category: IRC
created: 2024-06-18
--- 

## Simple Summary
A standard interface for native implementation and management of a token across multiple chains using ICON's [GMP](https://github.com/icon-project/IIPs/blob/master/IIPS/iip-52.md).

## Abstract
This document describes the standard token interface required for interoperability across multiple chains connected via ICON's GMP. It enables cross chain transfers between native implementations of the token and management of total supply across all connected chains.

The standard is abstracted from the first such implementation, [bnUSD](https://github.com/balancednetwork/balanced-java-contracts/tree/main/token-contracts/BalancedDollar), where minter contracts are deployed across connected chains to provide a native user experience. This setup ensures seamless operation and accessibility of tokens on multiple blockchains.

## Motivation
The ICON network's focus on cross-chain interoperability depends on creating low friction paths for developers to take advantage of its General Message Passing protocol. This token standard specifies how to implement token contracts that can access unified and concentrated liquidity pools on Balanced regardless of what chain they are starting from. By enabling the straightforward migration of tokens from single chain to multi-chain implementations, ICON aims to attract builders who want to deliver smooth token experiences to their end users and leverage novel technologies that exist across multiple chains.

## Specification
The cross chain token standard consists of two main libraries: HubToken and SpokeToken, and two extended libraries: IRC2 and XTokenReceiver. 
1. [HubToken](#hubtoken)

   HubToken is a token from the base chain. It controls token minting in the base chain. It also tracks the total supply of the token across all chains where the tokens are deployed. For example, bnUSD is a token that originates in the ICON chain through Balanced. So it is deployed as a hub token in the ICON  chain and, if we want to migrate bnUSD to other chains (say SUI chain), we will deploy it as a spoke token in SUI chain. Similarly, if we want to make any token from SUI, say PumpUp(PUP), a cross chain token then it is deployed as a hub token in SUI chain and as a spoke token in ICON chain.  HubToken contract extends the [SpokeToken](#spoketoken) contract.

2. [SpokeToken](#spoketoken)

   SpokeToken is a token from a foreign chain. In ICON, it extends the basic [IRC2](https://github.com/icon-project/IIPs/blob/master/IIPS/iip-2.md) token and has a cross chain transfer function and a function to get balance of a user across the chains. If we want to move a token from a base chain to some foreign chains, the token is deployed as a spoke token in all other foreign chains. For example, if we want to migrate SUI token to ICON, we will deploy a spoke token contract for SUI token in the ICON chain.

3. [IRC2](https://github.com/icon-project/IIPs/blob/master/IIPS/iip-2.md)
   
   IRC2 is a token standard that provides basic functionality to transfer tokens in ICON. The SpokeToken contract extends this library.

4. [XTokenReceiver](#xtokenreceiver)

   This interface is implemented by both the HubToken and SpokeToken contracts on the ICON chain, if the token contracts need to handle other tokens otherwise this is an optional interface. 
   
   It extends the TokenFallback method of ICON. The function [xTokenFallback](#xtokenfallback) is only callable via XCall services on ICON. It is called when transfer is called from the foreign chain and the receiving address is a contract in ICON. The receiving contract must have implemented `xTokenFallback` in order to get a successful transfer of tokens.

So, for a token to be a cross chain token, it needs to implement the cross chain token standard in their respective chain. A token is deployed as a HubToken in its base chain and as a SpokeToken in other foreign chains.

### SpokeToken
SpokeToken contract extends the basic token standard of the respective chains. In ICON, it extends basic IRC2 token and has a cross chain transfer function.

#### Methods

##### name
```java
/**
 * Returns the name of the token. e.g. `CrossChainToken`.
*/
@External(readonly=true)
public String name();
```

##### symbol
```java
/**
 * Returns the symbol of the token. e.g. `CCT`.
*/
@External(readonly=true)
public String symbol();
```

##### decimals
```java
/**
 * Returns the number of decimals the token uses. e.g. `18`.
*/
@External(readonly=true)
public BigInteger decimals();
```

##### totalSupply
```java
/**
 * Returns the total token supply.
*/
@External(readonly=true)
public BigInteger totalSupply();
```

##### balanceOf
```java
/**
 * Returns the account balance of another account with address {@code _owner}.
*/
@External(readonly=true)
public BigInteger balanceOf(Address _owner);
```

##### transfer
```java
/**
 * Transfer tokens from one address to another and MUST fire the {@code Transfer} event.
 * 
 * @param _to token receiver address 
 * @param _value amount to send
 * @param _data _data is empty
*/
@External
void transfer(Address _to, BigInteger _value, @Optional byte[] _data);
```

##### xBalanceOf
```java
/**
 * Returns the account balance of another account with string address {@code _owner},
 * which can be both ICON and NetworkAddress format.
*/
@External(readonly = true)
BigInteger xBalanceOf(String _owner);
```

##### hubTransfer
```java
/**
 * Method to transfer spoke token.
 * 
 * @param _to receiver address in string format
 * @param _value amount to send
 * @param _data _data can be empty
*/

/** 
 * If {@code _to} is a ICON address, use IRC2 transfer
 * Transfers {@code _value} amount of tokens to NetworkAddress {@code _to}, and MUST fire the {@code HubTransfer} event.
 * This function SHOULD throw if the caller account balance does not have enough tokens to spend.
 * 
 * The format of {@code _to} if it is NetworkAddress:
 * "<Network Id>.<Network System>/<Account Identifier>"
 * Examples:
 * "0x1.icon/hxc0007b426f8880f9afbab72fd8c7817f0d3fd5c0",
 * "0x5.moonbeam/0x5425F5d4ba2B7dcb277C369cCbCb5f0E7185FB41"
*/
@External
void hubTransfer(String _to, BigInteger _value, @Optional byte[] _data);
```

#### Cross Chain Methods
##### xHubTransfer
```java
/**
 * This method is callable only via XCall service on ICON. XCall triggers the handleCallMessage of the spoke token contract and the data from the transaction is decoded by the XCall processor. The first value decoded from data is always the method name. If the method name is `XHubTransfer` then the data is decoded based on the method signature and the method is called.
 * 
 * @param from sender NetworkAddress
 * @param _to receiving address, can be account or contract address
 * @param _value amount to transfer
 * @param _data call data and can be empty
*/

/** 
 * Transfers {@code _value} amount of tokens to address {@code _to}, and MUST fire the {@code HubTransfer} event.
 * This function SHOULD throw if the caller account balance does not have enough tokens to spend.
 * If {@code _to} is a contract, this function MUST invoke the function {@code xTokenFallback(String, int, bytes)} in {@code _to}.
 * If the {@code xTokenFallback} function is not implemented in {@code _to} (receiver contract), then the transaction must fail and the transfer of tokens should not occur.
 * If {@code _to} is an externally owned address, then the transaction must be sent without trying to execute {@code XTokenFallback} in {@code _to}.
*/
void xHubTransfer(String from, String _to, BigInteger _value, byte[] _data);
```

#### Eventlogs

##### Transfer
```java
/**
 * Must trigger on any successful token transfers.
*/
@EventLog
void Transfer(Address _from, Address _to, BigInteger _value, byte[] _data);
```

##### HubTransfer
```java
/**
 * Must trigger on any successful hub token transfers.
*/
@EventLog
void HubTransfer(String _from, String _to, BigInteger _value, byte[] _data);
```

### HubToken
HubToken contract extends the [SpokeToken](#spoketoken) interface. It controls token minting and includes a function to track total supply across all connected chains.

#### Methods

##### xTotalSupply
```java
/**
 * Returns the total token supply across all connected chains.
*/
@External(readonly = true)
BigInteger xTotalSupply();
```

##### xSupply
```java
/**
 * Returns the total token supply on a connected chain.
*/
@External(readonly = true)
BigInteger xSupply(String net);
```

##### getConnectedChains
```java
/**
 * Returns a list of all contracts across all connected chains
*/
@External(readonly = true)
String[] getConnectedChains();
```

##### crossTransfer
```java
/**
 * Method to transfer hub token.
 * 
 * @param _to NetworkAddress to send to
 * @param _value amount to send
 * @param _data used in tokenFallbacks
*/

/**
 * If {@code _to} is a ICON address, use IRC2 transfer
 * If {@code _to} is a NetworkAddress, then the transaction must trigger xTransfer via XCall on corresponding spoke chain and MUST fire the {@code XTransfer} event.
 * {@code _data} can be attached to this token transaction.
 * {@code _data} can be empty.
 * XCall rollback message is specified to match [xTransferRevert](#xcrosstransferrevert).
 * 
 * The format of {@code _to} if it is NetworkAddress: 
 * "<Network Id>.<Network System>/<Account Identifier>"```
 * Examples:
 * "0x1.icon/hxc0007b426f8880f9afbab72fd8c7817f0d3fd5c0",
 * "0x5.moonbeam/0x5425F5d4ba2B7dcb277C369cCbCb5f0E7185FB41"
*/
@External
@Payable
void crossTransfer(String _to, BigInteger _value, byte[] _data);
```

#### Cross Chain Methods
##### xCrossTransfer
```java
/**
 * This is a method for processing cross chain transfers from spokes. It is callable via XCall only. The XCall processor decodes the transaction data and if the method name in the decoded data is `xCrossTransfer` then it calls this function.
 * 
 * @param _from from NetworkAddress
 * @param _to NetworkAddress to send to
 * @param _value amount to send
 * @param _data used in tokenFallbacks
*/

/**
 * If {@code _to} is a contract trigger xTokenFallback(String, int, byte[]) instead of regular tokenFallback.
 * Internal behavior same as [xTransfer](#xtransfer) but the {@code from} parameter is specified by XCall rather than the blockchain.
*/
void xCrossTransfer(String from, String _from, String _to, BigInteger _value, byte[] _data);
```

##### xCrossTransferRevert
```java
/**
 * This method is callable via XCall only, and is called when the cross transfer transaction is reverted. XCall processor decodes the transaction data and if the method name in decoded data is `xCrossTransferRevert` then this function is called.
*/
void xCrossTransferRevert(String from, String _to, BigInteger _value);
```

##### xTransfer
```java
/**
 * Method for transferring hub balances to a spoke chain. It is callable via XCall only. The XCall processor decodes the transaction data and if the method name in decoded data is `xTransfer` then it calls this function.
 * 
 * @param from EOA address of a connected chain
 * @param _to native address on calling chain
 * @param _value amount to send
 * @param _data used in tokenFallbacks
 * 
 * Uses {@code from} to xTransfer the balance on ICON to native address on a calling chain.
*/
void xTransfer(String from, String _to, BigInteger _value, byte[] _data);
```

#### Eventlogs

##### XTransfer
```java
/**
 * Must trigger on any successful token transfers from cross-chain addresses.
*/
@EventLog(indexed = 1)
void XTransfer(String _from, String _to, BigInteger _value, byte[] _data);
```

### XTokenReceiver
This contract is to handle the cross chain token transfer to the contracts in the ICON chain. It extends the TokenFallback method of ICON. The function [xTokenFallback](#xtokenfallback) is only callable via XCall services on ICON. It is called when transfer is called from the foreign chain and the receiving address is a contract in ICON. The receiving contract must have implemented ```xTokenFallback``` inorder to get a successful transfer of tokens.

#### Cross Chain Methods

##### xTokenFallback
```java
/**
 * If the token issuer wants the contract to handle tokens in the ICON chain then they need to implement this method otherwise it is an optional method. 
 * Receives cross chain enabled tokens. This method is called if the {@code _to} in a XCall initiated method is a contract address.
 * 
 * @param _from NetworkAddress pointing to an address on a XCall connected chain
 * @param _value amount to receive
 * @param _data used in tokenFallbacks
*/
void xTokenFallback(String _from, BigInteger _value, byte[] _data);
```


### References
Existing code developed by the Balanced team.
* [IRC2](https://github.com/icon-project/IIPs/blob/master/IIPS/iip-2.md)

* [SpokeToken](https://github.com/balancednetwork/balanced-java-contracts/blob/1-balanced-on-havah/score-lib/src/main/java/network/balanced/score/lib/interfaces/tokens/SpokeToken.java)

* [HubToken](https://github.com/balancednetwork/balanced-java-contracts/blob/1-balanced-on-havah/score-lib/src/main/java/network/balanced/score/lib/interfaces/tokens/HubToken.java)

* [XTokenReceiver](https://github.com/balancednetwork/balanced-java-contracts/blob/1-balanced-on-havah/score-lib/src/main/java/network/balanced/score/lib/interfaces/tokens/XTokenReceiver.java)