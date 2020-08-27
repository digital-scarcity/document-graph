# Document Graph

## Simple Summary
This specification defines an abstraction data structure that can be used for flexible/variant key-value pairs in a smart contract. 

## Use Cases
- Facilitates creation/evolution of consensus among humans on human-readable and machine-interpreted documents, such as real-world contracts
- On-chain enforcement of business rules contained in documents
- Easier handling of variant data within a smart contract, particularly change management (devX)
- Greatly improved query and access capabilities

## Features
- [Content addressing](https://docs.ipfs.io/concepts/how-ipfs-works/#content-addressing)
- Akin to IPFS log or Git, documents are read-only and changes are appended as a new 'document' (node in the graph)
- Document content, not just an anchored hash, is stored on chain so that it can be used in smart contracts
- (*perhaps implementation specific*) Documents can be certified or 'signed' by other accounts
- Supports key eosio data types: ```name, int64, asset, time_point, string, and checksum256```
- Data elements (see below) support both optional labeling and sequencing
- Markdown support

## Data Structure
The core on-chain storage element is variant:

``` c++
    // flexvalue can any of these commonly used eosio data types
    // a checksum256 can be a link to another document, akin to an "edge" on a graph
    typedef std::variant<name, string, asset, time_point, int64_t, checksum256> flexvalue;
```

Flex values may be labeled as they are encapsulated into a content object.
``` c++
      struct content
      {
         string label;
         flexvalue value;
      };
```

Which are grouped: 
``` c++
    typedef vector<content> content_group;
```

And a ```document``` supports a vector of ```content_group```, a hash of these groups, plus additional metadata:
``` c++
    struct [[eosio::table]] document
    {
        uint64_t                id;
        checksum256             hash;
        name                    creator;
        vector<content_group>   content_groups;
        vector<certificate>     certificates;

        uint64_t primary_key() const { return id; }
        uint64_t by_creator() const { return creator.value; }
        checksum256 by_hash() const { return hash; }

        time_point created_date = current_time_point();
        uint64_t by_created() const { return created_date.sec_since_epoch(); }
    };
```

#### Certificates (implementation specific?)
Any document may be ```certified```, which is meant as an attestation, approval, signature, or a related use case depending on the specific needs. 

> Certificates are not required generally but they are a key element in multiple use cases for which this spec was designed for.
``` c++
    struct certificate
    {
        name        certifier;
        string      notes;
        time_point  certification_date = current_time_point();
    };
```

Of course, certificate creation requires the authority of the certifier.

## Content Addressing
*This can surely be improved upon for performance and scale, likely using bytes instead of strings, which are used in the imlementation for clarity*

In the current implementation, a ```document``` hash is a ```checksum256``` of the concatenation of the string representation of the ```vector<content_group>```.

If a 2nd identical document is attempted to be created, it fails, ensuring automatic de-duplication of content. 

Since certificates are added after ```document``` creation, they are of course not included in the hashing fingerprint. 

### Fork (Edit)

A ```document``` may be edited or 'forked'.  When this occurs, a new ```document``` is created with a link (graph edge) back to the parent ```document``` along with any content that is different than the parent.  Only diffs are stored.

A client/reader of the ```document``` can walk the graph to construct a current version. (TODO: bounds test recursion; address depth issues)

## Graph Access / Query 

Querying EOSIO via ```get_table_rows``` is notoriously heavy and inefficient. Entire rows are always returned, there is no support for joining data, only a single index can be used in each query, etc. 

As this data structure is a Graph, a key component is integration to a Graph-friendly database for improved access.

### DGraph
In this implementation, we use [DGraph](https://dgraph.io) for various reasons, but any Graph DB should be fine. Of course, proper security must be addressed when inserting records, and then the data is read-only by clients. 

Documents may be loaded with block streaming services such as dfuse or Hyperion.  Data is updated anytime a ```document``` is created or certified.

The schema (WIP):
*TODO: much improvement needed, but this works for basic insert/query*
``` graphql
      type Document {
        hash
        created_date
        creator
        content_groups
        certificates
      }
      
      type ContentGroup {
        content_group_sequence
        contents
      }
      
      type Content {
        label
        value
        type
        content_sequence
      }
      
      type Certificate {
        certifier
        notes
        certification_date
      }
      
      hash: string @index(exact) .
      created_date: datetime .
      creator: string @index(term) .
      content_groups: [uid] .
      certificates: [uid] .
      
      content_group_sequence: int .
      contents: [uid] .
      
      label: string @index(term) .
      value: string @index(term) .
      type: string @index(term) .
      content_sequence: int .
      
      certifier: string @index(term) .
      notes: string .
      certification_date: datetime .
```

## Implementation

The current implementation, a work-in-process, is located: https://github.com/hypha-dao/document

Test data has been loaded into a contract on the Telos testnet.  

Quick look:
```
cleos -u https://test.telos.kitchen get table -r docs.hypha docs.hypha documents
```

### npm package
An npm package abstracts the transformation of documents between the EOSIO required format and the DGraph required format so that it is transparent to clients. 

<img width="682" alt="image" src="https://user-images.githubusercontent.com/32852271/91439152-7f514080-e83a-11ea-8853-85026fffa39f.png">

### Vanilla walker and diff viewer
Simple GUI for navigating a graph

## IPLD integration (extra credit)
#### What is IPLD?
> IPLD is the data model of the content-addressable web. It allows us to treat all hash-linked data structures as subsets of a unified information space, unifying all data models that link data with hashes as instances of IPLD.

> Content addressing through hashes has become a widely-used means of connecting data in distributed systems, from the blockchains that run your favorite cryptocurrencies, to the commits that back your code, to the web’s content at large. Yet, whilst all of these tools rely on some common primitives, their specific underlying data structures are not interoperable.

> Enter IPLD: a single namespace for all hash-inspired protocols. Through IPLD, links can be traversed across protocols, allowing you to explore data regardless of the underlying protocol.

Implementing ```document``` as an IPLD type will enable seamless integration with [IPLD Explorer](https://explore.ipld.io/) and other commonly used web3 technologies, independent of the underlying protocol.

### multiaddr 
See: https://github.com/multiformats/go-multiaddr

## Open
- should we support multi-directional graphs or limit to [DAGs](https://docs.ipfs.io/concepts/merkle-dag/)?
- archival strategies, such as pinning
- contextual business rules will be used when constructing documents but it's probably not part of this spec
- additional document rendering instructions beyond markdown support?