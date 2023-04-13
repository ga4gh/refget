---
title: Seqcol specification version 0.1.0
---

<!-- Table of contents: 
* The generated Toc will be an unordered list
{:toc} -->

<h1>Seqcol specification version 0.1.0</h1>

Table of contents:

[TOC]

## Specification version

This specification is in **DRAFT** form. This is **NOT YET AN APPROVED GA4GH specification**.

## Introduction

Reference sequences are fundamental to genomic analysis. To make analysis of reference sequences reproducible and efficient, we require tools that can identify, store, retrieve, and compare reference sequences. One such tool is the recent GA4GH standard [refget](http://samtools.github.io/hts-specs/refget.html), which provides a way to 1. compute deterministic sequence identifiers from sequences themselves, and 2) retrieve individual sequences using these unique identifiers derived. Refget thus provides a way to identify and retrieve sequences; however, many applications require working with *collections* of sequences. The *Sequence Collections* (seqcol) protocol is a pending GA4GH standard that works with collections of sequences. The seqcol protocol can be used to create unique identifiers derived from multiple sequences. Seqcol uses refget identifiers for individual sequences, adding new functionality to handle the complexity of collections, as well as attributes of sequences, such as names, lengths, or topologies. 

The goal of the Seqcol project is **to standardize unique identifiers for collections of sequences**. It can be used to identify genomes, transcriptomes, or proteomes -- anything that can be represented as a collection of sequences. Seqcol uses a hash algorithm to generate a digest of the underlying collection of sequences. These unique identifiers are defined by an algorithm, rather than an accession authority, and are thus de-centralized and therefore usable for many purposes, including private or new sequence collections, cases without connection to a central database, or validation of sequence collection content and provenance.

Seqcol also specifies a RESTful API to enable retrieving the sequence collections given a unique identifier, which can be used for reproducibility and efficiency, as results can use the unique identifier to specify the exact reference genome used for an analysis. Reproducing the analysis could look up the original genome used. Furthermore, seqcol also provides a standardized method of comparing the contents of two sequence collections. This comparison function can be used to determine if analysis results that used different references genomes may still be compatible. In summary, the project specifies 3 procedures:

1. an algorithm for encoding sequence identifiers from collections
2. a lookup API to retrieve a collection given an identifier
3. a comparison API to assess compatibility of two collections



## Definitions of key terms

- **Sequence**: Seqcol uses refget to store actual sequences, so we use the term in the same way as refget. Refget was designed for nucleotide sequences; however, other sequences could be provided via the same mechanism, *e.g.*, cDNA, CDS, mRNA or proteins. Essentially any ordered list of valid characters qualifies.
- **Sequence collection**: An ordered list of sequences.
- **Digest**: A unique identifier resulting from a cryptographic hash function, such as `MD5` or `SHA512`, on input data.
- **Seqcol digest**: A digest for a sequence collection, computed according to the seqcol algorithm.
- **Seqcol algorithm**: The set of instructions used to compute a digest from a sequence collection.
- **Sequence digest** or **refget digest**: A digest for a sequence, computed according to the refget protocol.
- **Metadata**: Any extra data attached to a sequence collection, including a human-readable alias, source or provider of the collection, version information, etc.
- **Length**: The number of characters in a sequence.
- **Alias**: A human-readable identifier used to refer to a sequence collection.
- **Seqcol API**: The set of endpoints defined in the *retrieval* and *comparison* components of the seqcol protocol.
- **Seqcol protocol**: Collectively, the 3 operations outlined in this document, which include: 1. encoding of sequence collections; 2. retrieval of sequence collections; and 3. comparison of sequence collections.
- **Coordinate system**: A set of named sequence lengths, but without actual sequences.


## Seqcol protocol functionality

The core functionality of the seqcol protocol is to provide three operations:

1. *Encoding* - An algorithm for computing a unique identifier given a collection of sequences;
2. *Retrieval* - A RESTful lookup API specification for retrieving a collection of sequences from a database given a unique identifier.
3. *Comparison* - A standardized API to assess compatibility of two collections

An implementation of the seqcol protocol may implement only one of these functions, but to be fully compliant with the seqcol protocol an implementation must provide all `REQUIRED` capabilities of that function, as detailed below. 

The seqcol algorithm is based on the refget algorithm for individual sequences, and should use refget servers to store the actual sequence data. Seqcol servers therefore provide a lightweight organizational layer on top of refget servers.

### 1. Encoding: Computing sequence digests from sequence collections

The encoding function specifies an algorithm that takes as input a set of annotated sequences and produces a unique digest. This function is generally expected to be provided by local software that operates on a local set of sequences.


#### Standardized sequence collection object representation

The first thing we have to do is create an object representation of the attributes of the sequence collection. The structure of this object is critical, and is strictly controlled by the seqcol protocol. It must be defined in a schema. For example, a basic general schema is:


```YAML
description: "A collection of biological sequences."
type: object
properties:
  lengths:
    type: array
    description: "Number of elements, such as nucleotides or amino acids, in each sequence."
    items:
      type: integer
  names:
    type: array
    description: "Human-readable identifiers of each sequence (chromosome names)."
    items:
      type: string
  sequences:
    type: array
    description: "Digests of sequences computed using the GA4GH digest algorithm (sha512t24u)."
    items:
      type: string
      description: "Actual sequence content"
required:
  - lengths
inherent:
  - lengths
  - names
  - sequences
```

We refer to this as the *seqcol object representation*. An example of a sequence collection organized into the seqcol object representation would be this:

```json
{
  "lengths": [
    8,
    4,
    4
  ],
  "names": [
    "chrX",
    "chr1",
    "chr2"
  ],
  "sequences": [
    "5f63cfaa3ef61f88c9635fb9d18ec945",
    "31fc6ca291a32fb9df82b85e5f077e31",
    "92c6a56c9e9459d8a42b96f7884710bc"
  ],
```

The object is a series of arrays with matching length (`3`), with the corresponding entries collated such that the first element of each array corresponds to the first element of each other array.


#### Seqcol algorithm:

Once the content of the sequence collection has been organized in the *seqcol object representation*, we can then run the digest algorithm to compute the unique identifier for this sequence collection

1. **Step 1**. Put all attributes of the sequence collection (*e.g.* names, lengths, sequence digests) in seqcol object representation form, as defined above.
2. **Step 2**. Apply [RFC-8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785) (JCS) to canonicalize the JSON of the seqcol object.
3. **Step 3**. Digest the canonical representation of each attribute using the seqcol digest algorithm (defined below). This should convert the value of each attribute in the seqcol into a digest string.
4. **Step 4**. Apply [RFC-8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785) again to canonicalize the JSON of new seqcol object representation.
5. **Step 5**. Digest the final canonical representation again. This is the *seqcol digest*, the final unique identifier for this sequence collection.

#### Digest algorithm

The digest algorithm is `TRUNC-512`. Details to follow.

### 2. Retrieval: Retrieving sequence collections from digests

The retrieval function specifies an API endpoint that retrieves original sequences from a database keyed by the unique digest.

#### Endpoints

- `GET /collection/:digest/` - (`REQUIRED`) Here `:digest` is the seqcol digest computed above. This returns the sequence collection identified by the `:digest` variable. The form of the return `MUST` match the seqcol object representation form defined above.

### 3. Comparison: Determining compatibility between sequence collections

The comparison function specifies an API endpoint that allows a user to compare two sequence collections. The output is an assessment of compatibility between those sequence collections. 

#### Endpoints

- `GET /comparison/{digest1}/{digest2}` (`REQUIRED`) for comparing two collections in the database
- `POST /comparison/{digest1}` (`REQUIRED`) for comparing one database collection to a local user-provided collection.

#### Return value


The `/comparison` endpoint must `MUST` return an object in JSON format with these 3 keys: "digests", "arrays", and "elements", as described below:

- `digests`: an object with 2 elements, with keys *a* and *b*, and values either the level 0 seqcol digests for the compared collections, or *undefined (null)*. The value MUST be the level 0 seqcol digest for any digests provided by the user for the comparison. However, it is OPTIONAL for the server to provide digests if the user provided the sequence collection contents, rather than a digest. In this case, the server MAY compute and return the level 0 seqcol digest, or it MAY return *undefined (null)* in this element for any corresponding sequence collection.
- `arrays`: an object with 3 elements, with keys *a-only*, *b-only*, and *a-and-b*. The value of each element is a list of array names corresponding to arrays only present in a, only present in b, or present in both a and b.
- `elements`: An object with 3 elements: *total*, *a-and-b*, and *a-and-b-same-order*. *total* is an object with *a* and *b* keys, values corresponding to the total number of elements in the arrays for the corresponding collection. *a-and-b* is an object with names corresponding to each array present in both collections (in *arrays.a-and-b*), with values as the number of elements present in both collections for the given array. *a-and-b-same-order* is also an object with names corresponding to arrays, and the values a boolean following the same-order specification below.

Example: 

```
{
  "digests": {
    "a": "514c871928a74885ce981faa61ccbb1a",
    "b": "c345e091cce0b1df78bfc124b03fba1c"
  },
  "arrays": {
    "a-only": [],
    "b-only": [],
    "a-and-b": [
      "lengths",
      "names",
      "sequences"
    ]
  },
  "elements": {
    "total": {
      "a": 195,
      "b": 25
    },
    "a-and-b": {
      "lengths": 25,
      "names": 25,
      "sequences": 0
    },
    "a-and-b-same-order": {
      "lengths": false,
      "names": false,
      "sequences": null
    }
  }
}
```

#### Same-order specification

The comparison return includes an *a-and-b-same-order* boolean value for each array that is present in both collections. The defined value of this attribute is:

- *undefined (null)* if there are fewer than 2 overlapping elements
- *undefined (null)* if there are unbalanced duplicates present (see definition below)
- *true* if all matching elements are in the same order in the two arrays
- *false* otherwise.

An *unbalanced duplicate* is used in contrast with a *balanced duplicate*. Balanced means the duplicates are the same in both arrays. When the duplicates are balanced, order is still defined; but if duplicates are unbalanced, this means an array has duplicates not present in the other, and in that case, order is not defined.


This output can be used to make the following comparisons:

- Strict identity. 
- Order-relaxed identity.
- Name-relaxed identity. 
- Length-only compatible. 

