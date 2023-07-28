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

Reference sequences are fundamental to genomic analysis. To make their analysis reproducible and efficient, we require tools that can identify, store, retrieve, and compare reference sequences. The primary goal of the *Sequence Collections* (seqcol) project is **to standardize identifiers for collections of sequences**. Seqcol can be used to identify genomes, transcriptomes, or proteomes -- anything that can be represented as a collection of sequences. In brief, the project specifies 3 procedures:

1. **An algorithm for encoding sequence identifiers.**  The GA4GH standard [refget](http://samtools.github.io/hts-specs/refget.html) specifies a way to compute deterministic sequence identifiers from individual sequences. Seqcol uses refget identifiers and adds functionality to wrap them into collections of sequences. Secol also handles sequence attributes, such as their names, lengths, or topologies. Seqcol identifiers are defined by a hash algorithm, rather than an accession authority, and are thus de-centralized and usable for many purposes, including private or new sequence collections, cases without connection to a central database, or validation of sequence collection content and provenance.
2. **A lookup API to retrieve a collection given an identifier.** Seqcol specifies a RESTful API to enable retrieving the sequence collections given an identifier. This allows one to retrieve the exact reference genome used for an analysis, instead of guessing based on a human-readable identifier. 
3. **A comparison API to assess compatibility of two collections.** Finally, seqcol also provides a standardized method of comparing the contents of two sequence collections. This comparison function can be used to determine if analysis results that used different references genomes may still be compatible. 


## Use cases

1. Given a collection identifier, retrieve the underlying sequence identifiers.
2. Given a collection identifier, retrieve the underlying sequences.
3. Given two collection identifiers, determine if downstream results are compatible.
4. Given a collection identifier, retrieve metadata about the collection. This may include human-readable aliases, author of the collection, links to other collections, or other metadata.
5. Given a sequence collection, compute its identifier.



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

The seqcol algorithm is based on the refget algorithm for individual sequences, and should use refget servers to store the actual sequence data. Seqcol servers therefore provide a lightweight organizational layer on top of refget servers.  o be fully compliant with the seqcol protocol an implementation must provide all `REQUIRED` capabilities as detailed below.  The seqcol protocol defines two functions:

1. *Encoding* - An algorithm for computing an identifier given a collection of sequences.
2. *API* - A server RESTful API specification for retrieving and comparing sequence collections.
3. *Ancillary attribute management* - An optional specification for organizing non-inherent metadata as part of a sequence collection.

### 1. Encoding: Computing sequence digests from sequence collections

The encoding function specifies an algorithm that takes as input a set of annotated sequences and produces a unique digest. This function is generally expected to be provided by local software that operates on a local set of sequences. These steps of the encoding process are:

- **Step 1**. Organize the sequence collection data into *canonical seqcol object representation*.
- **Step 2**. Apply [RFC-8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785) (JCS) to canonicalize the value associated with each attribute individually.
- **Step 3**. Digest each canonicalized attribute value using the GA4GH digest algorithm.
- **Step 4**. Apply [RFC-8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785) again to canonicalize the JSON of new seqcol object representation.
- **Step 5**. Digest the final canonical representation again.

Example Python code for computing a seqcol digest can be found in *Footnote F3*. These steps are described in more detail below:

#### Step 1: Organize the sequence collection data into *canonical seqcol object representation*.

We first create an object representation of the attributes of the sequence collection. The structure of this object is critical, and is strictly controlled by the seqcol protocol. It must be defined in a JSON-schema. This is the general, minimal schema:


```YAML
description: "A collection of biological sequences."
type: object
properties:
  lengths:
    type: array
    collated: true
    description: "Number of elements, such as nucleotides or amino acids, in each sequence."
    items:
      type: integer
  names:
    type: array
    collated: true
    description: "Human-readable identifiers of each sequence (chromosome names)."
    items:
      type: string
  sequences:
    type: array
    collated: true
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

This schema is the *seqcol schema*, and sequence collection objects in this structure are said to be the *canonical seqcol object representation*. Here's an example of a sequence collection organized into the canonical seqcol object representation:

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

This object would validate against the JSON-schema above. The object is a series of arrays with matching length (`3`), with the corresponding entries collated such that the first element of each array corresponds to the first element of each other array. For rationale for this structure over an array of annotated sequences, see *Footnote F1*. Implementations `MUST` provide at least the structure specified in this schema. Implementations `MAY` choose to extend this schema by adding additional attributes.

##### Step 1a: Filter non-inherent attributes

The `inherent` section in the seqcol schema is an extension of the basic JSON-schema format that adds specific functionality. Inherent attributes are those that contribute to the identifier; *non-inherent* attributes are not considered in computing the top-level digest. Attributes of a seqcol that are *not* listed as `inherent` `MUST NOT` contribute to the identifier; they are therefore excluded from the digest calculation. Therefore, if the canonical seqcol representation includes any non-inherent attributes, these must be removed before proceeding to step 2. In the simple example, there are no non-inherent attributes. For further details about the rationale and examples of non-inherent attributes, see *Footnote F2*.

#### Step 2: Apply RFC-8785 to canonicalize the value associated with each attribute individually.

The [RFC-8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785) (JCS) standardizes whitespace, character encodings, and other details that would cause inconsequential variations to yield different digests. For most use cases, the following Python fuction suffices:

```python
def canonical_str(item: dict) -> str:
    """Convert a dict into a canonical string representation"""
    return json.dumps(
        item, separators=(",", ":"), ensure_ascii=False, allow_nan=False, sort_keys=True
    )
```

This will turn the values into canonicalized string representations of the list objects, so lengths becomes the string `[248956422,133797422,135086622]`, names becomes the string `["chr1","chr2","chr3"]`, and sequences becomes the string 

```
["2648ae1bacce4ec4b6cf337dcae37816","907112d17fcb73bcab1ed1c72b97ce68","1511375dc2dd1b633af8cf439ae90cec"]
```

#### Step 3: Digest each canonicalized attribute value using the GA4GH digest algorithm.

The GA4GH digest algorithm, `sha512t24u`, was created as part of the [Variation Representation Specification standard](https://vrs.ga4gh.org/en/stable/impl-guide/computed_identifiers.html).  This procedure is described as ([Hart _et al_. 2020](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0239883)):

- performing a SHA-512 digest on a binary blob of data
- truncate the resulting digest to 24 bytes
- encodes the 24 bytes using `base64url` ([RFC 4648](https://datatracker.ietf.org/doc/html/rfc4648#section-5)) resulting in a 32 character string

This converts the value of each attribute in the seqcol into a digest string. Applying this to each value will produce a structure that looks like this:

```json
{
  "lengths": "20e95aade8e72d399dbf7f82a9e84ba5cc4047dc8d791d62",
  "names": "834e2529dc6262d1b774e19e502e4074a1227f0eb91b45a9",
  "sequences": "78f45f5aa3b36a2a8fe1eec415258a036b3753f69acf05df"
}
```

#### Step 4: Apply RFC-8785 again to canonicalize the JSON of new seqcol object representation.

Here, we repeat step 2, except instead of applying to each value separately, we apply to the entire object. This will result in a canonical string representation of the object, which is this string:

```
{"lengths":"20e95aade8e72d399dbf7f82a9e84ba5cc4047dc8d791d62","names":"834e2529dc6262d1b774e19e502e4074a1227f0eb91b45a9","sequences":"78f45f5aa3b36a2a8fe1eec415258a036b3753f69acf05df"}
```

#### Step 5: Digest the final canonical representation again.

Again using the same approach as in step 3, we now apply the GA4GH digest algorithm to this string. The result is the final unique identifier for this sequence collection:

```
64ff00b85402a4dc821752e1e8d56d3ecc4e29b55a930748
```

---

### 2. API: A server RESTful API specification for retrieving and comparing sequence collections.

The API has 3 top-level endpoints, for 3 functions:

1. `/service-info`, for describing information about the service;
2. `/collection`, for retrieving sequence collections; and
3. `/comparison`, for comparing two sequence collections.

Under these umbrella endpoints are a few more specific sub-endpoints, described in detail below:

#### 2.1 Service info 
- *Endpoint*: `GET /service-info` (`REQUIRED`)
- *Description*: The service info endpoint provides information about the service
- *Return value*: Must include the Seqcol schema that this server uses.

#### 2.2 Collection

- *Endpoint*: `GET /collection/:digest?level=:level` (`REQUIRED`)
- *Description*: The retrieval function specifies an API endpoint that retrieves original sequences from a database keyed by the unique digest. Here `:digest` is the seqcol digest computed above.  The level corresponds to the "expansion level" of the returned sequence collection returned. The default is `?level=2`, which returns the canonical structure.
- *Return value*: The sequence collection identified by the `:digest` variable. The structure of the data `MUST` be modulated by the `:level` query parameter.  Specifying `?level=2` returns the canonical structure, and `?level=1` returns the collection with digested attributes.

Non-inherent attributes `MUST` be stored and returned by the collection endpoint alongside inherent attributes.

#### 2.3 Comparison

- *Endpoint variant 1*: Two-digest comparison`GET /comparison/{digest1}/{digest2}` (`REQUIRED`)  
- *Endpoint variant 2*: POST comparison with one digest  `POST /comparison/{digest1}` (`REQUIRED`)  
- *Description*: The comparison function specifies an API endpoint that allows a user to compare two sequence collections. The `POST` version Ccompares one database collection to a local user-provided collection.  
- *Return value*: The output is an assessment of compatibility between those sequence collections. Both variants of the `/comparison` endpoint must `MUST` return an object in JSON format with these 3 keys: "digests", "arrays", and "elements", as described below:
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


### 3. Ancillary attribute management: recommended non-inherent attributes

In *Section 1: Encoding*, we distinguished between *inherent* and *non-inherent* attributes. Here, we specify standardized, useful non-inherent attributes.

#### 3.2 The `sorted_name_length_pairs` attribute (`RECOMMENDED`)

The `sorted_name_length_pairs` attribute is a *non-inherent* attribute of a sequence collection with a formal definition, provided here. It is `RECOMMENDED` that all seqcol implementations add this attribute to all sequence collections. When digested, this attribute provides an identifier for an order-invariant coordinate system for a sequence collection. Because it is *non-inherent*, it does not affect the identity (digest) of the collection. It is created deterministically from the `names` and `lengths` attributes in the collection; it *does not* depend on the actual sequence content, so it is consistent across two collections with different sequence content if they have the same `names` and `lengths`, in the same order.

Algorithm: 

1. Lump together each name-length pair from the primary collated `names` and `lengths` into an object, like `{"length":123,"name":"chr1"}`.
2. Canonicalize JSON according to the seqcol spec (using RFC-8785).
3. Digest each name-length pair string individually.
4. Sort the digests lexographically.
5. Add as a non-inherent, non-collated attribute to the sequence collection object.

## Footnotes

### F1. Why use an array-oriented structure instead of a sequence-oriented structure?

In the canonical seqcol object structure, we first organize the sequence collection into what we called an "array-oriented" data structure, which is a list of collated arrays (names, lengths, sequences, *etc.*). An alternative we considered was a "sequence-oriented" structure, which would group each sequence with some attributes, like `{name, length, sequence}`, and structure the collection as an array of such units. While this is intuitive, as it captures each sequence object with some accompanying attributes as independent entities, there are several reasons we settled on the array-oriented structure instead: 

  1. Flexibility and backwards compatibility of sequence attributes. What happens for an implementation that adds a new attribute? For example, if an implementation adds a `topology` attribute to the sequences, in the sequence-oriented structure, this would alter the sequence object and thereby change its digest. In the array-based structure, since we digest the arrays individually, the array digest is not changed. Thus, the array-oriented structure emphasizes flexibility of attributes, where the sequence-oriented structure would emphasize flexibility of sequences. In other words, the array-based structure makes it straightforward to mix-and-match *attributes* of the collection. Because each attribute is independent, and not integrated into individual sequence objects, it is simpler to select and build subsets and permutations of attributes. We reasoned that flexibility of attributes was desirable.

   2. Conciseness. Sequence collections may be used for tens of thousands or even millions of sequences. For example, a transcriptome may contain a million transcripts. The array-oriented data structure is a more concise representation for digesting collections with many elements because the attribute names are specified once *per collection* instead of once *per element*. Furthermore, the level 1 representation of the sequence collection is more concise for large collections, since we only need one digest *per attribute*, rather than one digest *per sequence*. 

   3. Utility of intermediate digests. The array-oriented approach provides useful intermediate digests for each attribute. This digest can be used to test for matching sets of sequences, or matching coordinate systems, using the individual component digests. With a sequence-oriented framework, this would require traversing down a layer deeper, to the individual elements, to establish identity of individual components. The alternative advantage we would have from a sequence-oriented structure would be identifiers for *annotated sequences*. We gain the advantages of these digests through the *names-lengths* attribute.

### F2. Details of inherent and non-inherent attributes

The specification in section 1, *Encoding*, described how to structure a sequence collection and then apply an algorithm to compute a digest for it. What if you have ancillary information that goes with a collection, but shouldn't contribute to the digest? We have found a lot of useful use cases for information that should go along with a seqcol, but should not contribute to the *identity* of that seqcol. This is a useful construct as it allows us to include information in a collection that does not affect the identifier that is computed for that collection. One simple example is the "author" or "uploader" of a reference sequence; this is useful information to store alongside this collection, but we wouldn't want the same collection with two different authors to have a different identifier! Seqcol refers to these as *non-inherent attributes*, meaning they are not part of the core identity of the sequence collection. Non-inherent attributes are defined in the seqcol schema, but excluded from the `inherent` list. 
