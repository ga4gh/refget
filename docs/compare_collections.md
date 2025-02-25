# How to: Compare two collections

## Use case

- You have a local sequence collection, and a digest for a collection in a server. You want to compare the two to see if they have the same coordinate system.
- You have two digests for collections you know are stored by a server. You want to compare them.

## How to do it

You can use the `/comparison/:digest1/:digest2` endpoint to compare two collections.
The comparison function gives information-rich feedback about the two collections, but it can take some thought to interpret. Here are some examples

### Strict identity

Some analyses may require that the collections be *strictly identical* -- that is, they have the same sequence content, with the same names, in the same order.
For example, aligning against sequence collections that differ in any aspect (sequence name, order difference, etc) with bowtie2 will not necessarily produce the same output.
Therefore, we must be able to identify that two sequence collections are identical in terms of sequence content, sequence name, and sequence order. 

This comparison can easily be done by simply comparing the seqcol digest, you don't need the `/comparison` endpoint.
**Two collections will have the same digest if they are identical in content and order for all `inherent` attributes.**
Therefore, if the digests differ, then you know the collections differ in at least one inherent attribute.
If you have a local sequence collection and a digest, then you can compare them for strict identity by computing the digest for the local collection and seeing if they match.

### Order-relaxed identity

A process that treats each sequence independently and re-orders its results will return identical results as long as the sequence content and names are identical, even if the order doesn’t match. Therefore, you may be interested in saying "these two sequence collections have identical content and sequence names, but differ in order". The `/comparison` return value can answer this question:

Two collections meet the criteria for order-relaxed identity if:

1. the value of the `elements.total.a` and `elements.total.b` match, (the collections have the same number of elements).
2. this value is the same as `elements.a_and_b.<attribute>` for all attributes (the content is the same)
3. entries in `elements.a_and_b-same-order.<attribute>` may be either all true (indicating the order matches) or all false (indicating the order differs)

Then, we know the sequence collection content is identical, without controlling for order. 

###### Name-relaxed identity

Some analysis (for example, a [`salmon` RNA pseudo-alignment](https://salmon.readthedocs.io/en/latest/salmon.html)) will be identical regardless of the chromosome names, as it considers the digest of the sequence only.
Thus, we'd like to be able to say "These sequence collections have identical content, even if their names and/or orders differ."

How to assess: As long as the `a_and_b` number for `sequences` equals the values listed in `elements.total`, then the sequence content in the two collections is identical

###### Length-only compatible (shared coordinate system)

A much weaker type of compatibility is two sequence collections that have the same set of lengths, though the sequences themselves may differ.
In this case we may or may not require name identity. For example, a set of ATAC-seq peaks that are annotated on a particular genome could be used in a separate process that had been aligned to a different genome, with different sequences -- as long as the lengths and names were shared between the two analyses.

How to assess: We will ignore the `sequences` attribute, but ensure that the `names` and `lengths` numbers for `a_and_b` match what we expect from `elements.total`.
If the `a_and_b-same-order` is also true for both `names` and `lengths`, then we can be assured that the two collections share an ordered coordinate system.
If however, their coordinate system matches but is not in the same order, then we require looking at the `sorted_name_length_pairs` attribute. If the `a_and_b` entry for `sorted_name_length_pairs` is the same as the number for `names` and `lengths`, then these collections share an (unordered) coordinate system.

### Others...

There are also probably other types of compatibility you can assess using the result of the `/comparison` function.
Now that you know the basics, and once you have an understanding of what the comparison function results mean, it should be possible to figure out if you can assess a particular type of compatibility for your use case.

## Limitation of the comparison function: distinguishing out-of-order from mismatched arrays

One limitation of the comparison function is that it does comparisons at the level of arrays, not at the level of individual elements. What would the comparison function return for two sequence collections that have the same content, but in different orders, AND where in addition two of the sequences have swapped names?

Because the sequence array would contain the same sequences, the comparison function will count them all as matching.
Similarly, the names arrays contain the same names and so all will be counted as a match.
However, the same_order will *not* be true; it will yield false for all attributes.

This is the same output as a comparison of two sequence collections in different orders, without the name swap. This is a fundamental limitation of the array-based method of comparing.

In this particular example, these results can be distinguished by the `sorted_name_length_pairs` attribute, because this would yield a perfect match for the second example, where all the pairs are intact but in a different order -- but it would NOT yield a match for the example with swapped names, because the name-length pairs would be different.

This solves the issue for swapped names, but there is still potential for problems with other arrays or custom attributes. Therefore, we warn users that when the `_same_order` is flagged as false, this *does not imply that the pairs are intact*, and if this is a requirement, further investigation would be necessary. If distinguishing these scenarios is important, one possible solution would be to add another non-inherent collated attribute, similar to `sorted_name_length_pairs`, but including *all* collated attributes for each element rather than just the names and lengths. The comparison function would then immediately provide an answer as to whether the annotated sequence elements match *as units* between two collections.

