# Architectural Decision Record

## 2021-01-20 - Use the SAM specification v1 description of sequence names

### Decision

Sequence collections will use the [v1 SAM specification (subsection 1.2.1)](https://samtools.github.io/hts-specs/SAMv1.pdf#subsubsection.1.2.1) to define the allowable characters in a sequence name. As of January 2021, this is:

> Reference sequence names may contain any printable ASCII characters in the range[!-~]apart from backslashes, commas, quotation marks, and brackets—i.e., apart from ‘\ , "‘’ () [] {} <>’—and may notstart with ‘*’ or ‘=’.4

Should a wider GA4GH standard appear from [TASC issue 5](https://github.com/ga4gh/TASC/issues/5) the sequence collection spec will review this decision. The long-term vision of the sequence collections specification is to comply with any eventual GA4GH standard for sequence names.

### Linked issues

- https://github.com/ga4gh/seqcol-spec/issues/2

### Known limitations

- There have been a handful of reports of old sequences with disallowed characters in the sequence name rows (`>`) of FASTA files, particularly from the microbiome community. These sequence collections would have to be changed to include only SAM-compatible ASCII characters, which could restrict usage of the sequence collections protocols and delay uptake.

## 2021-06-16 - Controlled vocabulary for the supported fields

### Decision

The attributes allowed to be used in the sequence collection digest construction are: 
 - lengths: the length of the sequences
 - sequences: the Refget digests of the sequences
 - names: the given name of the sequence in this collection 
 - topologies: the topology of each sequence (can be either linear or circular)
 - priorities: importance of the sequence in the assembly
 - order: the original order in which the sequence were defined (list of position)

Additional optional fields can be added to extend the model as required but the only mandatory fields are lengths and order

### Linked issues

- https://github.com/ga4gh/seqcol-spec/issues/8

### Known limitations
