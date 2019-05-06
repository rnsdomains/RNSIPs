Multi-chain Resolver
===

| RNSIP | 03 |
| - | - |
| **Title** | Multi-chain Resolver |
| **Created** |	23-APR-19 |
| **Author** | IO, AB |
| **Purpose** | Usa |
| **Layer** | 2nd |
| **Complexity** | 1 |
| **Status** | Draft |

## Abstract

The current RNSIP describes a protocol that different resolver implementations can adhere to in order to facilitate consistent multi-chain resolutions, keeping the address-resolution data active and accessible.

## Motivation

RNS enables users more convenient ways to handle addresses. [RNSIP02](https://github.com/rnsdomains/RNSIPs/blob/master/IPs/RNSIP02.md) describes an interface for Resolvers wishing to support address resources. The Public Resolver’s actual implementation provides a structure that allows only one address to be stored for each name.

Currently, blockchain users manage multiple types of coins and assets. This can be observed everywhere on the present-day where wallets support different types of currencies, exchanges allow subscribing and withdrawing different assets, and services receive any payment method.

The Public Resolver protocol is short to cover this use-case since it only supports single-address resolution. However, it is in active use, so migration to a new contract or structure is not desirable.

## Specification

### Functionality

Today there are more than 800 cryptocurrencies in circulation<sup>[1](https://coinmarketcap.com/coins/views/all/)</sup>. The address-storage model must be flexible enough to store most of them.

Each chain has a hexadecimal value identifier, defined in [SLIP-0044](https://github.com/satoshilabs/slips/blob/master/slip-0044.md).

Resolvers wishing to support multi-chain address resources must provide the following function:

```
function chainAddr(bytes32 node, bytes4 chain) constant returns (string);
```

- `node`: RNS node to get the chain address from.
- `chain`: Chain identifier described above.
- Returns an address of the node and chain specified.

If the Resolver supports `chainAddr` lookups but the requested node does not have a record, the Resolver must return an empty string.

Clients resolving the `chainAddr` record must check for a `length < 1` value, and treat this in the same manner as a name that does not have a specified chain address resolution - that is, refuse to send funds to or interact with the address. Failure to do this can result in users accidentally sending funds to the 0 address of any chain allowing this behavior.

The function signature is `0x8be4b5f6`. This must return true on `supportsInterface` method.

Changes to a chain address must trigger the following event:

```
event ChainAddrChanged(bytes32 indexed node, bytes4 chain, string addr);
```

### Backwards Compatibility

According to [RNSIP02](https://github.com/rnsdomains/RNSIPs/blob/master/IPs/RNSIP02.md), any resolver must implement `supportsInterface` method and throw on fallback function. If it implements `addr` interface, it must emit `AddrChanged` event.

Public Resolver has one resource to keep compatibility with: `addr`. For any name using the Public Resolver that is already resolving `addr`, should keep this value accessible from the Multi Chain Resolver. When the owner decides to change the value of any of these two resources, the new value is stored and replaces the value in the Public Resolver.

## Implementation

### Storage

This contract should store one string byte for each network of each RNS node.

```
mapping (bytes32 => mapping (bytes4 => string)) chainAddresses;
```

### Backward Compatibility

To keep compatibility with RNSIP02:

1. It should implement `addr`

    ```
    bytes4 constant RSK_CHAIN_ID = 0x80000089;

    function addr (bytes32 node) public view returns (address) {
        string memory _addr = chainAddresses[node][RSK_CHAIN_ID];

        if (bytes(_addr).length > 0) {
            return addressHelper.stringToAddress(_addr);
        }

        return publicResolver.addr(node);
    }
    ```

2. It should implement `supportsInterface`:

    ```
    bytes4 constant ADDR_SIGN = 0x3b3b57de;
    bytes4 constant CHAIN_ADDR_SIGN = 0x8be4b5f6;

    function supportsInterface (bytes4 interfaceId) public pure returns (bool) {
        return ((interfaceId == ADDR_SIGN) || interfaceId == (CHAIN_ADDR_SIGN));
    }
    ```

3. It should emit `AddrChanged`:

    ```
    function setAddr (bytes32 node, address addrValue) public onlyOwner(node) {
        chainAddresses[node][RSK_CHAIN_ID] = addressHelper.addressToString(addrValue);
        emit AddrChanged(node, addrValue);
    }
    ```

### Multi-chain functionality

To be RNSIP03:

1. It should implement `chainAddr`:

    ```
    function chainAddr (bytes32 node, bytes4 chain) public view returns (string memory) {
        return chainAddresses[node][chain];
    }
    ```

2. It should emit `ChainAddrChanged`:

    ```
    function setChainAddr (bytes32 node, bytes4 chain, string memory addrValue) public onlyOwner(node) {
        chainAddresses[node][chain] = addrValue;
        if (chain == RSK_CHAIN_ID) {
            address _addr = addressHelper.stringToAddress(addrValue);
            emit AddrChanged(node, _addr);
        } else {
            emit ChainAddrChanged(node, chain, addrValue);
        }
    }
    ```
