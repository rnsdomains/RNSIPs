# Registry and Resolver of RNS

|RNSIP          |02         |
| :------------ |:-------------|
|**Title**      |Registry and Resolver of RNS |
|**Created**    | 28-SEP-18 |
|**Author**     |JL |
|**Purpose**    | Usa |
|**Layer**      | 2nd |
|**Complexity** |1 |
|**Status**     |Adopted |


# **Abstract**

The current RNSIP presents Registry and Resolver architecture. These are based on Ethereum Name Service (ENS). Registry is the contract that handles the mapping between domain name and its owner. Each Registry entry references to a Resolver which handles the resolution between name domains and resources. The Resolver is an interface, a Public Resolver contract is provided by RSK and a user can implement their own Resolver contract.

# Motivation

RSK Name Service (RNS) allows users more convenient ways to handle addresses. As DNS resolves IPs and URLs, RNS does the same with addresses and resources.

Existing [specifications](https://github.com/ethereum/wiki/wiki/Registrar-ABI) and [implementations](https://ethereum.gitbooks.io/frontier-guide/content/registrar_services.html) for name resolution in Ethereum provide basic functionality, but suffer several shortcomings that will significantly limit their long-term usefulness:
- A single global namespace for all names with a single 'centralized' resolver.
- Limited or no support for delegation and sub-names/sub-domains.
- Only one record type, and no support for associating multiple copies of a record with a domain.
- Due to a single global implementation, no support for multiple different name allocation systems.
- Conflation of responsibilities: Name resolution, registration, and who is information.

Use-cases that these features would permit include:
- Support for subnames/sub-domains - eg, live.mysite.tld and forum.mysite.tld.
- Multiple services under a single name, such as a DApp hosted in Swarm, a Whisper address, and a mail server.
- Support for DNS record types, allowing blockchain hosting of 'legacy' names. This would permit an RSK client such as Mist to resolve the address of a traditional website, or the mail server for an email address, from a blockchain name.
- DNS gateways, exposing RNS domains via the Domain Name Service, providing easier means for legacy clients to resolve and connect to blockchain services.

The first two use-cases, in particular, can be observed everywhere on the present-day internet under DNS, and we believe them to be fundamental features of a name service that will continue to be useful as the RNS platform develops and matures.

The normative parts of this document do not specify an implementation of the proposed system; its purpose is to document a protocol that different resolver implementations can adhere to in order to facilitate consistent name resolution. An appendix provides sample implementations of resolver contracts and libraries, which should be treated as illustrative examples only.

Likewise, this document does not attempt to specify how domains should be registered or updated, or how systems can find the owner responsible for a given domain. Registration is the responsibility of registrars, and is a governance matter that will necessarily vary between top-level domains.

Updating of domain records can also be handled separately from resolution. Some systems, such as swarm, may require a well-defined interface for updating domains, in which event we anticipate the development of a standard for this.


# **Specification**
## Overview

The RNS system comprises three main parts:
- The RNS Registry
- Resolvers
- Registrars

In this RNSIP will be focused in the two first. For more information about Registrar read the RNS Specification.

The Registry is a single contract that provides a mapping from any registered name to the Resolver responsible for it, and permits the owner of a name to set the Resolver address, and to create subdomains, potentially with different owners to the parent domain.

Resolvers are responsible for performing resource lookups for a name - for instance, returning a contract address, a content hash, or IP address(es) as appropriate. The Resolver specification, defined here and extended in other RNSIPs, defines what methods a Resolver may implement to support resolving different types of records. If users don't set their own Resolver implementation on the Registry entry, a default Resolver is set. This default Resolver is the parent domain’s Resolver of the new owned domain. Then the new domain owner should create a Resolver entry with her new domain, this is explained above.

Resolving a name in RNS is a two-step process. First, the RNS Registry is called with the name to Resolver, after hashing it using the procedure described below. If the record exists, the Registry returns the address of its Resolver. Then, the Resolver is called, using the method appropriate to the resource being requested. The Resolver then returns the desired result.

For example, suppose you wish to find the address of the token contract associated with 'alice.rsk'. First, get the resolver:

```
var node = namehash("alice.rsk");
var resolver = rns.resolver(node);
```

Then, ask the resolver for the address of the contract:

```
var hash = resolver.addr(node);
```

Because the `namehash` procedure depends only on the name itself, this can be precomputed and inserted into a contract, removing the need for string manipulation, and permitting O(1) lookup of RNS records regardless of the number of components in the raw name.

## Registry specification

The RNS Registry contract exposes the following functions:

```
function owner(bytes32 node) constant returns (address);
```

Returns the owner (registrar) of the specified node.

```
function resolver(bytes32 node) constant returns (address);
```

Returns the resolver for the specified node.

```
function ttl(bytes32 node) constant returns (uint64);
```

Returns the time-to-live (TTL) of the node; that is, the maximum duration for which a node's information may be cached.

```
function setOwner(bytes32 node, address owner);
```

Transfers ownership of a node to another registrar. This function may only be called by the current owner of `node`. A successful call to this function logs the event `Transfer(bytes32 indexed, address)`.

```
function setSubnodeOwner(bytes32 node, bytes32 label, address owner);
```

Creates a new node, `sha3(node, label)` and sets its owner to `owner`, or updates the node with a new owner if it already exists. This function may only be called by the current owner of `node`. A successful call to this function logs the event `NewOwner(bytes32 indexed, bytes32 indexed, address)`.

```
function setResolver(bytes32 node, address resolver);
```

Sets the Resolver address for `node`. This function may only be called by the owner of `node`. A successful call to this function logs the event `NewResolver(bytes32 indexed, address)`.

```
function setTTL(bytes32 node, uint64 ttl);
```

Sets the TTL for a node. A node's TTL applies to the 'owner' and 'resolver' records in the Registry, as well as to any information returned by the associated resolver.

```
function setDefaultResolver(address resolver);
```
Sets the default resolver for new nodes

## Resolver specification

Resolvers may implement any subset of the record types specified here. Where a record types specification requires a resolver to provide multiple functions, the resolver MUST implement either all or none of them. Resolvers MUST specify a fallback function that throws.

Resolvers have one mandatory function:

```
function supportsInterface(bytes4 interfaceID) constant returns (bool)
```

The `supportsInterface` function is documented in [EIP 165](https://github.com/ethereum/EIPs/issues/165), and returns true if the resolver implements the interface specified by the provided 4-byte identifier. An interface identifier consists of the XOR of the function signature hashes of the functions provided by that interface; in the degenerate case of single-function interfaces, it is simply equal to the signature hash of that function. If a resolver returns `true` for `supportsInterface()`, it must implement the functions specified in that interface.

`supportsInterface` must always return true for `0x01ffc9a7`, which is the interface ID of `supportsInterface` itself.

 Currently standardized resolver interfaces are specified in the table below.

The following interfaces are defined:

| Interface name | Interface hash | Specification |
| --- | --- | --- |
| `addr` | 0x3b3b57de | [Contract address](#addr) |
| `hash` | 0xd8389dc5 | Hash |

RNSIPs may define new interfaces to be added to this registry.

### <a name="addr"></a>Contract Address Interface

Resolvers wishing to support contract address resources must provide the following function:

```
function addr(bytes32 node) constant returns (address);
```

If the Resolver supports `addr` lookups but the requested node does not have a record, the Resolver MUST return the zero address.

Clients resolving the `addr` record MUST check for a zero-return value, and treat this in the same manner as a name that does not have a Resolver specified - that is, refuse to send funds to or interact with the address. Failure to do this can result in users accidentally sending funds to the 0 address.

Changes to an address MUST trigger the following event:

```
event AddrChanged(bytes32 indexed node, address a);
```
# Appendix A: Registry Implementation

```
contract RNS {
    struct Record {
        address owner;
        address resolver;
        uint64 ttl;
    }

    mapping(bytes32=>Record) records;

    event NewOwner(bytes32 indexed node, bytes32 indexed label, address owner);
    event Transfer(bytes32 indexed node, address owner);
    event NewResolver(bytes32 indexed node, address resolver);

    modifier only_owner(bytes32 node) {
        require(records[node].owner == msg.sender);
        _;
    }

    constructor() public {
        records[bytes32(0)].owner = msg.sender;
    }

    function owner(bytes32 node) constant returns (address) {
        return records[node].owner;
    }

    function resolver(bytes32 node) constant returns (address) {
        return records[node].resolver;
    }

    function ttl(bytes32 node) constant returns (uint64) {
        return records[node].ttl;
    }

    function setOwner(bytes32 node, address ownerAddress) public only_owner(node) {
        emit Transfer(node, ownerAddress);
        records[node].owner = ownerAddress;
    }

    function setSubnodeOwner(bytes32 node, bytes32 label, address ownerAddress) public only_owner(node) {
        bytes32 subnode = keccak256(abi.encodePacked(node, label));
        emit NewOwner(node, label, ownerAddress);
        records[subnode].owner = ownerAddress;

        emit NewResolver(subnode, records[node].resolver);
        records[subnode].resolver = records[node].resolver;
    }

    function setResolver(bytes32 node, address resolverAddress) public only_owner(node) {
        emit NewResolver(node, resolverAddress);
        records[node].resolver = resolverAddress;
    }

    function setTTL(bytes32 node, uint64 ttlValue) public only_owner(node) {
        emit NewTTL(node, ttlValue);
        records[node].ttl = ttlValue;
    }

    function setDefaultResolver(address resolver) public only_owner(0) {
        records[bytes32(0)].resolver = resolver;
    }
}
```
# Appendix B: Sample Resolver Implementations
### Built-in resolver

The simplest possible resolver is a contract that acts as its own name resolver by implementing the contract address resource profile:

```
contract DoSomethingUseful {
    // Other code

    function addr(bytes32 node) constant returns (address) {
        return this;
    }

    function supportsInterface(bytes4 interfaceID) constant returns (bool) {
        return interfaceID == 0x3b3b57de || interfaceID ==  0xd8389dc5;
    }

    function() {
        throw;
    }
}
```

Such a contract can be inserted directly into the RNS registry, eliminating the need for a separate resolver contract in simple use-cases. However, the requirement to 'throw' on unknown function calls may interfere with normal operation of some types of contract.

### Standalone resolver

A basic resolver that implements the contract address profile, and allows only its owner to update records:

```
contract Resolver {
    event AddrChanged(bytes32 indexed node, address a);

    address owner;
    mapping(bytes32=>address) addresses;

    modifier only_owner(bytes32 node) {
        require(records[node].owner == msg.sender);
        _;
    }

    function Resolver() {
        owner = msg.sender;
    }

    function addr(bytes32 node) constant returns(address) {
        return addresses[node];    
    }

    function setAddr(bytes32 node, address addr) only_owner {
        addresses[node] = addr;
        AddrChanged(node, addr);
    }

    function supportsInterface(bytes4 interfaceID) constant returns (bool) {
        return interfaceID == 0x3b3b57de || interfaceID ==  0xd8389dc5;
    }

    function() public {
        revert();
    }
}
```

After deploying this contract, use it by updating the RNS registry to reference this contract by name, then calling `setAddr()` with the same node to set the contract address it will resolve to.

### Public resolver

Similar to the resolver above, this contract only supports the contract address profile, but uses the RNS registry to determine who should be allowed to update entries:

```
contract PublicResolver {
    event AddrChanged(bytes32 indexed node, address a);
    event ContentChanged(bytes32 indexed node, bytes32 hash);

    RNS rns;
    mapping(bytes32=>address) addresses;

    modifier only_owner(bytes32 node) {
        require(rns.owner(node) == msg.sender);
        _;
    }

    constructor(AbstractRNS rnsAddr) public {
        rns = rnsAddr;
    }

    function addr(bytes32 node) constant returns (address ret) {
        ret = addresses[node];
    }

    function setAddr(bytes32 node, address addrValue) public only_owner(node) {
        addresses[node] = addrValue;
    }

    function supportsInterface(bytes4 interfaceID) public pure returns (bool) {
        return interfaceID == 0x3b3b57de || interfaceID == 0xd8389dc5;
    }

    function() public {
        revert();
    }
}
```

# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
