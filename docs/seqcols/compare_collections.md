# How to: Compare two collections

## Use case

- You have two digests for collections you know are stored by a server. You want to compare them.
- You have a digest for a collection from a server, and a local sequence. You want to compare the two to see if they have the same coordinate system.

## How to do it

You can use the `/comparison/:digest1/:digest2` endpoint to compare two collections.
You can also `POST` a local collection to `/comparison/:digest1` to compare it to a single remote collection.
The comparison function gives information-rich feedback about the two collections, but it can take some thought to interpret.
Here are some examples.

The best way is to use the Refget [Seqcol Comparison Interpretation Module (SCIM)](https://refget.databio.org/scim/).
You paste in the JSON output of a comparison, and it provides an interpretation for you.

## Interpretation details

### Strict identity

Some analyses may require that the collections be *strictly identical* -- that is, they have the same sequence content, with the same names, in the same order.
For example, aligning with bowtie2 against sequence collections that differ in either content, name, or order will not necessarily produce the same output.
Therefore, we must be able to identify that two sequence collections are identical in terms of sequence content, sequence name, and sequence order. 

For this simple comparison, you don't need the `/comparison` endpoint -- just compare the top-level digests.
**Two collections will have the same digest if they are identical in content, names, and order for all `inherent` attributes.**
Therefore, if the digests differ, then you know the collections differ in at least one inherent attribute.

### Order-relaxed identity

A process that treats each sequence independently and re-orders its results will return identical results as long as the sequence content and names are identical, even if the order doesn’t match. Therefore, you may be interested in saying "these two sequence collections have identical sequence names and content, but differ in order".
Relying on top-level digests will not work for this comparison, but you can answer this question using `/comparison` return value:

Two collections meet the criteria for order-relaxed identity if:

1. the value of the `elements.total.a` and `elements.total.b` match, (the collections have the same number of elements).
2. this value is the same as `elements.a_and_b.<attribute>` for all attributes (the content is the same)
3. any entries in `elements.a_and_b-same-order.<attribute>` may be true (indicating the order matches) or false (indicating the order differs)

Then, we know the sequence content and names are identical, but not in the same order. 

###### Name-relaxed identity

Some analysis (for example, a [`salmon` RNA pseudo-alignment](https://salmon.readthedocs.io/en/latest/salmon.html)) will be identical regardless of the chromosome names, as it considers the digest of the sequence only.
Thus, we'd like to be able to say "These sequence collections have identical content, even if their names and/or orders differ."

There are two convenient ways to answer this question.
First, you can use the attribute (level1) digest, for the `sorted_sequences` attribute.
If this digest matches, then you know you have identical sequence content, without controlling for names or sequence order.

Second, you can also answer this question using the `/comparison` function. As long as the `a_and_b` number for `sequences` equals the values listed in `elements.total`, then the sequence content in the two collections is identical.

###### Length-only compatible (shared coordinate system)

A much looser type of compatibility is two sequence collections that have the same set of sequence lengths, though the sequences themselves may differ.
In this case we may or may not require name identity. For example, a set of ATAC-seq peaks that are annotated on a particular genome could be used in a separate process that had been aligned to a different genome, with different sequences -- as long as the lengths and names were shared between the two analyses.

How to assess: We will ignore the `sequences` attribute, but ensure that the `names` and `lengths` numbers for `a_and_b` match what we expect from `elements.total`.
If the `a_and_b-same-order` is also true for both `names` and `lengths`, then we can be assured that the two collections share an ordered coordinate system.
If however, their coordinate system matches but is not in the same order, then we require looking at the `sorted_name_length_pairs` attribute. If the `a_and_b` entry for `sorted_name_length_pairs` is the same as the number for `names` and `lengths`, then these collections share an (unordered) coordinate system.

## Complex cases

For more complex cases, the comparison function and the level1 digests can sometimes be used to figure out what is going on, but they are limited by design -- for situations that are more complex than these methods can handle, it is always possible to look deeper at the contents of the sequence collection and compare them directly. 

The `/comparison` endpoint only tests the order of each array attribute independently. There is no general test of order consistency across several array attributes, *e.g.* whether a single set of collated values for names, lengths, and sequences retains the same index across all three arrays if reordered. A concrete example for interpreting such a case will be added later.