# Refget

Unique identifiers and lookup service for reference sequences and sequence collections.

<img src="img/seqcol_abstract_simple.svg" alt="Refget abstract" class="img-responsive">


## What is refget?


Refget is a protocol for identifying and distributing biological sequence references. It currently consists of 2 standards:

1. Refget sequences: a GA4GH-approved standard for individual sequences
2. Refget sequence collections: a standard for collections of sequences, under review 

## What is the refget sequences standard?

The original refget handled sequences only. Refget enables access to reference sequences using an identifier derived from the sequence itself.

## What is the refget sequence collections standard?

*Sequence Collections*, or `seqcol` for short, standardizes unique identifiers for collections of sequences. Seqcol identifiers can be used to identify genomes, transcriptomes, or proteomes -- anything that can be represented as a collection of sequences. The seqcol protocol provides:

- implementations of an algorithm for computing sequence identifiers;
- a lookup service to retrieve sequences given a seqcol identifier
- programmatic approach to assessing compatibility among sequence collections.

