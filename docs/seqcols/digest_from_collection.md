
# How to: Compute a seqcol digest given a sequence collection

## Use case

One of the most common uses of the seqcol specification is to compute a standard, universal digest for a particular sequence collection. You have a collection of sequences, like a reference genome or transcriptome, and you want to determine its seqcol digest. There are two ways to approach this: 1. Using an existing implementation; 2. Implement the seqcol digest algorithm yourself (it's not that hard).


## 1. Using existing implementations

- The `refget` Python package provides an implementation for local computation of digests: <https://refgenie.org/refget/>.

## 2. Implement the seqcol digest algorithm yourself

Follow the procedure under the section for [Encoding](README.md#2-encoding-computing-sequence-digests-from-sequence-collections). Briefly, the steps are:

- **Step 1**. Organize the sequence collection data into *canonical seqcol object representation*.
- **Step 2**. Apply [RFC-8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785) (JCS) to canonicalize the value associated with each attribute individually.
- **Step 3**. Digest each canonicalized attribute value using the GA4GH digest algorithm.
- **Step 4**. Apply [RFC-8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785) again to canonicalize the JSON of new seqcol object representation.
- **Step 5**. Digest the final canonical representation again.

Details on each step can be found in the specification.


### Example Python code for computing a seqcol encoding

```python
# Demo for encoding a sequence collection

import base64
import hashlib
import json

def canonical_str(item: dict) -> bytes:
    """Convert a dict into a canonical string representation"""
    return json.dumps(
        item, separators=(",", ":"), ensure_ascii=False, allow_nan=False, sort_keys=True
    ).encode("utf8")

def sha512t24u_digest(seq: bytes) -> str:
    """ GA4GH digest function """
    offset = 24
    digest = hashlib.sha512(seq).digest()
    tdigest_b64us = base64.urlsafe_b64encode(digest[:offset])
    return tdigest_b64us.decode("ascii")

# 1. Get data as canonical seqcol object representation

seqcol_obj = {
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
    "SQ.2648ae1bacce4ec4b6cf337dcae37816",
    "SQ.907112d17fcb73bcab1ed1c72b97ce68",
    "SQ.1511375dc2dd1b633af8cf439ae90cec"
  ]
}

# Step 1a: Remove any non-inherent attributes,
# so that only the inherent attributes contribute to the digest.
# Only names and sequences are inherent.

seqcol_obj_inherent = {
  "names": seqcol_obj["names"],
  "sequences": seqcol_obj["sequences"]
}

# Step 2: Apply RFC-8785 to canonicalize the value 
# associated with each attribute individually.

seqcol_obj2 = {}
for attribute in seqcol_obj_inherent:
    seqcol_obj2[attribute] = canonical_str(seqcol_obj_inherent[attribute])
seqcol_obj2  # visualize the result

# Step 3: Digest each canonicalized attribute value
# using the GA4GH digest algorithm.

seqcol_obj3 = {}
for attribute in seqcol_obj2:
    seqcol_obj3[attribute] = sha512t24u_digest(seqcol_obj2[attribute])
print(json.dumps(seqcol_obj3, indent=2))  # visualize the result

# Step 4: Apply RFC-8785 again to canonicalize the JSON 
# of new seqcol object representation.

seqcol_obj4 = canonical_str(seqcol_obj3)
seqcol_obj4  # visualize the result

# Step 5: Digest the final canonical representation again.

seqcol_digest = sha512t24u_digest(seqcol_obj4)
# Answer: 'KxZO6qIbVNCIKtQj0WR3fwzg2rsJLlC3'

# Equivalent to:
# import refget
# sc = refget.SequenceCollection.from_dict(seqcol_obj)
# sc.digest
```
