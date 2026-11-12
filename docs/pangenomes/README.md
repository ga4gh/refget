---
icon: material/traffic-cone
---

# Refget Pangenomes

!!! warning "Under development"
    This specification is under development. Terms and decisions on this page may change.

The refget pangenome specification will define identifiers for pangenome graphs, built in the same way as refget sequence and sequence collection identifiers.

## Definitions of key terms

- **Node**: a point in the graph that edges connect to and paths pass through. Each node has an ID and carries one sequence.
- **Node ID**: the name or number that identifies a node. GFA calls it the segment name, and it can be any text; vg, odgi and GBZ use integers. Tools that cut long nodes into shorter ones ("chopping") give out new node IDs.
- **Node sequence**: the characters (e.g. DNA) a node holds. Graph papers call it the node's label. Many nodes can carry the same sequence (millions of nodes may be just "A"), so the sequence alone can't tell nodes apart; the node ID does.
- **Graph coordinate**: how you point to one base in a graph. There may be many ways to do this; one simple example is a node ID plus an offset into that node's DNA. It plays the role of "chromosome + position" in a linear genome. For this system, if the node IDs change, every coordinate changes, even when the DNA and links stay the same.
- **Edge**: a link that joins one end (front or back) of a node to one end of another node. A graph can have edges that no genome's path uses.
- **Path**: an ordered walk through the nodes, each crossed in a set direction. Put another way, an ordered list of nodes, each read forward or reversed. Joining the DNA along the path spells out a genome's sequence, usually one path per chromosome.
- **Seqgraph**: the nodes (IDs and DNA) plus the edges, with no paths. It is the shared scaffold that paths walk through. One seqgraph covers the whole pangenome, all chromosomes, because edges between chromosomes can exist.
- **Seqgraph identifier**: a digest computed from the node IDs, the node DNA and the edges, with no paths. [pggname](https://github.com/jltsiren/pggname) is the same idea. Renaming nodes changes it.
- **Pangenome graph**: a seqgraph plus every genome's paths. It holds everything needed to spell out each genome in the pangenome.
- **Subgraph**: a part of a seqgraph that keeps only some of its nodes and edges. It reuses the parent's node IDs, so a coordinate in the subgraph also works in the full graph.
- **Pangenome graph identifier**: a digest computed from the seqgraph plus all the paths. It changes if any genome's path changes, even when the seqgraph stays the same.

## Key concepts

### Pangenome directionality

Pangenome graphs are "bidirected": an edge has no direction and can be crossed either way, but each of its two ends attaches to a specific side (front/left or back/right) of its node.
This differs from a directed graph, where edges are one-way arrows, and from an undirected graph, where edges join nodes without regard to sides.

### Node identity vs. attached content

A node in a pangenome graph has two separate properties.
Its identity is its node ID.
The ID is unique, it tells this node apart from every other node, and it is what graph coordinates point to.
Its attached content is its node sequence.
The sequence is the DNA the node carries, and it need not be unique: many nodes can carry the same sequence.

Graph theory uses the word "label" for both properties.
A "labeled graph" or "canonical labeling" refers to identity, while a node label ℓ(v) in sequence-graph papers refers to content.
So we avoid "label" and say node ID for identity and node sequence for content.

Both are part of the graph and both count toward the seqgraph identifier, but they do different jobs.
Changing a node ID changes where coordinates point without changing any DNA.
Changing a node sequence changes the DNA without changing any coordinate's node.

### Paths and edges

In a sequence graph (seqgraph), edges record which nodes connect, the set of all possible steps.
A path is one walk through the graph, written as an ordered list of nodes, each read forward or reversed.
The path doesn't list its edges, because each pair of neighboring nodes in the list implies the edge between them.
So a path is valid only if every step it takes matches an edge in the graph.

The reverse doesn't hold: a graph can have edges that no path uses, so you can't rebuild the edges from the paths.
Edges belong to the graph; paths are separate walks that rely on them.
