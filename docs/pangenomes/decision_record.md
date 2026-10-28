# Architectural Decision Record for refget pangenomes

*This is a record of decisions made during specification development. Each entry describes a decision that has been approved by the team members. Collectively, this ADR describes an institutional memory for decisions and their rationales, including known limitations. The goal is to avoid repeated discussion of previous decisions, formally acknowledge limitations, preserve and articulate reasons behind the decisions, and share this information with the broader community.* 

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

## Contents

[TOC]

## 2026-10-28 The pangenome graph topology identifier must include node identifiers

### Decision

The identifier describing the topology of a pangenome sequence graph MUST be computed from information about the graph's nodes, including each node's `node_id`, in addition to the edges connecting them.
Two graphs with identical sequences and edges, but different node identifiers, will therefore have different topology identifiers.

The data structure used to encode the graph topology (and thus the exact content that is digested) is not yet defined and will be the subject of a later decision.

### Rationale

In a pangenome sequence graph, a single residue is located by a pair of coordinates: the `node_id` and the offset within that node's sequence.
The `node_id` is therefore an inherent component of the coordinate space of the graph, in the same way that a sequence name is part of the coordinate system of a linear reference.

If node identifiers change, any coordinate expressed against the graph (e.g. a variant, an alignment, or an annotation) no longer points to the same residue, even if the underlying sequences and edges are unchanged.
In other words, renumbering the nodes produces a different coordinate space.
The topology identifier is intended to unambiguously identify that coordinate space, so it must change whenever the coordinate space changes, which requires the node identifiers to be part of the digested content.

### Known limitations

- Node identifiers can be modified without changing any of the underlying sequences or the structure of the graph. Two graphs that are essentially the same (same nodes, same sequences and same edges) but that use different node identifiers will receive different topology identifiers. Determining that two such graphs are structurally equivalent will require a separate comparison (e.g. an identifier computed without node identifiers), which is outside the scope of this decision.
