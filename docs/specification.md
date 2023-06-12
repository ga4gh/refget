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

Reference sequences are fundamental to genomic analysis. To make analysis of reference sequences reproducible and efficient, we require tools that can identify, store, retrieve, and compare reference sequences. One such tool is the recent GA4GH standard [refget](http://samtools.github.io/hts-specs/refget.html), which provides a way to 1. compute deterministic sequence identifiers from sequences themselves, and 2) retrieve individual sequences using these identifiers. Refget thus provides a way to identify and retrieve sequences; however, many applications require working with *collections* of sequences. The *Sequence Collections* (seqcol) protocol is a pending GA4GH standard that works with collections of sequences. The seqcol protocol can be used to create unique identifiers derived from multiple sequences. Seqcol uses refget identifiers for individual sequences, adding new functionality to handle the complexity of collections, as well as attributes of sequences, such as names, lengths, or topologies. 

The goal of the Seqcol project is **to standardize identifiers for collections of sequences derived from the collections themselves**. It can be used to identify genomes, transcriptomes, or proteomes -- anything that can be represented as a collection of sequences. Seqcol uses a hash algorithm to generate a digest of the underlying collection of sequences. These unique identifiers are defined by an algorithm, rather than an accession authority, and are thus de-centralized and therefore usable for many purposes, including private or new sequence collections, cases without connection to a central database, or validation of sequence collection content and provenance.

Seqcol also specifies a RESTful API to enable retrieving the sequence collections given an identifier, which can be used for reproducibility and efficiency, as results can use the identifier to specify the exact reference genome used for an analysis. Reproducing the analysis could look up the original genome used. Furthermore, seqcol also provides a standardized method of comparing the contents of two sequence collections. This comparison function can be used to determine if analysis results that used different references genomes may still be compatible. In summary, the project specifies 3 procedures:

1. an algorithm for encoding sequence identifiers from collections
2. a lookup API to retrieve a collection given an identifier
3. a comparison API to assess compatibility of two collections



## Definitions of key terms

- **Sequence**: Seqcol uses refget to store actual sequences, so we generally use the term in the same way as refget. Refget was designed for nucleotide sequences; however, other sequences could be provided via the same mechanism, *e.g.*, cDNA, CDS, mRNA or proteins. Essentially any ordered list of refget-valid characters qualifies. However, sequence collections may also contain sequences of non-specified characters, which therefore have a length but no actual sequence content. 
- **Sequence collection**: A representation of 1 or more sequences that is structured according to the sequence collection schema
- **Digest**: An identifier resulting from a cryptographic hash function, such as `MD5` or `SHA512`, on input data.
- **Seqcol digest**: A digest for a sequence collection, computed according to the seqcol algorithm.
- **Seqcol algorithm**: The set of instructions used to compute a digest from a sequence collection.
- **Sequence digest** or **refget digest**: A digest for a sequence, computed according to the refget protocol.
- **Length**: The number of characters in a sequence.
- **Array**: An ordered list of elements
- **Alias**: A human-readable identifier used to refer to a sequence collection.
- **Seqcol API**: The set of endpoints defined in the *retrieval* and *comparison* components of the seqcol protocol.
- **Seqcol protocol**: Collectively, the 3 operations outlined in this document, which include: 1. encoding of sequence collections; 2. retrieval of sequence collections; and 3. comparison of sequence collections.
- **Coordinate system**: A set of named sequence lengths, but without actual sequences.


## Seqcol protocol functionality

The seqcol protocol defines two functions:

1. *Encoding* - An algorithm for computing an identifier given a collection of sequences;
2. *API* - A server RESTful API specification for retrieving and comparing sequence collections.

To be fully compliant with the seqcol protocol an implementation must provide all `REQUIRED` capabilities as detailed below. 

The seqcol algorithm is based on the refget algorithm for individual sequences, and should use refget servers to store the actual sequence data. Seqcol servers therefore provide a lightweight organizational layer on top of refget servers.

### 1. Encoding: Computing sequence digests from sequence collections

The encoding function specifies an algorithm that takes as input a set of annotated sequences and produces a unique digest. This function is generally expected to be provided by local software that operates on a local set of sequences.


#### Standardized sequence collection object representation

We first create an object representation of the attributes of the sequence collection. The structure of this object is critical, and is strictly controlled by the seqcol protocol. It must be defined in a JSON-schema. This is the general, minimal schema:


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
    items:
      type: string
      description: "Digests of sequences computed using the GA4GH digest algorithm (sha512t24u)."
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
    248956422,
    133797422,
    135086622
  ],
  "names": [
    "chr1",
    "chr2",
    "chr3"
  ],
  "sequences": [
    "2648ae1bacce4ec4b6cf337dcae37816",
    "907112d17fcb73bcab1ed1c72b97ce68",
    "1511375dc2dd1b633af8cf439ae90cec"
  ]
}
```

The object is a series of arrays with matching length (`3`), with the corresponding entries collated such that the first element of each array corresponds to the first element of each other array. For rationale for this structure over an array of annotated sequences, see *Footnote F1*.

`REQUIRED`: Implementations `MUST` at least provide the structure specified in this schema. Implementations `MAY` choose to extend this schema by adding additional attributes.


#### Seqcol algorithm:

Once the content of the sequence collection has been organized in the *seqcol object representation*, we can then run the digest algorithm to compute the unique identifier for this sequence collection

1. **Step 1**. Put all attributes of the sequence collection (*e.g.* names, lengths, sequence digests) in seqcol object representation form, as defined above.
2. **Step 2**. Apply [RFC-8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785) (JCS) to canonicalize the JSON of the seqcol object.
3. **Step 3**. Digest the canonical representation of each attribute's value using the seqcol digest algorithm (defined below). This should convert the value of each attribute in the seqcol into a digest string.
4. **Step 4**. Apply [RFC-8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785) again to canonicalize the JSON of new seqcol object representation.
5. **Step 5**. Digest the final canonical representation again. This is the *seqcol digest*, the final unique identifier for this sequence collection.

#### Digest algorithm

The digest algorithm is `TRUNC-512`. Details to follow.


---

### 2. API

The API has 3 top-level endpoints, for 3 functions: 1) `/service-info`, for describing information about the service; 2) `/collection`, for retrieving sequence collections; and 3) `/comparison` for comparing two sequence collections. Under these umbrella endpoints are a few more specific sub-endpoints, described in detail below:

#### `GET /service-info`

The service info endpoint provides information about the service

##### Return value

Must include

####  `GET /collection/:digest?level=:level` 

`REQUIRED`

The retrieval function specifies an API endpoint that retrieves original sequences from a database keyed by the unique digest. Here `:digest` is the seqcol digest computed above. This returns the sequence collection identified by the `:digest` variable. The form of the return `MUST` match the seqcol object representation form defined above. The `:level` query parameter provides a way for the user to specify the structure of the return value. The level corresponds to the "expansion level" of the returned sequence collection returned. The default is `?level=2`, which returns the canonical structure. 

- `?level=1` 


#### `GET /comparison/{digest1}/{digest2}` 
`REQUIRED`

The comparison function specifies an API endpoint that allows a user to compare two sequence collections. The output is an assessment of compatibility between those sequence collections. 

 
#### `POST /comparison/{digest1}` 

`REQUIRED`

Compares one database collection to a local user-provided collection.

Return value:


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

## Footnotes

### F1. Why use an array-oriented structure instead of a sequence-oriented structure?

In the final structure, we first organize the sequence collection into what we called an "array-oriented" data structure, which is a set of collated arrays (names, lengths, sequences, *etc.*). An alternative we considered first was a "sequence-oriented" structure, with `{name, length, sequence}` first grouped as a unit, and then a collection structured as an array of such units. There are 3 reasons we settled on the array-based nature. 1) Flexibility of sequence attributes. This makes it straightforward to mix-and-match components of the collection. Because each component is independent, and not integrated in with the sequence, it is simpler to select and build subsets and permutations. 2) Backwards compatibility. Along the same lines, what happens for an implementation that adds a new attribute? For example, if an implementation adds a `topology` attribute to the sequences, in the sequence-oriented structure, this would alter the sequence object and thereby change its digest. In the array-based structure, since we digest the arrays individually, the array digest is not changed. 3) Conciseness. Sequence collections may be used for tens of thousands or even millions of sequences. For example, a transcriptome may containe a million possible transcripts. The array-oriented data structure is a more concise representation for digesting collections with many elements becuase the attributes are only specified once instead of once per element. Furthermore, the level 1 representation of the sequence collection is more concise for large collections, since we only need one digest per attribute, rather than one digest per sequence. 4) Utility of intermediate digests. The array-oriented approach provides useful intermediate digests for each attribute. This digest can be used to test for matching sets of sequences, or matching coordinate systems, using the individual component digests. With a sequence-oriented framework, this would require traversing down a layer deeper, to the individual elements, to establish identity of individual components.

