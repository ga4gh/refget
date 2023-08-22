
# How to: Retrieve a collection given a digest 

## Use case

You have a seqcol digest, and you'd like to retrieve the underlying sequence identifiers, or sequences themselves.

## How to do it

To look up the contents of a digest will require a seqcol service that stores the collection in a database.

### 1. Retriving the sequence identifiers

You can retrieve the canonical seqcol representation by hitting the `/collection/:digest` endpoint, where `:digest` should be changed to the digest in question. If all you need is sequence identifiers, then you're done.


### 2. Retrieving underlying sequences

If you need sequences, then you'll also need a refget server. Sequence collection services don't necessarily store sequences themselves; this task is typically outsource to a refget server. The seqcol server simply stores the group information, and metadata accompanying the sequences. Therefore, to retrieve the underlying sequences, you can first retrieve the sequence identifiers, and then use these identifiers to query a refget service.
