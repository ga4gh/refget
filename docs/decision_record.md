# Architectural Decision Record

*This is a record of decisions made during specification development. Each entry describes a decision that has been approved by the team members. Collectively, this ADR describes an institutional memory for decisions and their rationales, including known limitations. The goal is to avoid repeated discussion of previous decisions, formally acknowledge limitations, preserve and articulate reasons behind the decisions, and share this information with the broader community.* 

## Contents: 

[TOC]

## 2022-05-11 - Sequence digest recommendation

### Decision

The GA4GH digest will be used as our default sequence checksum derived identifier instead of an MD5 generated one.

### Rationale 

The GA4GH digest was created as part of the [Variation Representation Specification standard](https://vrs.ga4gh.org/en/stable/impl-guide/computed_identifiers.html). This document included a way of creating identifiers to be used with sequences e.g. ACGT results in the identifier `ga4gh:SQ.aKF498dAxcJAqme6QYQ7EZ07-fiw8Kw2`. The scheme uses the [`sha512t24u` function](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0239883) to create a base64 URL-safe representation of a sha512 digest. Adopting this standard ensures sequence collections remains inline with newer standards within the GA4GH ecosystem.

### Limitations

A GA4GH digest of sequences is not the default identifier used by standards such as CRAM, which uses MD5. We expect sequence collection providers to offer additional checksum arrays to provide compatability with these other formats and to declare their sequence identifier support via service-info.

### Linked issues

- [https://github.com/ga4gh/seqcol-spec/issues/30](https://github.com/ga4gh/seqcol-spec/issues/30)

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
