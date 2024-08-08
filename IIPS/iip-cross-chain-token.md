--- 
iip: <to be assigned>
title: Cross Chain Token Standard
author: Sudeep Simkhada (sudeep.simkhada@venture23.io), Dan Brehmer (dan@venture23.io), Sudeep Bhandari (sudeep@venture23.io)
discussions-to: https://github.com/icon-project/IIPs/discussions/74
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

   HubToken is a token deployed on the ICON chain. It controls token minting/burning in the ICON chain. It also tracks the total supply of the token across all chains where the tokens are deployed. For example, bnUSD is a token that originates in the ICON chain through Balanced. So it is deployed as a hub token in the ICON chain and, if we want to migrate bnUSD to other chains (say SUI chain), we will deploy it as a spoke token in SUI chain. Similarly, if we want to make any token from SUI, say XYZ Token(XYZ), a cross chain token then it is deployed as a spoke token in the SUI chain and its corresponding Hub Token is deployed in the ICON chain. However, there are 3 different ways to implement the Cross Token Standard which is described in details in [Implementation](#implementation). HubToken contract extends the [SpokeToken](#spoketoken) contract. 

2. [SpokeToken](#spoketoken)

   SpokeToken is a token on foreign chain. In ICON, it extends the basic [IRC2](https://github.com/icon-project/IIPs/blob/master/IIPS/iip-2.md) token and has a cross chain transfer function and a function to get balance of a user across the chains. If we want to make a token (from chains other than ICON) a cross chain token, then the token is deployed as a spoke token in its base chains and all other foreign chains except ICON where its Hub Token is deployed. For example, if we want to migrate SUI token to ICON, we will deploy a spoke token contract for SUI token in SUI chain and its Hub Token on ICON chain. Likewise, there are 3 different ways to implement Cross Token Standard for Spoken Tokens as well.

3. [IRC2](https://github.com/icon-project/IIPs/blob/master/IIPS/iip-2.md)
   
   IRC2 is a token standard that provides basic functionality to transfer tokens in ICON. The SpokeToken contract extends this library.

4. [XTokenReceiver](#xtokenreceiver)

   This interface is implemented by both the HubToken and SpokeToken contracts on the ICON chain, if the token contracts need to handle other tokens otherwise this is an optional interface. 
   
   It extends the TokenFallback method of ICON. The function [xTokenFallback](#xtokenfallback) is only callable via XCall services on ICON. It is called when transfer is called from the foreign chain and the receiving address is a contract in ICON. The receiving contract must have implemented `xTokenFallback` in order to get a successful transfer of tokens.

So, for a token to be a cross chain token, it needs to implement the cross chain token standard in their respective chain. A token is deployed as a HubToken in the ICON chain and as a SpokeToken in its base and all other foreign chains except the ICON chain.

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
 * Returns the total supply on the chain to which the spoke token is deployed.
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
 * Transfer tokens from the originating address to the `_to` address.
 * MUST fire the {@code Transfer} event.
 * SHOULD throw if the originating account balance is less than `_value`.
 * If `_to` is a contract, this function MUST invoke the function `tokenFallback(Address, int, bytes)` in `_to`.
 * If the tokenFallback function is not implemented in `_to` (receiver contract), then the transaction must fail and the transfer of tokens should not occur.
 * If `_to` is an externally owned address, then the transaction must be sent without trying to execute `tokenFallback` in `_to`.
 * `_data` can be attached to this token transaction. `_data` can be empty.
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
 * Returns the account balance of another account with string address {@code _owner}.
 * This function MUST support both ICON and NetworkAddress formats.
*/
@External(readonly = true)
BigInteger xBalanceOf(String _owner);
```

##### hubTransfer
```java
/**
 * Method to transfer a spoke token.
 * 
 * @param _to receiver address in string format
 * @param _value amount to send
 * @param _data _data can be empty
*/

/** 
 * {@code _to} MUST support both ICON and NetworkAddress formats.
 * If {@code _to} is an ICON address, use IRC2 transfer.
 * If {@code _to} is a NetworkAddress, transfer {@code _value} amount of tokens to {@code _to}, and MUST fire the {@code HubTransfer} event.
 * This function SHOULD throw if the caller account balance does not have enough tokens to spend.
 * 
 * {@code _to} NetworkAddresses MUST have the following format:
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
HubToken contract extends the [SpokeToken](#spoketoken) interface. It controls token minting/burning and includes a function to track total supply across all connected chains.

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

## Implementation
The motivation of Cross Token Standard is to enable interoperability across multiple chains connected via ICON's GMP. This created 3 different scenarios for its implementation. First, a new cross chain token can be directly deployed extending the Hub Token Implementation on ICON and Spoke Token Implementaion in any other foreign chains. Second, If there already exists a token in any chains, it can be upgraded to cross chain token by extending the respective token standard and upgrading the smart contract. And the third, If there already exists a token in any chains, it can be upgraded to a cross chain token by deploying a Spoke Token manager contract. In this case, token deployer don't have to upgrade their smart contract.

The implementation of the above 3 scenarios are listed below:
1. Deploy New Cross Chain Token 
   If a user wants to deploy a fresh new cross chain token, he/she can just extend the Hub/Spoke Token Standard Implementation library based on their requirement. If the chain is ICON, Hub Token is supposed to be deployed and if it is any other foreign chains then Spoke Token is supposed to be deployed. This way of implementation helps to deploy a native token in any other foreign chains. For example, if a token issuer wants to issue a new cross chain token on ICON, he/she will extend the HubToken Implementation library (and add some custom functionality if necessary) into the token contract and deploy it on ICON, and deploy a SpokeToken Implementaion contract on all other foreign chains.

   * [Hub Token Implementaion on ICON](https://github.com/venture-23/icon-token-standard/blob/development/java-contracts/CrossChainToken/src/main/java/icon/cross/chain/token/lib/tokens/HubTokenImpl.java)

   Example for deploying a new cross chain token on:
      * [ICON](https://github.com/venture-23/icon-token-standard/blob/development/java-contracts/token-examples/NewCrossTokenDeploy/src/main/java/icon/cross/chain/token/examples/NewCrossTokenImpl.java)
      * [SUI](https://github.com/venture-23/icon-token-standard/blob/development/sui-move-contracts/spoke_token/sources/impl/new_cross_token.move)
      * [EVM](https://github.com/venture-23/icon-token-standard/blob/development/solidity-contracts/icon-cross-chain-token/src/implementation/NewCrossToken.sol)

2. Upgrade Existing Token to Cross Chain Token
   If a token issuer wants to upgrade the already existing token to a cross chain token then he/she can upgrade their existing token contract by extending the Cross Token Standard library based on token type if it is ICON extend Hub Token, if any other foreign chains extend Spoke Token. This way of implementation helps to deploy a native token in any other foreign chains. For example, if there exists a token ABC on SUI chain and the issuer wants to upgrade it to cross chain standard then, the issuer will upgrade the ABC token by extending the Spoke Token library on SUI.

   * [Spoke Token Implementation on SUI](https://github.com/venture-23/icon-token-standard/blob/development/sui-move-contracts/spoke_token/sources/spoke_token.move)

   * [Spoke Token Implementation on EVM](https://github.com/venture-23/icon-token-standard/blob/development/solidity-contracts/icon-cross-chain-token/src/tokens/SpokeToken.sol)

3. Deploy Spoke Manager for Existing Token
   If a token issuer wants to upgrade the already existing token to a cross chain token then he/she can do so without upgrading their existing token contract. The token issue can deploy a Spoke Token Manager in order to lock their cross transferred token and release it on withdrawal of the token. This way of implementation will deploy a wrapped version of the token in any other foreign chains. For example, if there exists a XYZ token on EVM chain and the issuer wants to upgrade it to cross token standard then, the issuer will deploy a Spoke Token Manager on EVM chain. The cross transfer function is called from the manager contract which locks the XYZ token and the token is minted through Spoke Token on the destination chain. And if the XYZ token is withdrawn back to native chain, the manager contract releases the fund to destination address.

   * [Spoke Manager Implementation on SUI](https://github.com/venture-23/icon-token-standard/blob/development/sui-move-contracts/spoke_token/sources/spoke_manager.move)
   * [Spoke Manager Implemantation on EVM](https://github.com/venture-23/icon-token-standard/blob/development/solidity-contracts/icon-cross-chain-token/src/tokens/SpokeTokenManager.sol)
   

## References
Existing code developed by the Balanced team.
* [IRC2](https://github.com/icon-project/IIPs/blob/master/IIPS/iip-2.md)

* [SpokeToken](https://github.com/balancednetwork/balanced-java-contracts/blob/1-balanced-on-havah/score-lib/src/main/java/network/balanced/score/lib/interfaces/tokens/SpokeToken.java)

* [HubToken](https://github.com/balancednetwork/balanced-java-contracts/blob/1-balanced-on-havah/score-lib/src/main/java/network/balanced/score/lib/interfaces/tokens/HubToken.java)

* [XTokenReceiver](https://github.com/balancednetwork/balanced-java-contracts/blob/1-balanced-on-havah/score-lib/src/main/java/network/balanced/score/lib/interfaces/tokens/XTokenReceiver.java)



## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
