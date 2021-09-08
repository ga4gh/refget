# Architectural Decision Record

## 2021-08-25 - Sequence collection digests will reflect sequence order

### Decision

The final sequence collection digests must reflect the order of the sequences given. In other words, changing the order of the sequences will change the identifier.

### Rationale

In some scenarios, the order of the sequences in a collection is irrelevant, and therefore, two collections with identical content but a different order should be considered equivalent. This could lead to an approach of first lexographically sorting input sequences before digesting, so that the final identifier is identical regardless of input order.

However, there are also scenarios for which the order of sequences in a collection matters; for example, some aligners output different results based on the input order of the sequences. Or, order may be used to encode sequence priority. Therefore, it is critical that the final identifiers be able to uniquely identify sequence collections with different orders. Because some use cases require order-aware digests, the final algorithm will have to accommodate this, and we will need to come up with another way to identify two collections that are identical in content but with different order, without relying on the digests being identical

### Linked issues

- https://github.com/ga4gh/seqcol-spec/issues/5

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

- https://github.com/ga4gh/seqcol-spec/issues/2

### Known limitations

- There have been a handful of reports of old sequences with disallowed characters in the sequence name rows (`>`) of FASTA files, particularly from the microbiome community. These sequence collections would have to be changed to include only SAM-compatible ASCII characters, which could restrict usage of the sequence collections protocols and delay uptake.
