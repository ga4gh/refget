---
title: Refget Sequence Collections v1.0.0
---

# Refget Sequence Collections v1.0.0

## Introduction

Reference sequences are fundamental to genomic analysis.
To make their analysis reproducible and efficient, we require tools that can identify, store, retrieve, and compare reference sequences.
The primary goal of the *Refget Sequence Collections* (seqcol) project is **to standardize identifiers for collections of sequences**.
Seqcol can be used to identify genomes, transcriptomes, or proteomes -- anything that can be represented as a collection of sequences.
A common example and primary use case of sequence collections is for a reference genome, so this documentation sometimes refers to reference genomes for convenience; really, it can be applied to any collection of sequences.

In brief, the project specifies several procedures:

1. **An algorithm for encoding sequence collection identifiers.**
Refget Sequence Collections extends [Refget Sequences](sequences.md) to collections of sequences.
Seqcol also handles sequence attributes, such as their names, lengths, or topologies.
Like Refget sequences, seqcol digests are defined by a hash algorithm, rather than an accession authority.
2. **An API describing lookup and comparison of sequence collections.**
Seqcol specifies an http API to retrieve a sequence collection given its digest.
This can be used to reproduce the exact sequence collection instead of guessing based on a human-readable identifier.
Seqcol also provides a standardized method of comparing the contents of two sequence collections.
3. **Recommended ancillary attributes.**
Finally, the protocol defines several recommended procedures that will improve compatibility across Seqcol servers, and beyond.

## Use cases

Sequence collections represent fundamental concepts, making the specification adaptable to a wide range of use cases.
A primary goal is to enable sequence collection (seqcol) digests to replace or complement the human-readable identifiers currently used for reference genomes (e.g., "hg38" or "GRCh38").
Unfortunately, these simple identifiers often refer to references with subtle (or not so subtle) differences. Such variation leads to fundamental issues in analyses relying on reference genomes, undermining the utility of these identifiers.  

Unique identifiers, such as those provided by the NCBI Assembly database, partially address this problem by unambiguously identifying specific assemblies. However, this approach has limitations:

- It depends on a central authority, which excludes custom genomes and doesn't cover all reference providers.
- Centralized identifiers alone cannot *confirm* identity, as identity also depends on the genome's content.  
- It does not address the related challenge of determining compatibility among reference genomes. Analytical results or annotations based on different references may still be integrable if certain conditions are met, but current tools and standards lack the means to formalize and simplify compatibility comparisons.  

The [refget sequences protocol](sequences.md) provides a partial solution applicable to individual sequences, such as a single chromosome.
However, refget does not directly address collections of sequences, such as a linear reference genome.
Building on refget, the sequence collections specification introduces foundational concepts that support diverse use cases, including:  

- **Accessing sequences**:  *As a data analyst, I want to know which sequences are in a specific collection so I can analyze them further.*  
- **Comparing collections**:  *As a data analyst, I want to compare the sequence collections used in two separate analyses to assess the compatibility of their resulting data.*  
- **Annotation curation**: *As a data curator for SNP data, I want an unambiguous reference genome identifier upon which my SNP annotations can be interpreted, so I can compare them with confidence*.
- **Extracting subsets**:  *As a data analyst, I want to extract specific sequences, such as those composing the chromosomes or karyotype of a genome.*  
- **Validating submissions**: *As a submission system, I need to determine the exact content of a sequence collection to validate data file submissions.*  
- **Embedding identifiers**:  *As a software developer, I want to embed a sequence collection identifier in my tool's output, allowing downstream tools to identify the exact sequence collection used.*  
- **Checking compatibility**:  *As a data analyst using published data, I have a chromosome sizes file (a set of lengths and names) and want to determine whether a given sequence collection is length- or name-compatible with this file.*  
- **Genome browser integration**:  *As a genome browser, I use one sequence collection for the displayed coordinate system and want to check if a digest representing a given BED file's coordinate system is compatible with it.*  
- **Annotating unknown references**:  *As a data processor, I encounter input data without reference genome information and want to generate a sequence collection digest to attach, enabling further processing with seqcol features.*  

## Architectural decision record

For a chronological record of decisions related to this specification, see the [Architectural decision record](decision_record.md).

## Definitions of key terms

### General terms

- **Alias**: A human-readable identifier used to refer to a sequence collection.
- **Array**: An ordered list of elements.
- **Coordinate system**: An ordered list of named sequence lengths, but without actual sequences.
- **Digest**: A string resulting from a cryptographic hash function, such as `MD5` or `SHA512`, on input data.
- **Length**: The number of characters in a sequence.
- **Level**: A way of specifying the completeness of a sequence collection representation. Level 0 is the simplest representation, level 1 more complete, level 2 even more complete, and so forth. Representation levels are described in detail under [terminology](#terminology).
- **Qualifier**: A reserved term used in the schema to indicate a quality of an attribute, such as whether it is required, collated, or inherent. Qualifiers are listed below.
- **Seqcol algorithm**: The set of instructions used to compute a digest from a sequence collection.
- **Seqcol API**: The set of endpoints defined in the *retrieval* and *comparison* components of the seqcol protocol.
- **Seqcol digest**: A digest for a sequence collection, computed according to the seqcol algorithm.
- **Seqcol protocol**: Collectively, the operations outlined in this document, which include: 1. encoding of sequence collections; 2. API describing retrieval and comparison ; and 3. specifications for ancillary recommended attributes.
- **Sequence**: Seqcol uses refget sequences to identify actual sequences, so we generally use the term "sequence" in the same way. Refget sequences was designed for nucleotide sequences; however, other sequences could be provided via the same mechanism, *e.g.*, cDNA, CDS, mRNA or proteins. Essentially any ordered list of refget-sequences-valid characters qualifies.
- **Sequence digest** or **refget sequence digest**: A digest for a sequence, computed according to the refget sequence protocol.
- **Sequence collection**: A representation of 1 or more sequences that is structured according to the sequence collection schema.
- **Sequence collection attribute**: A property or feature of a sequence collection (*e.g.* names, lengths, sequences, or topologies).

### Attribute qualifiers

These qualifiers apply to a seqcol attribute. These definitions specify something about the attribute if the qualifier is true:

- **Collated**: the values of the attribute match 1-to-1 with the sequences in the collection and are represented in the same order.
- **Inherent**: the attribute is part of the definition of the sequence collection and therefore contributes to its digest.
- **Passthru**: the attribute is *not digested* in transition from level 2 to level 1. So its value at level 1 is the same as at level 2.
- **Transient**: the attribute *cannot be retrieved through the `/attribute` endpoint*.

## Seqcol protocol functionality

The seqcol algorithm is based on the refget sequence algorithm for individual sequences and should use refget sequence servers to store the actual sequence data.
Seqcol servers therefore provide a lightweight organizational layer on top of refget sequence servers.
To be fully compliant with the seqcol protocol an implementation must provide all `REQUIRED` capabilities as detailed below.

The seqcol protocol defines the following:

1. *Schema* - The way an implementation should define the attributes of sequence collections it holds.
2. *Encoding* - An algorithm for computing a digest given a sequence collection.
3. *API* - A server API specification for retrieving and comparing sequence collections.
4. *Ancillary attribute management* - A specification for organizing non-inherent metadata as part of a sequence collection.

### 1. Schema: Defining the attributes in the collection

The first step for a Sequence Collections implementation is to define the *list of contents*, that is, what attributes are allowed in a collection, and which of these affect the digest.
The sequence collections standard is flexible with respect to the schema used, so implementations of the standard can use the standard with different schemas, as required by a particular use case.
This divides the choice of content from the choice of algorithm, allowing the algorithm to be consistent even in situations where the content is not.
Nevertheless, we RECOMMEND that all implementations start from the same base schema, and add additional attributes as needed, which will not break interoperability.
We RECOMMEND *not changing the inherent attributes list*, because this will keep the identifiers compatible across implementations.
Implementations that use different inherent attributes are still compliant with the specification generally, but do so at the cost of top-level digest interoperability.
For more information about community-driven updates to the base schema, see [*Footnote F5*](#f5-adding-new-schema-attributes).

This is the RECOMMENDED minimal base schema:

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
    description: "Human-readable labels of each sequence (chromosome names)."
    items:
      type: string
  sequences:
    type: array
    collated: true
    items:
      type: string
      description: "Refget sequences v2 identifiers for sequences."
required:
  - names
  - lengths
  - sequences
ga4gh:
  inherent:
    - names
    - sequences
```

Sequence collection objects that follow the base minimal structure are said to be the *canonical seqcol object representation*.
The implementation `MUST` define its structure in a JSON Schema, such as this example.
Implementations `MAY` choose to extend this schema by adding additional attributes.
Implementations `MAY` also use a different schema, but we `RECOMMEND` the schema extend the base schema defined above.

This schema extends vanilla JSON Schema with a few Seqcol-specific *attribute qualifiers*: the `collated` and `inherent` qualifiers. 
The specification also defines other attribute qualifiers that are not used in the base schema. 
For further details about attribute qualifiers, see [*Section 4*](#4-extending-the-schema-schema-attribute-qualifiers).

### 2. Encoding: Computing sequence digests from sequence collections

The encoding function is the algorithm that produces a unique digest for a sequence collection.
The input to the function is a set of annotated sequences.
This function is generally expected to be provided by local software that operates on a local set of sequences.
Example Python code for computing a seqcol digest can be found in the [tutorial for computing seqcol digests](digest_from_collection.md).
For information about the possibilty of deviating from this procedure for custom attributes, see [*Footnote F6*](#f6-custom-encoding-algorithms).
The steps of the encoding process are described in detail below; briefly, the steps are:

- **Step 1**. Organize the sequence collection data into *canonical seqcol object representation* and filter the non-inherent attributes.
- **Step 2**. Apply [RFC-8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785) (JCS) to canonicalize the value associated with each attribute individually.
- **Step 3**. Digest each canonicalized attribute value using the GA4GH digest algorithm.
- **Step 4**. Apply [RFC-8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785) again to canonicalize the JSON of the new seqcol object representation.
- **Step 5**. Digest the final canonical representation again using the GA4GH digest algorithm.



#### Step 1: Organize the sequence collection data into *canonical seqcol object representation*.

We first create an object representation of the attributes of the sequence collection.
The structure of this object is critical, and is strictly controlled by the seqcol protocol.
The sequence collection object MUST first be structured according to the schema definition for the implementation.

Here's an example of a sequence collection organized into the canonical seqcol object representation following the minimal schema example above:

```json
{
  "lengths": [
    248956422,
    242193529,
    198295559
  ],
  "names": [
    "chr1",
    "chr2",
    "chr3"
  ],
  "sequences": [
    "SQ.2YnepKM7OkBoOrKmvHbGqguVfF9amCST",
    "SQ.lwDyBi432Py-7xnAISyQlnlhWDEaBPv2",
    "SQ.Eqk6_SvMMDCc6C-uEfickOUWTatLMDQZ"
  ]
}
```

This object would validate against the JSON Schema above.
The object is a series of arrays with matching length (`3`), with the corresponding entries collated such that the first element of each array corresponds to the first element of each other array.
For the rationale why this structure was chosen instead of an array of annotated sequences, see [*Footnote F1*](#f1-why-use-an-array-oriented-structure-instead-of-a-sequence-oriented-structure).

#### Step 2: Apply RFC-8785 to canonicalize the value associated with each attribute individually.

The [RFC-8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785) (JCS) standardizes whitespace, character encodings, and other details that would cause inconsequential variations to yield different digests.
For most use cases*, the following Python function suffices:

```python
import json

def canonical_str(item: [list, dict]) -> bytes:
    """Convert a list or dict into a canonicalized UTF8-encoded bytestring representation"""
    return json.dumps(
        item, separators=(",", ":"), ensure_ascii=False, allow_nan=False, sort_keys=True
    ).encode("utf8")
```

This will turn the values into canonicalized UTF8-encoded bytestring representations of the list objects. Using Python notation, the value of the lengths attribute becomes `b'[248956422,242193529,198295559]'`, the value of the names attribute becomes `b'["chr1","chr2","chr3"]'`, and the value of the sequences attribute becomes 

```
b'["SQ.2YnepKM7OkBoOrKmvHbGqguVfF9amCST","SQ.lwDyBi432Py-7xnAISyQlnlhWDEaBPv2","SQ.Eqk6_SvMMDCc6C-uEfickOUWTatLMDQZ"]'
```

_* The above Python function suffices if (1) attribute keys are restricted to ASCII, (2) there are no floating point values, and (3) for all integer values `i`:  `-2**63 < i < 2**63`_

In this process, RFC-8785 is applied only to objects; we assume the sequence digests are computed through an external process (the refget sequences protocol), and are not computed as part of the sequence collection.
The refget sequences protocol digests sequence strings without JSON-canonicalization.
For more details, see [*Footnote F2*](#f2-rfc-8785-does-not-apply-to-refget-sequences).


!!! warning "Exception for passthru attributes"

    This per-attribute digesting procedure (Steps 2 and 3) is applied by default to all attributes of a sequence collection, *except for attributes qualified in the schema as passthru*; these attributes are NOT digested in this way, but "passed through" unchanged to the JSON structure described in Step 3.
    For more information about passthru attributes, see [Section 4](#4-extending-the-schema-schema-attribute-qualifiers).

    
#### Step 3: Digest each canonicalized attribute value using the GA4GH digest algorithm.

Apply the GA4GH digest algorithm to each attribute value.
The GA4GH digest algorithm is described in detail in [*Footnote F3*](#f3-the-ga4gh-digest-algorithm).
This converts the value of each attribute in the seqcol into a digest string.
Applying this to each value will produce the following structure:

```json
{
  "lengths": "5K4odB173rjao1Cnbk5BnvLt9V7aPAa2",
  "names": "g04lKdxiYtG3dOGeUC5AdKEifw65G0Wp",
  "sequences": "rD29ZKmEqwwHRXjiQ36p6UMZQ5hemmsb"
}
```

!!! warning "Filter non-inherent attributes"

    The `inherent` section in the seqcol schema is an extension of the basic JSON Schema format that adds specific functionality.
    Inherent attributes are those that contribute to the top-level digest; *non-inherent* attributes are not considered when computing the top-level digest.
    Attributes of a seqcol that are *not* listed as `inherent` `MUST NOT` contribute to the digest; they are therefore excluded from the top-level digest calculation (Steps 4 and 5).
    Therefore, if the intermediate seqcol representation includes any non-inherent attributes, these must be removed before proceeding to step 4.
    In the simple example, the `lengths` attribute is not `inherent` and must be filtered.


#### Step 4: Apply RFC-8785 again to canonicalize the JSON of the new seqcol object representation.

Here, we repeat step 2, except instead of applying RFC-8785 to each value separately, we apply it to the entire object.
This will result in a canonical bytestring representation of the object, shown here using Python notation:

```
b'{"names":"g04lKdxiYtG3dOGeUC5AdKEifw65G0Wp","sequences":"rD29ZKmEqwwHRXjiQ36p6UMZQ5hemmsb"}'
```

#### Step 5: Digest the final canonical representation again using the GA4GH digest algorithm.

Again using the same approach as in step 3, we now apply the GA4GH digest algorithm to the canonicalized bytestring.
The result is the final unique digest for this sequence collection:

```
sjNNwm4zov3Dl0FRWbRTcZwzqrTQKIqL
```

#### Terminology

The recursive encoding algorithm leads to several ways to represent a sequence collection. We refer to these representations as "levels". The level number represents the number of "lookups" you'd have to do from the "top level" digest. So, we have:

##### Level 0 (top-level digest)

Just a plain digest, also known as the "top-level digest". This corresponds to **0 database lookups**. Example:
```
Zjx9_tD2o-1yKB6RR2v2g3W9c5ufydUc
```

##### Level 1

What you'd get when you look up the digest with **1 database lookup**. We sometimes refer to this as the "array digests" or "attribute digests", because it is made up a digest for each attribute in the sequence collection. Example:

```json
{
  "lengths": "QWhPI-Cll_0Y5NJ_2krRryuV97vzhbgJ",
  "names": "1zOnTYE5slcISev72o62ySxbssEXeoUL",
  "sequences": "uPCc00rq-daL3zPnzYH-sBg9_z7HpB8B"
}
```

##### Level 2

What you'd get with **2 database lookups**. This is the most common representation, and hence, it the level of the *canonical seqcol representation*. Example:

```json
{
  "lengths": [
    1216,
    970,
    1788
  ],
  "names": [
    "A",
    "B",
    "C"
  ],
  "sequences": [
    "SQ.OL3sVAcd_5IZaDxUkH-yQkLmBz2iwY0s",
    "SQ.kny8cdhEEPHXoNlXmps8NQapGtUKZlM9",
    "SQ.DA-GLdXVihnYKs-fBS5MMgqMi7tVMJbt"
  ]
}
```


### 3. API: A server API specification for retrieving and comparing sequence collections.

The API has these top-level endpoints:

1. `/service-info`, for describing information about the service.
2. `/collection`, for retrieving sequence collections.
3. `/comparison`, for comparing two sequence collections.
4. `/list`, for retrieving a list of objects.
5. `/attribute`, for retrieving the value of a specific attribute.

In addition, a RECOMMENDED endpoint at `/openapi.json` SHOULD provide OpenAPI documentation.

Under these umbrella endpoints are a few more specific sub-endpoints, described in detail below:



#### 3.1 Service info 
- *Endpoint*: `GET /service-info` (`REQUIRED`)
- *Description*: The service info endpoint provides information about the service
- *Return value*: Must include the Seqcol JSON Schema that is used by this server

The `/service-info` endpoint should follow the [GA4GH-wide specification for service info](https://github.com/ga4gh-discovery/ga4gh-service-info/) for general description of the service.
Then, it should also add a few specific pieces of information under a `seqcol` property:

 - `schema`: MUST return the JSON Schema implemented by the server.

##### The service-info JSON-schema document

The `schema` attribute of `service-info` return value MUST provide a single schema, this MUST include all possible attributes your service may provide. In other words, you cannot have a collection with an attribute that is not defined in your schema.

We RECOMMEND the schema only define terms actually used in at least one collection served; however, it is allowed for the schema to contain extra terms that are not used in any collections in the server.

We RECOMMEND your schema use property-level refs to point to terms defined by the minimal base seqcol schema. However, it is also allowed for the schema to embed all definitions locally. The base seqcol schema will be made available as the spec is finalized.

For example, here's a JSON schema that uses a `ref` to reference the approved seqcol schema:

```yaml
description: "A collection of biological sequences."
type: object
"$id": "https://example.com/sequence_collection",  # URI to main seqcol definition
properties:
  lengths:
    "$ref": "/lengths"
  names:
    "$ref": "/names"
  sequences:
    "$ref": "/sequences"
required:
  - names
  - sequences
ga4gh:
  inherent:
    - names
    - sequences
```


#### 3.2 Collection

- *Endpoint*: `GET /collection/:digest?level=:level` (`REQUIRED`)
- *Description*: Retrieves original sequence collection from a database, keyed by the unique top-level digest. Here `:digest` is the seqcol digest computed above. The level corresponds to the "expansion level" of the returned sequence collection returned. The default is `?level=2`, which returns the canonical structure.
- *Return value*: The sequence collection identified by the `:digest` variable. The structure of the data `MUST` be modulated by the `:level` query parameter.  Specifying `?level=2` returns the canonical structure, and `?level=1` returns the collection with digested attributes.

Non-inherent attributes `MUST` be stored and returned by the collection endpoint alongside inherent attributes.

#### 3.3 Comparison

- *Endpoint 1*: `GET /comparison/:digest1/:digest2` (`RECOMMENDED`) Two-digest comparison    
- *Endpoint 2*: `POST /comparison/:digest1` (`RECOMMENDED`) One-digest POST comparison 
- *Description*: The comparison function specifies an API endpoint that allows a user to compare two sequence collections. The collections are provided either as two digests (the `GET` endpoint) or as one digest representing a database collection, and one local user-provided collection provided via `POST`. For the `POST` endpoint variant, the user-provided local collection should be provided as a [level 2 representation](#terminology) (AKA the canonical seqcol representation) in the `BODY` of the `POST` request.
- *Return value*: The output is an assessment of compatibility between those sequence collections. If implemented, both variants of the `/comparison` endpoint must `MUST` return an object in JSON format with these 3 keys: `digests`, `attributes`, and `elements`, as described below (see also an example after the descriptions):
    - `digests`: an object with 2 elements, with keys *a* and *b*, and values either the [level 0 seqcol digests](#terminology) for the compared collections, or *null* (undefined). The value MUST be the level 0 seqcol digest for any digests provided by the user for the comparison. However, it is OPTIONAL for the server to provide digests if the user provided the sequence collection contents, rather than a digest. In this case, the server MAY compute and return the level 0 seqcol digest, or it MAY return *null* (undefined) in this element for any corresponding sequence collection.
    - `attributes`: an object with 3 elements, with keys *a_only*, *b_only*, and *a_and_b*. The value of each element is a list of array names corresponding to arrays only present in a, only present in b, or present in both a and b.
    - `array_elements`: An object with 4 elements: *a_count*, *b_count*, *a_and_b_count*, and *a_and_b_same_order*. The 3 attributes with *_count* are objects with names corresponding to each array present in the collection, or in both  collections (for *a_and_b_count*), with values as the number of elements present either in one collection, or in both collections for the given array. *a_and_b_same_order* is also an object with names corresponding to arrays, and the values a boolean following the same-order specification below.


Example `/comparison` return value: 
```json
{
  "digests": {
    "a": "514c871928a74885ce981faa61ccbb1a",
    "b": "c345e091cce0b1df78bfc124b03fba1c"
  },
  "attributes": {
    "a_only": [],
    "b_only": [],
    "a_and_b": [
      "lengths",
      "names",
      "sequences"
    ]
  },
  "array_elements": {
    "a_count": {
      "lengths": 195,
      "names": 195,
      "sequences": 195
    },
    "b_count": {
      "lengths": 25,
      "names": 25,
      "sequences": 25
    },
    "a_and_b_count": {
      "lengths": 25,
      "names": 25,
      "sequences": 0
    },
    "a_and_b_same_order": {
      "lengths": false,
      "names": false,
      "sequences": null
    }
  }
}
```

##### Same-order specification

The comparison return includes an *a_and_b_same_order* boolean value for each array that is present in both collections. The defined value of this attribute is:

- undefined (*null*) if there are fewer than 2 overlapping elements
- undefined (*null*) if there are unbalanced duplicates present (see definition below)
- *true* if all matching elements are in the same order in the two arrays
- *false* otherwise.

An *unbalanced duplicate* is used in contrast with a *balanced duplicate*. Balanced means the duplicates are the same in both arrays.
When the duplicates are balanced, order is still defined; but if duplicates are unbalanced, this means an array has duplicates not present in the other, and in that case, order is not defined.

##### Interpreting the result of the compare function

The output of the comparison function provides information-rich feedback about the two collections.
These details can be used to make a variety of inferences comparing two collections, but it can take some thought to interpret.
For more details about how to interpret the results of the comparison function to determine different types of compatibility, please see the [howto guide on comparing sequencing collections](compare_collections.md).


#### 3.4 List

- *Endpoint*: `GET /list/:object_type?page=:page&page_size=:page_size&:attribute1=:attribute1_level1_repr&attribute2=:attribute2_level1_repr` (`REQUIRED`)
- *Description*: Lists identifiers for a given object type in singular form (*e.g.* `/list/collection`). This endpoint provides a way to discover what sequence collections a service provides.
  Returned lists can be filtered to only objects with certain attribute values using query parameters.
  Page numbering begins at page 0 (the first page is page 0).
- *Return value*: The output is a paged list of identifiers following the [GA4GH paging guide format](https://www.ga4gh.org/what-we-do/technical-alignment-subcommittee-tasc/#section_1), grouped into a `results` and a `pagination` section. If no `?:attribute=:attribute_level1_repr` query parameters are provided, the endpoint will return all items (paged). Adding one `:attribute` and `:attribute_level1_repr` pair as *query parameters*  will filter results to only the collections with the given level 1 attribute digest. If multiple attributes are provided, the filter should require ALL of these attributes to match (so multiple attributes are treated with an `AND` operator).


Example return value:

```json
{
  "results": [
    ...
  ],
  "pagination": {
    "page": 1,
    "page_size": 100,
    "total": 165,
  }
}
```

The `list` endpoint MUST be implemented, and MUST allow filtering using any attribute defined in the schema, *except attributes marked as passthru*.
For attributes marked as *passthru*, the list endpoint MAY provide filtering capability.

#### 3.5 Attribute

- *Endpoint*: `GET /attribute/:object_type/:attribute_name/:digest` (`REQUIRED`)
- *Description*: Retrieves values of specific attributes in a sequence collection. Here `:object_type` must be `collection` for a sequence collection object; `:attribute_name` is the name of an attribute, such as `sequences`, `names`, or `sorted_sequences`. `:digest` is the digest of the attribute value, as computed above.
- *Return value*: The attribute value identified by the `:digest` variable. The structure of the should correspond to the value of the attribute in the canonical structure.


Example `/attribute/collection/lengths/QWhPI-Cll_0Y5NJ_2krRryuV97vzhbgJ` return value:

```
[1216, 970, 1788]
```

Example `/attribute/collection/names/1zOnTYE5slcISev72o62ySxbssEXeoUL` return value:

```
["A", "B", "C"]
```

The attribute endpoint MUST be functional for any attribute defined in the schema, *except those marked as transient or passthru*.
The endpoint SHOULD NOT respond to requests for attributes marked as *passthru* OR *transient*.
For more information on transient and passthru attributes, see [Section 4](#4-extending-the-schema-schema-attribute-qualifiers).


!!! note "Definition of `object_type`"

    The `/list` and `/attribute` endpoints both use an `:object_type` path parameter. The `object_type` should always be the *singular* descriptor of objects provided by the server. In this version of the Sequence Collections specification, the `object_type` is always `collection`; so the only allowable endpoints would be `/list/collection` and `/attribute/collection/:attribute_name/:digest`. We call this `object_type` because future versions of the specification may allow retrieving lists or attributes of other types of objects.


#### 3.6 OpenAPI documentation

In addition to the primary top-level endpoints, it is RECOMMENDED that the service provide `/openapi.json`, an OpenAPI-compatible description of the endpoints.




### 4. Extending the schema: Schema attribute qualifiers

#### 4.1 Introduction to attribute qualifiers

The Sequence Collections specification is designed to be extensible.
This will let you build additional capability on top of the minimal Sequence Collections standard.
You can do this by extending the schema to include ancillary custom attributes.
To allow other services to understand something about what these attributes are for, you can annotate them in the schema using  *attribute qualifiers*.
This allows you to indicate what *type* of attribute your custom attributes are, which govern how the service should respond to requests.
This section will describe the 4 attribute qualifiers you may add to the schema to qualify custom attributes.

#### 4.2 Collated attributes

Collated attributes are attributes whose values match 1-to-1 with the sequences in the collection and are represented in the same order.
A collated attribute by definition has the same number of elements as the number of sequences in the collection.
It is also in the same order as the sequences in the collection.


#### 4.3 Inherent attributes

Inherent attributes are those that contribute to the digest.
The specification in section 1, *Encoding*, described how to structure a sequence collection and then apply an algorithm to compute a digest for it.
What if you have ancillary information that goes with a collection, but shouldn't contribute to the digest?
We have found a lot of useful use cases for information that should go along with a seqcol, but should not contribute to the *identity* of that seqcol.
This is a useful construct as it allows us to include information in a collection that does not affect the digest that is computed for that collection.
One simple example is the "author" or "uploader" of a reference sequence; this is useful information to store alongside this collection, but we wouldn't want the same collection with two different authors to have a different digest! Seqcol refers to these as *non-inherent attributes*, meaning they are not part of the core identity of the sequence collection.
Non-inherent attributes are defined in the seqcol schema, but excluded from the `inherent` list. 

See: [ADR on 2023-03-22 regarding inherent attributes](decision_record.md#2023-03-22-seqcol-schemas-must-specify-inherent-attributes)

#### 4.4 Passthru attributes

Passthru attributes have the same representation at level 1 and level 2.
In other words, they are *not digested* in transition from level 2 to level 1.
This is not the case for most attributes; most attributes of the canonical (level 2) seqcol representation are digested to create the level 1 representation.
But sometimes, we have an attribute for which digesting makes little sense. 
These attributes are passed through without transformation, so they show up on the level 1 representation in the same form as the level 2 representation.
Thus, we refer to them as passthru attributes.

Here's how passthru attributes behave in the endpoints:

- `/list`: The server MAY allow filtering on passthru attributes, but this is not required.
- `/collection`: At both level 1 and level 2, the collection object includes the same passthru attribute representation. 
- `/comparison`: Passthru attributes are listed in the 'attributes' section, but are not listed under 'array_elements'.
- `/attribute`: Passthru attributes cannot be used with the attribute endpoint, as the return would be the same as the query.


#### 4.5 Transient attributes

A transient attribute is an attribute that only has a level 1 representation stored in the server.
Transient attributes therefore *cannot be retrieved* through the `/attribute` endpoint.
All other attributes of the sequence collection can be retrieved through the `/attribute` endpoint.
The transient qualifier would apply to attribute that we intend to be used primarily as an identifier.
In this case, we don't necessarily want to store the original content that went into the digest into the database.
We really just want the final attribute.
These attributes are called transient because the content of the attribute is no longer stored and is therefore no longer retrievable.

Here's how transient attributes behave in the endpoints:

- `/list`: No change; a transient attribute level1 representation can be used to list sequence collections that contain it.
- `/collection`: For level 1 representation, no change; the collection object includes the transient attribute level 1 representation. For level 2 representation, there *is* a change; transient attributes have no level 2 representation on the server, so the sequence collection SHOULD leave this attribute out of the level 2 representation.
- `/comparison`: Transient attributes are listed in the 'attributes' section, but are not listed under 'array_elements' because there is no level 2 representation.
- `/attribute`: Transient attributes cannot be used with the attribute endpoint (there is no value to retrieve)


#### 4.6 Qualifier summary table

The global qualifiers are all concerned with how the representations are treated when converting between different detail levels.
The *inherent* qualifier is related to the level 1 &rarr; 0 transition. 
It is true if the level 1 representation is included during creation of level 0 representation.
Then the *passthru* and *transient* qualifiers are related to the level 2 &rarr; 1 transition.


Qualifer | Level1? | Level2? | Notes
--- | ----- | ----- | -----
*none* | as normal | full content | Default state; the level 1 representation is a digest of the level 2 representation.
passthru | as normal | same as level1 | True if the level 2 representation is the same as the level 1 representation.
transient | as normal | not present | True if the level 2 representation is not present.



#### 4.7 Method of specifying attribute qualifiers

In JSON Schema, there are 2 ways to qualify properties: 1) a local qualifier, using a key under a property; or 2) an object-level qualifier, which is specified with a keyed list of properties up one level.
For example, you annotate a property's `type` with a local qualifier, underneath the property, like this:

```console
properties:
  names:
    type: array
```

However, you specify that a property is `required` by adding it to an object-level `required` list that's parallel to the `properties` keyword:

```console
properties:
  names:
    type: array
required:
  - names
```

In sequence collections, we define `collated` as a local qualifier.
Local qualifiers fit better for qualifiers independent of the object as a whole.
They are qualities of a property that persist if the property were moved onto a different object.
For example, the `type` of an attribute is consistent, regardless of what object that attribute were defined on.
In contrast, object-level qualifier lists fit better for qualifiers that depend on the object as a whole.
They are qualities of a property that depend on the object context in which the property is defined.
For example, the `required` modifier is not really meaningful except in the context of the object as a whole.
A particular property could be required for one object type, but not for another, and it's really the object that induces the requirement, not the property itself.


We reasoned that `inherent`, `transient`, and `passthru` are global qualifiers, like `required`, which describe the role of an attribute in the context of the whole object.
For example, an attribute that is inherent to one type of object need not be inherent to another.
Therefore, it makes sense to treat this concept the same way JSON schema treats `required`.
In contrast, the idea of `collated` describes a property independently: Whether an attribute is collated is part of the definition of the attribute; if the attribute were moved to a different object, it would still be collated.

Finally, the 3 global qualiers are grouped under the 'ga4gh' key for consistency with other GA4GH specifications, and to group the seqcol-specific extended functionality into one place.



### 5. Custom and recommended ancillary attributes

The base schema described in Section 1 is a minimal starting point, which can be extended with additional custom attributes as needed.
As stated in Section one, we RECOMMEND the schemas *extend* the base schema with any custom attributes.
Furthermore, we RECOMMEND the extended schema add only non-inherent attributes, so that the top-level digests remain compatible.
Here, we specify several standard non-inherent attributes that we RECOMMEND be also included in the schema.

Furthermore, some attributes do not need to follow the typical encoding process, for whatever reason.
Basically, custom attributes can be defined, and they are also allowed to specify their own encoding process.


#### 5.1 The `name_length_pairs` attribute (`RECOMMENDED`)

The `name_length_pairs` attribute is a *non-inherent* attribute of a sequence collection with a formal definition, provided here.
This attribute provides a way to look up the ordered coordinate system (the "chrom sizes") for a sequence collection.
It is created deterministically from the `names` and `lengths` attributes in the collection; it *does not* depend on the actual sequence content, so it is consistent across two collections with different sequence content if they have the same `names` and `lengths`, which are correctly collated.
This attribute is `RECOMMENDED` to allow retrieval of the coordinate system for a given reference sequence collection.

##### Algorithm

1. Lump together each name-length pair from the primary collated `names` and `lengths` into an object, like `{"length":123,"name":"chr1"}`.
2. Build a collated list, corresponing to the names and lengths of the object (*e.g.* `[{"length":123,"name":"chr1"},{"length":456,"name":"chr2"}],...`)
3. Add as a collated attribute to the sequence collection object.

##### Qualifiers (RECOMMENDED)

- inherent: false
- collated: true
- passthru: false
- transient: false


#### 5.2 The `sorted_name_length_pairs` attribute (`RECOMMENDED`)

The `sorted_name_length_pairs` attribute is similar to the `name_length_pairs` attribute, but it is sorted.
When digested, this attribute provides a digest for an order-invariant coordinate system for a sequence collection.
Because it is *non-inherent*, it does not affect the identity (digest) of the collection.
but with pairs not necessarily in the same order.

This attribute is `RECOMMENDED` to allow unified genome browser visualization of data defined on different reference sequence collections. For more rationale and use cases of `sorted_name_length_pairs`, see [*Footnote F4*](#f4-use-cases-for-the-sorted_name_length_pairs-non-inherent-attribute).

##### Algorithm

1. Lump together each name-length pair from the primary collated `names` and `lengths` into an object, like `{"length":123,"name":"chr1"}`.
2. Canonicalize JSON according to the seqcol spec (using RFC-8785).
3. Digest each name-length pair string individually.
4. Sort the digests lexicographically.
5. Add to the sequence collection object.

##### Qualifiers (RECOMMENDED)

- inherent: false
- collated: false
- passthru: false
- transient: **true**

#### 5.3 The `sorted_sequences` attribute (`OPTIONAL`)

The `sorted_sequences` attribute is a *non-inherent* attribute of a sequence collection, with a formal definition.
Providing this attribute is `OPTIONAL`.
When digested, this attribute provides a digest representing an order-invariant set of unnamed sequences.
It provides a way to compare two sequence collections to see if their sequence content is identical, but just in a different order.
Such a comparison can, of course, be made by the comparison function, so why might you want to include this attribute as well?
Simply that for some large-scale use cases, comparing the sequence content without considering order is something that needs to be done repeatedly and for a huge number of collections.
In these cases, using the comparison function could be computationally prohibitive.
This digest allows the comparison to be pre-computed, and more easily compared.

##### Algorithm

1. Take the array of the `sequences` attribute (an array of sequence digests) and sort it lexicographically.
2. Add to the sequence collection object as the `sorted_sequences` attribute, which is non-inherent and non-collated.

##### Qualifiers (RECOMMENDED)

- inherent: false
- collated: false
- passthru: false
- transient: false

---

## Footnotes

### F1. Why use an array-oriented structure instead of a sequence-oriented structure?

In the canonical seqcol object structure, we first organize the sequence collection into what we called an "array-oriented" data structure, which is a list of collated arrays (names, lengths, sequences, *etc.*).
An alternative we considered was a "sequence-oriented" structure, which would group each sequence with some attributes, like `{name, length, sequence}`, and structure the collection as an array of such objects.
While the latter is intuitive, as it captures each sequence object with some accompanying attributes as independent entities, there are several reasons we settled on the array-oriented structure instead: 

  1. Flexibility and backwards compatibility of sequence attributes. What happens for an implementation that adds a new attribute? For example, if an implementation adds a `topology` attribute to the sequences, in the sequence-oriented structure, this would alter the sequence objects and thereby change their digests. In the array-based structure, since we digest the arrays individually, the digests of the other arrays are not changed. Thus, the array-oriented structure emphasizes flexibility of attributes, where the sequence-oriented structure would emphasize flexibility of sequences. In other words, the array-based structure makes it straightforward to mix-and-match *attributes* of the collection. Because each attribute is independent and not integrated into individual sequence objects, it is simpler to select and build subsets and permutations of attributes. We reasoned that flexibility of attributes was desirable.

  2. Conciseness. Sequence collections may be used for tens of thousands or even millions of sequences. For example, a transcriptome may contain a million transcripts. The array-oriented data structure is a more concise representation for digesting collections with many elements because the attribute names are specified once *per collection* instead of once *per element*. Furthermore, the level 1 representation of the sequence collection is more concise for large collections, since we only need one digest *per attribute*, rather than one digest *per sequence*. 

  3. Utility of intermediate digests. The array-oriented approach provides useful intermediate digests for each attribute. This digest can be used to test for matching sets of sequences, or matching coordinate systems, using the individual component digests. With a sequence-oriented framework, this would require traversing down a layer deeper, to the individual elements, to establish identity of individual components. The alternative advantage we would have from a sequence-oriented structure would be identifiers for *annotated sequences*. We can gain the advantages of these digests through adding a custom non-inherent, but collated attribute that calculates a unique digest for each element based on the selected attributes of interest, *e.g.* `named_sequences` (digest of *e.g.* `b'{"name":"chr1","sequence":"SQ.2648ae1bacce4ec4b6cf337dcae37816"}'`).

See [ADR on 2021-06-30 on array-oriented structure](decision_record.md#2021-06-30-use-array-based-data-structure-and-multi-tiered-digests)

### F2. RFC-8785 does not apply to refget sequences

A note to clarify potential confusion with RFC-8785. While the sequence collection specification determines that RFC-8785 will be used to canonicalize the JSON before digesting, this is specific to sequence collections, it *does not apply to the original refget sequences protocol*. According to the sequences protocol, sequences are digested as un-quoted strings. If RFC-8785 were applied at the level of individual sequences, they would be quoted to become valid JSON, which would change the digest. Since the sequences protocol predated the sequence collections protocol, it did not use RFC-8785; and anyway, the sequences are just primitive types so a canonicalization scheme doesn't add anything. This leads to the slight confusion that RFC-8785 canonicalization is only applied to the objects in the sequence collections, and not to the primitives when the underlying sequences are digested. In other words, from the perspective of Sequence Collections, we just take the sequence digests at face value, as handled by a third party; their content is not digested as part of the collections algorithm at a deeper level.

### F3. The GA4GH digest algorithm

The GA4GH digest algorithm, `sha512t24u`, was created as part of the [Variation Representation Specification standard](https://vrs.ga4gh.org/en/stable/impl-guide/computed_identifiers.html). 
This procedure is described as ([Hart _et al_. 2020](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0239883)):

- performing a SHA-512 digest on a binary blob of data
- truncate the resulting digest to 24 bytes
- encodes the 24 bytes using `base64url` ([RFC 4648](https://datatracker.ietf.org/doc/html/rfc4648#section-5)) resulting in a 32 character string

In Python, the digest can be computed with this function:

```python
import base64
import hashlib

def sha512t24u_digest(seq: bytes) -> str:
    """ GA4GH digest function """
    offset = 24
    digest = hashlib.sha512(seq).digest()
    tdigest_b64us = base64.urlsafe_b64encode(digest[:offset])
    return tdigest_b64us.decode("ascii")
```

See: [ADR from 2023-01-25 on digest algorithm](decision_record.md#2023-01-25-digest-algorithm)

### F4. Use cases for the `sorted_name_length_pairs` non-inherent attribute

One motivation for this attribute comes from genome browsers, which may display genomic loci of interest (*e.g.* BED files).
The genome browser should only show BED files if they annotate the same coordinate system as the reference genome.
This is looser than strict identity, since we don't really care what the underlying sequence characters are, as long as the positions are comparable.
We also don't care about the order of the sequences.
Instead, we need them to match level 1 digest of the `sorted_name_length_pairs` attribute.
Thus, to assert that a BED file can be viewed for a particular genome, we compare the `sorted_name_length_pairs` digest of our reference genome with the sequence collection used to generate the file.
There are only two possibilities for compatibility: 1) If the digests are equal, then the data file is directly compatible; 2) If not, we must check the `comparison` endpoint to see whether the `name_length_pairs` attribute of the sequence collection is a direct subset of the same array in the sequence collection attached to the genome browser instance.
If so, the data file is still compatible. 

For efficiency, if the second case is true, we may cache the `sorted_name_length_pairs` digest in a list of known compatible reference genomes.
In practice, this list will be short.
Thus, in a production setting, the full compatibility check can be reduced to a lookup into a short, pre-generated list of `sorted_name_length_pairs` digests.

See: [ADR from 2023-07-12 on sorted name-length pairs](decision_record.md#2023-07-12-implementations-should-provide-sorted_name_length_pairs-and-comparison-endpoint)

### F5. Adding new schema attributes

A strength of the seqcol standard is that the schema definition can be modified for particular use cases, for example, by adding new attributes into a sequence collection.
This will allow different communities to use the standard without necessarily needing to subscribe to identical schemas, allowing the standard to be more generally useful.
However, if communities define too many custom attributes, this leads to the possibility of fragmentation.
For example, two implementations may start using the same attribute name to refer to different concepts.
While the attributes would still be formally defined in the respective schemas provided by each implementation, calling different concepts by the same name would come at the cost of interoperability.
Therefore, the standard will also include in the schema a list of formally defined attributes, to encourage interoperability of these attributes.
The goal is not to include all possible attributes in the schema, but just those likely to be used repeatedly, to encourage interoperable use of those attribute names.
An implementation may propose a new attribute to be added to this extended schema by raising an issue on the GitHub repository.
The proposed attributes and definition can then be approved through discussion during the refget working group calls and ultimately added to the approved extended seqcol schema.
These GitHub issues should be created with the label 'schema-term'.
You can follow these issues (or raise your own) at <https://github.com/ga4gh/seqcol-spec/issues?q=is%3Aissue+label%3Aschema-term+>.

### F6. Custom encoding algorithms

A core part of Sequence Collections specification is the *encoding* algorithm, which describes how to create the digest for a sequence collection.
The encoding process can be divided into two steps; first, the attributes are encoded into the level 1 representation, and then this is encoded to produce the final digest (also called the level 0 or top level representation).
The first part of this process, encoding from level 2 to level 1, is the default; this is applied to any attributes that don't have something else defined specifically as part of the attribute definition.
This is the way all the minimal attributes (names, lengths, and sequences) should behave.
But custom attributes MAY diverge from this approach by defining their own encoding procedure that defines how the level 1 digest is computed from the level 2 representation.
For example, in the list of recommended ancillary attributes, `name_length_pairs` does not define a custom procedure for encoding, so this would follow the default procedure.
An alternative custom attribute, though, MAY specify how this encoding procedure happens.
