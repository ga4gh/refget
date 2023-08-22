

## Operations enabled by seqcol, organized by input

### Seqcol digest as input

* **seqcol digest -> sequence digests**: Given a seqcol digest, retrieve sequence digests for all contained sequences with the `/collection/:digest/:level` endpoint by setting the `level` to 1, or omitting it.
* **seqcol digest -> sequences**: Given a seqcol digest, sequences themselves for all contained sequences can be retrieved by the `/collection/:digest/:level` endpoint by setting the `level` to 2 (if this is allowed by the server). Or, you can use the sequence digests retrieve at `level=1` to look up actual sequences using a refget server.
* **seqcol digest -> metadata**: Any metadata known by the server will be retrieved by using `/collection/:digest`.
* **seqcol digest -> aliases of seqcol**: Aliases are a not a built-in part of the seqcol spec, so this capability will depend on the underlying provider. If the provider provides aliases, you can retrieve them using `/collection/:digest`.
* **2 seqcol digests -> assessment of compatibility**: Provided by the `/comparison` endpoint.
* **seqcol digest + sequences -> validation claim**: Use the *seqcol algorithm* to compute the digest of the set of sequences, and then use the *comarison* function to validate (or, if you validation requires strict identity, just confirm that the digests match).


### Sequence collection as input

* **sequences -> refget digests**: Individual sequences can be converted into refget digests using the refget algorithm
* **sequences -> seqcol digest**: Sequence collections can be converted into seqcol digests using the seqcol algorithm. To obtain a sequence digest, follow the algorithm outlined above, sequences -> refget digests -> seqcol digest.

### Refget digest as input

* **refget digest -> seqcol digests containing this sequence**: Use the `containing_collections` secondary endpoint. 
* **many refget digests -> seqcol digests containing all of these sequences**: Repeated application of `containing_collections` endpoint?

## Coordinate system as input

* **coordinate system -> seqcol digest**: If you have only a coordinate system (lengths with no sequences), you may still compute a seqcol digest using the seqcol algorithm, because the sequences themselves are optional. Coordinate systems can therefore be uniquely encoded, retrieved, and compared using the seqcol protocol.


## Related operations: What seqcol does *not* do

* **sequences -> sequence names**: This is the refget reverse lookup.
* **alias -> seqcol digest**: Will the API offer this?
* **seqcol digest -> other seqcol digests compatible at some level**: The database does not store comparison flags, so if you're searching for compatible collections you'd need to retrieve and compute the compatible yourself using the comparison function.

