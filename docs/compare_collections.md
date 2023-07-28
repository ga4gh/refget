
# How to: Compare two collections

## Use case

- You have a local sequence collection, and an identifier for a collection in a server. You want to compare the two to see if they have the same coordinate system.
- You have two identifiers for collections you know are stored by a server. You want to compare them.


## How to do it

You can use the `/comparison/:digest1/:digest2` endpoint to compare two collections. The comparison function gives information-rich feedback about the two collections, but it can take some thought to interpret. Here are some examples

### Strict identity

If you're looking to ensure that the two sequence collections are *strictly identical* -- that is, they have the same sequence content, with the same names, in the same order... then you actually don't need the `/comparison` endpoint; **two collections will have the same digest if they are identicial in content and order for all `inherent` attributes.** Therefore, if the digests differ, then you know the collections differ in at least one inherent attribute.

If you have a local sequence collection, and an identifier, then you can compare them for strict identity by computing the identifier for the local collection and seeing if they match.

### Order-relaxed identity

A process that treats each sequence independently and re-orders its results will return identical results as long as the sequence content and names are identical, even if the order doesn’t match. Therefore, you may be interested in saying "these two sequence collections have identical content and sequence names, but differ in order". The `/comparison` return value can answer this question:

Two collections meet the criteria for order-relaxed identity if:

1. the value of the `elements.total.a` and `elements.total.b` match, (the collections have the same number of elements).
2. this value is the same as `elements.a-and-b.<attribute>` for all attributes (the content is the same)
3. all entries in `elements.a-and-b-same-order.<attribute>` are false (the order differs for all attributes)

### Others...

There are many other types of compatibilty you can assess using the result of the `/comparison` function, which will be documented later.
