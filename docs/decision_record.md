# Architectural Decision Record

*This is a record of decisions made during specification development. Each entry describes a decision that has been approved by the team members. Collectively, this ADR describes an institutional memory for decisions and their rationales, including known limitations. The goal is to avoid repeated discussion of previous decisions, formally acknowledge limitations, preserve and articulate reasons behind the decisions, and share this information with the broader community.* 

## Contents: 

[TOC]

## 2022-09-05 - How sequence collection are serialized prior to digestion

### Decision

The serialisation of a sequence collection will use the following steps

 1. Apply RFC-8785 on each array of level 2
 2. Digest each the canonical representation of each array
 3. Create object representation of the seq-col using array names and digested arrays
 4. Apply RFC-8785 on the object representation
 5. Digest the final canonical representation


#### 1. Apply RFC-8785 for converting from level 2 to level 1

For example the length array at level 2:
```json
[248956422, 242193529, 198295559]
```

Will be serialised using RFC-8785 and digested as a binary string. Here the output of the python implementation: 

```python
b'[248956422,242193529,198295559]'
```

It would also support any UTF-8 character. For example this array of names
```json
["染色体-1","染色体-2","染色体-3"]
```

Would create the following serialisation:

```python
b'["\xe6\x9f\x93\xe8\x89\xb2\xe4\xbd\x93-1","\xe6\x9f\x93\xe8\x89\xb2\xe4\xbd\x93-2","\xe6\x9f\x93\xe8\x89\xb2\xe4\xbd\x93-3"]'
```

#### 2. Digest of the canonical representation

The canonical string representation is then digested. Assuming the use of GA4GH (sha512 trim to 24) digest, the following array of length

```python
b'[248956422,242193529,198295559]'
```

would be converted

```json
"5K4odB173rjao1Cnbk5BnvLt9V7aPAa2"
```

#### 3. Creation of an object composed of the array names and the digested arrays
An object is created with the array name as properties and the digest as value.
Example the following collection: 
```json
{
    "sequences": "EiYgJtUfGyad7wf5atL5OG4Fkzohp2qe",
    "lengths": "5K4odB173rjao1Cnbk5BnvLt9V7aPAa2",
    "names": "g04lKdxiYtG3dOGeUC5AdKEifw65G0Wp"
}
```

#### 4. Use RFC-8785 on the object
This will create a canonical representation of the object 

```python
b'{"lengths":"5K4odB173rjao1Cnbk5BnvLt9V7aPAa2","names":"g04lKdxiYtG3dOGeUC5AdKEifw65G0Wp","sequences":"EiYgJtUfGyad7wf5atL5OG4Fkzohp2qe"}'
```

#### 5. Digest the final canonical representation
Finally the canonical, representation is digested again to produce the identifier

```json
"S3LCyI788LE6vq89Tc_LojEcsMZRixzP"
```

### Rationale
The decision to use the serialisation of array and object provided in RFC-8785 allows sequence collection to support any type of characters and rely on a documented standard that offer implementation in multiple languages.
It also future-proofs the serialisation method if we ever allow complex object to be element of the array.

### Linked issues

 - [https://github.com/ga4gh/seqcol-spec/issues/1](https://github.com/ga4gh/seqcol-spec/issues/1)
 - [https://github.com/ga4gh/seqcol-spec/issues/25](https://github.com/ga4gh/seqcol-spec/issues/25)
 - [https://github.com/ga4gh/seqcol-spec/issues/33](https://github.com/ga4gh/seqcol-spec/issues/33)


### Known limitations

The JSON canonical serialisation defined in RFC-8785 has a limited set of reference implementation. It is possible that its implementation might make sequence collection implementation more difficult in other languages.  

## 2022-06-15 - Structure for the return value of the comparison API endpoint

### Decision

The compare function return value MUST be an object following the REQUIRED format specified below.

**REQUIRED**: The endpoint MUST return, in JSON format, an object with these 3 keys: "digests", "arrays", "elements". 

- *digests*: an object with 2 elements, with keys *a* and *b*, and values either the level 0 seqcol digests for the compared collections, or *undefined (null)*. The value MUST be the level 0 seqcol digest for any digests provided by the user for the comparison. However, it is OPTIONAL for the server to provide digests if the user provided the sequence collection contents, rather than a digest. In this case, the server MAY compute and return the level 0 seqcol digest, or it MAY return *undefined (null)* in this element for any corresponding sequence collection.
- *arrays*: an object with 3 elements, with keys *a-only*, *b-only*, and *a-and-b*. The value of each element is a list of array names corresponding to arrays only present in a, only present in b, or present in both a and b.
- *elements*: An object with 3 elements: *total*, *a-and-b*, and *a-and-b-same-order*. *total* is an object with *a* and *b* keys, values corresponding to the total number of elements in the arrays for the corresponding collection. *a-and-b* is an object with names corresponding to each array present in both collections (in *arrays.a-and-b*), with values as the number of elements present in both collections for the given array. *a-and-b-same-order* is also an object with names corresponding to arrays, and the values a boolean following the same-order specification below.

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

### Rationale

The primary purpose of the compare function is to provide a high-level view of how two sequence collections match and differ. The primary use cases are to see if collections are identical or subsets, and to assess the degree of overlap in each attribute (such as sharing all sequence digests, sequence names, or lengths). If more details are needed, the user will need to look in more depth at the raw elements of the sequence collection. It's important to have a fast, easy-to-implement, and minimal payload function to provide answers to the common question about "how compatible are these two collections".

### Linked issues

- https://github.com/ga4gh/seqcol-spec/issues/21
- https://github.com/ga4gh/seqcol-spec/issues/7

### Alternatives considered

We considered a simpler arrangement that would only return true/false values as to whether the arrays matched but in the different order, or contained any matching elements vs no matching elements. While this would have been faster to compute than the counting approach we settled on, there was concern that it would not be enough information to interpret the comparison. We also considered more information-rich values that would enumerate overlapping or non-overlapping elements. We finally concluded that the most useful would be the middle ground proposed here, where you get counts but no enumerated elements. This provides sufficient information to make a pretty detailed comparison, and can still be computed relatively quickly and keeps the payload size small and predictable.

### Known limitations

Someone may want to return more information than this, such as enumerating the specific elements in each category. However, this use case would be problematic for large collections, like a transcriptome. We may in the future provide an update to the specification that defines how this information should be returned, but for now, we leave the specification at this minimum requirement.

## 2022-06-15 We will define the elements of a sequence collections using a schema

### Decision

The elements of a sequence collection will be defined and described using JSON Schema. The exact contents of the JSON Schema will be determined later. Here is an example JSON Schema as an illustration of the concept (encoded in YAML for readability):

```yaml

description: "A collection of sequences, representing biological sequences including nucleotide or amino acid sequences. For example, a sequence collection may represent the set of chromosomes in a reference genome, the set of transcripts in a transcriptome, a collection of reference microbial sequences of a specific gene, or any other set of sequences."
type: object
properties:
  lengths:
    type: array
    description: "Number of elements, such as nucleotides or amino acids, in each sequence."
    items:
      type: integer
  masks:
    type: array
    description: "Digests of subsequence masks indicating subsequences to be excluded from an analysis, such as repeats"
    items:
      type: string
  names:
    type: array
    description: "Human-readable identifiers of each sequence, commonly called chromosome names."
    items:
      type: string
  priorities:
    type: array
    description: "Annotation of whether each sequence is a primary or secondary component in the collection."
    items:
      type: boolean
  sequences:
    type: array
    description: "Digests of sequences computed using the GA4GH digest algorithm (sha512t24u)."
    items:
      type: string
      description: "Actual sequence content"
  topologies:
    type: array
    description: "Annotation of whether each sequence represents a linear or other topology."
    items:
      type: string
      enum: ["circular", "linear"]
      default: "linear"
required:
  - lengths
digested:
  - lengths
  - names
  - masks
  - priorities
  - sequences
  - topologies
```



### Rationale

We need a formal definition of a sequence collection. The schema provides a machine-usable definition that specifies the names and types of what we envision as a sequence collection. It also provides a convenient way to describe those elements in human-understandable ways through the `description` field. Thus, this schema solves a number of issues. Most importantly, it answers the question of *what are the elements of the sequence collection, and how are they defined?* For example, this schema answers the more specific question: *what are the allowable or expected algorithms for items included in the sequence array (or possibly other, custom digested arrays?)*. The schema establishes a standard expectation for elements that will go into general sequence collections. Implementations are free to add to this schema for their instances as needed.

### Linked issues

- https://github.com/ga4gh/seqcol-spec/issues/8
- https://github.com/ga4gh/seqcol-spec/issues/6



## 2021-12-01 - Endpoint names and structure

### Decision

The endpoint names will be:

- `GET /service-info` for GA4GH service info
- `GET /collection/{digest}` for retrieving a sequence collection
- `GET /comparison/{digest1}/{digest2}` for comparing two collections in the database
- `POST /comparison/{digest1}` for comparing one database collection to a local user-provided collection.

The POST body for the local comparison is a "level 2" sequence collection, like this:

```
{
  "lengths": [
    "248956422",
    "242193529",
    "198295559"
  ],
  "names": [
    "chr1",
    "chr2",
    "chr3"
  ],
  "sequences": [
    "a004bc1b0bf05fc668cab6bbfd93d3eb",
    "0ccf3a67666ac53f99fcad19768f2dde",
    "bda7b228789169ae811dd8d676d517ca"
  ]
}
```

### Rationale

We wanted to stick with the REST guideline of noun endpoints with GET that describe what you are retrieving. As recommended in the [service-info specification](https://github.com/ga4gh-discovery/ga4gh-service-info#how-do-i-describe-a-service-implementing-multiple-specifications), a prefix, like `/seqcol/...` could be added by a service that implemented multiple specifications, but this kind of namespace it outside the scope of the specification itself. We considered doing `/{digest1}/compare/{digest2}` and that would have been fine. In the end we liked the symmetry of `/comparison` and `/collection` as parallel endpoints. For the retrieval endpoint we considered `/secol` or `/sequence-collection` or `/seqCol`, but wanted to keep structure parallel to the refget `/sequence` endpoint.

### Limitations

For the `POST comparison` endpoint, we made 2 limitations to simplify the implementation of the function. First, we do not require it to allow comparing 2 local collections, which could be enabled, but we reason that users should always be comparing against something in the database, and this prevents abusing the system as a computing engine. We also disallowed (or at least don't explicitly require) comparing a level 1 collection (which consists of a named list of array digests), as we figured that most frequently the user will have the array details, and if not, they could look them up.

### Linked issues

- [https://github.com/ga4gh/seqcol-spec/issues/21](https://github.com/ga4gh/seqcol-spec/issues/21)
- [https://github.com/ga4gh/seqcol-spec/issues/23](https://github.com/ga4gh/seqcol-spec/issues/23)

## 2021-08-25 - Sequence collection digests will reflect sequence order

### Decision

The final sequence collection digests must reflect the order of the sequences given. In other words, changing the order of the sequences will change the identifier.

### Rationale

In some scenarios, the order of the sequences in a collection is irrelevant, and therefore, two collections with identical content but a different order should be considered equivalent. This could lead to an approach of first lexographically sorting input sequences before digesting, so that the final identifier is identical regardless of input order.

However, there are also scenarios for which the order of sequences in a collection matters; for example, some aligners output different results based on the input order of the sequences. Or, order may be used to encode sequence priority. Therefore, it is critical that the final identifiers be able to uniquely identify sequence collections with different orders. Because some use cases require order-aware digests, the final algorithm will have to accommodate this, and we will need to come up with another way to identify two collections that are identical in content but with different order, without relying on the digests being identical

### Linked issues

- [https://github.com/ga4gh/seqcol-spec/issues/5](https://github.com/ga4gh/seqcol-spec/issues/5)

### Known limitations

This decision will lead to a bit more complexity for use cases that do not care about sequence order, because these use cases will no longer be able to rely on simple, identical digest matching. However, this is a necessary increase in complexity to accommodate other use cases where order matters.


## 2021-06-30 - Use array-based data structure and multi-tiered digests

### Decision

Our original formulation structured data with groups of a sequence plus its  annotation, like `chr1|248956422|2648ae1bacce4ec4b6cf337dcae37816/chr2|242193529|4bb4f82880a14111eb7327169ffb729b|`. Instead, we decided to switch to an array-based model, in which a sequence collection will be constructed as a dictionary object, with attributes as named arrays, like this:

```
seqcol = {'names': ['chrUn_KI270742v1',   'chrUn_GL000216v2',   'chrUn_GL000218v1'],
  'lengths': ['186739', '176608', '161147'],
  'sequences': ['2f31c013a4a8301deb8ab7ed1ca1cd99',   '725009a7e3f5b78752b68afa922c090c',   
'1d708b54644c26c7e01c2dad5426d38c']}
```

Digests of this object will be the unique identifier of the sequence collection. To compute the digest of this object, we will use a multi-tiered digest approach, where each array will first be digested separately, and then the final identify will be a digest of digests. A sequence collection may thus be represented as a dictionary of digests:

```
{ 'sequences': '8dd93796fa0225e92eb159a8779f1b254776557f748f8bfb',
 'lengths': '501fd98e2fdcc276c47306bd72c9155489ed2b23123ddfa2',
 'names': '7bc90a07cf25f2f64f33baee3d420ad1ae5f442055280d43',
}
```

This will allow retrieving individual attributes, and testing for identity of individual attributes without retrieving more data.

### Rationale

- This makes it straightforward to mix-and-match components of the collection. Because each component is independent, and not integrated in with the sequence, it is simpler to select and build subsets and permutations.
- We can add a new component without eliminating backwards compatibility with previous digests that did not include it, because leaving out an array doesn't change the string to digest.
- This makes it easier to test for matching sets of sequences, or matching coordinate systems, using the individual component digests. This way we don't have to traverse down a layer deeper, to the individual elements, to establish identity of individual components.
- It requires only a single delimiter, rather than multiple, which has been a sticking point.

### Linked issues

- [https://github.com/ga4gh/seqcol-spec/issues/8#issuecomment-773489450](https://github.com/ga4gh/seqcol-spec/issues/8#issuecomment-773489450)
- [https://github.com/ga4gh/seqcol-spec/issues/10](https://github.com/ga4gh/seqcol-spec/issues/10)

### Known limitations

- We may need to enforce that arrays be the same length, at least for attributes that provide one value per sequence. Also, the order of items within each array must match in order for the attributes to correctly collate to a specific sequence.


## 2021-01-20 - Use the SAM specification v1 description of sequence names

### Decision

Sequence collections will use the [v1 SAM specification (subsection 1.2.1)](https://samtools.github.io/hts-specs/SAMv1.pdf#subsubsection.1.2.1) to define the allowable characters in a sequence name. As of January 2021, this is:

> Reference sequence names may contain any printable ASCII characters in the range[!-~]apart from backslashes, commas, quotation marks, and brackets—i.e., apart from ‘\ , "‘’ () [] {} <>’—and may notstart with ‘*’ or ‘=’.4

Should a wider GA4GH standard appear from [TASC issue 5](https://github.com/ga4gh/TASC/issues/5) the sequence collection spec will review this decision. The long-term vision of the sequence collections specification is to comply with any eventual GA4GH standard for sequence names.

### Linked issues

- [https://github.com/ga4gh/seqcol-spec/issues/2](https://github.com/ga4gh/seqcol-spec/issues/2)

### Known limitations

- There have been a handful of reports of old sequences with disallowed characters in the sequence name rows (`>`) of FASTA files, particularly from the microbiome community. These sequence collections would have to be changed to include only SAM-compatible ASCII characters, which could restrict usage of the sequence collections protocols and delay uptake.
