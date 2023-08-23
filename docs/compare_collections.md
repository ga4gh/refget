
# How to: Compare two collections

## Use case

- You have a local sequence collection, and an identifier for a collection in a server. You want to compare the two to see if they have the same coordinate system.
- You have two identifiers for collections you know are stored by a server. You want to compare them.

## How to do it

You can use the `/comparison/:digest1/:digest2` endpoint to compare two collections. The comparison function gives information-rich feedback about the two collections, but it can take some thought to interpret. Here are some examples

### Strict identity

Some analyses may require that the collections be *strictly identical* -- that is, they have the same sequence content, with the same names, in the same order. For example, a bowtie2 index produced from one sequence collection that differs in any aspect (sequence name, order difference, etc), will not necessarily produce the same output. Therefore, we must be able to identify that two sequence collections are identical in terms of sequence content, sequence name, and sequence order. 

 This comparison can easily be done by simply comparing the seqcol digest, you don't need the `/comparison` endpoint. **Two collections will have the same digest if they are identicial in content and order for all `inherent` attributes.** Therefore, if the digests differ, then you know the collections differ in at least one inherent attribute. If you have a local sequence collection, and an identifier, then you can compare them for strict identity by computing the identifier for the local collection and seeing if they match.

### Order-relaxed identity

A process that treats each sequence independently and re-orders its results will return identical results as long as the sequence content and names are identical, even if the order doesn’t match. Therefore, you may be interested in saying "these two sequence collections have identical content and sequence names, but differ in order". The `/comparison` return value can answer this question:

Two collections meet the criteria for order-relaxed identity if:

1. the value of the `elements.total.a` and `elements.total.b` match, (the collections have the same number of elements).
2. this value is the same as `elements.a-and-b.<attribute>` for all attributes (the content is the same)
3. any entries in `elements.a-and-b-same-order.<attribute>` may be true (indicating the order matches) or false (indicating the order differs)

Then, we know the sequence collection content is identical, without controlling for order. 

###### Name-relaxed identity

Some analysis (for example, a [`salmon` RNA pseudo-alignment](https://salmon.readthedocs.io/en/latest/salmon.html)) will be identical regardless of the chromosome names, as it considers the digest of the sequence only. Thus, we'd like to be able to say "These sequence collections have identical content, even if their names and/or orders differ."

How to assess: As long as the `a-and-b` number for `sequences` equals the values listed in `elements.total`, then the sequence content in the two collections is identical

###### Length-only compatible (shared coordinate system)

A much weaker type of compatibility is two sequence collections that have the same set of lengths, though the sequences themselves may differ. In this case we may or may not require name identity. For example, a set of ATAC-seq peaks that are annotated on a particular genome could be used in a separate process that had been aligned to a different genome, with different sequences -- as long as the lengths and names were shared between the two analyses.

How to assess: We will ignore the `sequences` attribute, but ensure that the `names` and `lengths` numbers for `a-and-b` match what we expect from `elements.total`. If the `a-and-b-same-order` is also true for both `names` and `lengths`, then we can be assured that the two collections share an ordered coordinate system. If however, their coordinate system matches but is not in the same order, then we require looking at the `sorted_name_length_pairs` attribute. If the `a-and-b` entry for `sorted_name_length_pairs` is the same as the number for `names` and `lengths`, then these collections share an (unordered) coordinate system.

### Others...

There are also probably other types of compatibility you can assess using the result of the `/comparison` function. Now that you know the basics, and once you have an understanding of what the comparison function results mean, it should be possible to figure out if you can assess a particular type of compatibility for your use case.
