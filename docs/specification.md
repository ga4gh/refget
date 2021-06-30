---
title: Seqcol specification version 0.0.1
---

<!-- Table of contents: 
* The generated Toc will be an unordered list
{:toc} -->

<h1>Seqcol specification version 0.0.1</h1>

Table of contents:

[TOC]

NOTICE! THIS IS A DRAFT SPECIFICATION FOR FEEDBACK. NOTHING IN HERE IS OFFICIAL OR FINAL.

## Introduction

Reference sequences are fundamental to genomic analysis. The [refget protocol](http://samtools.github.io/hts-specs/refget.html) is a GA4GH standard that provides a way to access individual sequences using unique identifiers derived from sequences themselves. However, many applications require us to operate on *collections* of sequences. The *Sequence Collections*  (seqcol) protocol extends the refget protocol to collections of sequences. The seqcol protocol can be used to create unique identifiers derived from sequences collections themselves. Seqcol identifiers use refget identifiers under the hood.

Seqcol uses a hash algorithm (either MD5 of TRUNC512) to generate a digest of the underlying collection of sequences. These unique identifiers are defined by an algorithm, rather than an accession authority, and are thus de-centralized and therefore usable for many purposes, including private or new sequence collections, cases without connection to a central database, or validation of sequence collection content and provenance.

Seqcol also specifies a RESTful API to enable retrieving the sequence collections given a unique identifier.

## Definitions of terms

The key terms are:

- **Sequence**: Seqcol uses refget to store actual sequences, so we use the term in the same way as refget. Refget was designed for nucleotide sequences; however, other sequences could be provided via the same mechanism, *e.g.*, cDNA, CDS, mRNA or proteins. Essentially any ordered list of valid characters qualifies.
- **Sequence collection**: An ordered list of sequences.
- **Digest**: A unique identifier resulting from a cryptographic hash function, such as `MD5` or `SHA512`, on input data.
- **Seqcol digest**: A digest for a sequence collection, computed according to the seqcol algorithm.
- **Seqcol algorithm**: The set of instructions used to compute a digest from a sequence collection.
- **Sequence digest** or **refget digest**: A digest for a sequence, computed according to the refget protocol.
- **Annotated sequence digest**: A digest of a sequence plus its metadata, such as name and length. This should be read as `{{annotated sequence} digest}`, not `{{annotated {sequence digest}}`.
- **Metadata**: Any extra data attached to a sequence collection, including a human-readable alias, source or provider of the collection, version information, etc.
- **Length**: The number of characters in a sequence.
- **Alias**: A human-readable identifier used to refer to a sequence collection.
- **Seqcol API**: The set of endpoints defined in the *retrieval* component of the seqcol protocol.
- **Seqcol protocol**: Collectively, the 3 operations outlined in this document, which include: 1. encoding of sequence collections; 2. retrieval of sequence collections; and 3. comparison of sequence collections.
- **Coordinate system**: A set of sequence lengths, but without actual sequences.


## Design principles

- The seqcol protocol provides 3 primary procedures: the *algorithm* for encoding; the *API* for retrieval; and the *flag framework* for comparison.
- The seqcol algorithm is based on the refget algorithm for individual sequences, and should use refget servers to store the actual sequence data. Seqcol servers therefore provide a lightweight organizational layer on top of refget servers.

## Seqcol protocol functionality

The core functionality of the seqcol protocol is to provide three operations:

1. *Encoding* - An algorithm for computing a unique identifier given a collection of sequences; and
2. *Retrieval* - A RESTful API specification for retrieving a collection of sequences from a database given a unique identifier.
3. *Comparison* - A flag-based framework to compare sequence collections.

Implementations of the seqcol protocol may implement only one of these functions, but to be compliant, must implement all `REQUIRED` capabilities of that function, as detailed below.

### 1. Encoding: Computing sequence digests from sequence collections

The encoding function is fulfilled by an algorithm that takes as input a set of annotated sequences and produces a unique digest. This function should be provided by local software that operates on a local set of sequences.

#### Algorithm:


**Step 1**. Repeat for each sequence in the collection:  

- 1a. Compute the refget digest for the sequence.
- 1b. Compute the length of the sequence.
- 1c. Concatenate the name, length, and refget digest, in the order provided in the collection, delimited by `ATTRIBUTE_DELIMITER`.
- 1d. Compute the digest of the result of step 1c. This is the *annotated sequence digest* for the sequence.

**Step 2**. Concatenate the computed annotated sequence digests, in the order provided in the collection, delimited by `ITEM_DELIMITER`.

**Step 3**. Compute the digest of the result of step 2. This is the *seqcol digest*, the final unique identifier for this sequence collection.

In step 1, there are a few constraints:

1. Name may only contain certain characters.
2. Name may be empty, and refget digest may be empty, but length is required.

#### Delimiters and hash functions

- `ATTRIBUTE_DELIMITER` is defined as: `undecided`
- `ITEM_DELIMITER` is defined as: `undecided`
- `hash function` is allowed to be either `MD5` OR `TRUNC-512`. Details to follow.

### 2. Retrieval: Retrieving sequence collections from digests

The retrieval function is fulfilled by an API specification that provides an ability to retrieve original sequences from a database keyed by the unique digest.

#### Core endpoints

Core endpoints are `REQUIRED` for an implementation to be compliant with the seqcol protocol.

- `GET /seqcol/:digest/:reclimit` -- Here `:digest` is the seqcol digest computed above. `:reclimit` is the recursion limit; it may be left off for no recursion limit. This endpoint returns the sequence collection identified by the `:digest` variable, and recursively retrieves the values for any digests within the returned sequence, until the `:reclimit` value has been reached.

#### Secondary endpoints

Secondary endpoints are `OPTIONAL` for an implementation to be compliant with the seqcol protocol.

- `GET /metadata/:digest` -- Returns all available metdata for the specified digest.
- `GET /containing_collections/:refget_digest` -- Returns a list of digests of any collections containing the given sequence digest.

### 3. Comparison: Determining compatibility between sequence collections

Input is 2 sequence collections, and output is an assessment of compatibility between those sequence collections. The comparison function can be used to make the following comparisons:

- Strict identity. 
- Order-relaxed identity.
- Name-relaxed identity. 
- Length-only compatible. 

More details to be added here later.


## Operations enabled by seqcol, organized by input

### Seqcol digest as input

* **seqcol digest -> sequence digests**: Given a seqcol digest, sequence digests for all contained sequences can be retrieved by the `/seqcol/:digest/:reclimit` endpoint by setting the `reclimit` to 1.
* **seqcol digest -> sequences**: Given a seqcol digest, sequences themselves for all contained sequences can be retrieved by the `/seqcol/:digest/:reclimit` endpoint by setting the `reclimit` to 2, or omitting it.
* **seqcol digest -> metadata**: Retrieved by using the `metadata` endpoint.
* **seqcol digest -> aliases of seqcol**: Does the API offer this? Should be consistent with refget.
* **seqcol digest -> metadata of seqcol**: Provided by the `metadata` endpoint.
* **2 seqcol digests -> assessment of compatibility**: Provided by the compatibility function of the seqcol protocol. First, retrieve the annotated sequence digests for each sequence collection using the *Retrieval* function, then use the *Comparison* function to assess compatibility.
* **seqcol digest + sequences -> validation claim**: Use the *seqcol algorithm* to compute the digest of the set of sequences, and then use the *comarison* function to validate (or, if you validation requires strict identity, just confirm that the digests match).


### Sequence collection as input

* **sequences -> refget digests**: Individual sequences can be converted into refget digests using the refget algorithm
* **sequences -> seqcol digest**: Sequence collections can be converted into seqcol digests using the seqcol algorithm. To obtain a sequence digest, follow the algorithm outlined above, sequences -> refget digests -> seqcol digest.

### Refget digest as input

* **refget digest -> seqcol digests containing this sequence**: Use the `containing_collections` secondary endpoint. 
* **many refget digests -> seqcol digests containing all of these sequences**: Repeated application of `containing_collections` endpoint?

## Coordinate system as input

* **coordinate system -> seqcol digest**: If you have only a coordinate system (lengths with no sequences), you may still compute a seqcol digest using the seqcol algorithm, because the sequences themselves are optional. Coordinate systems can therefore be uniquely encoded, retrieved, and compared using the seqcol protocol.


## Related operations: What seqcol does *not* do

* **sequences -> sequence names**: This is the refget reverse lookup.
* **alias -> seqcol digest**: Will the API offer this?
* **seqcol digest -> other seqcol digests compatible at some level**: The database does not store comparison flags, so if you're searching for compatible collections you'd need to retrieve and compute the compatible yourself using the comparison function.

