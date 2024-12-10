# Architectural Decision Record

*This is a record of decisions made during specification development. Each entry describes a decision that has been approved by the team members. Collectively, this ADR describes an institutional memory for decisions and their rationales, including known limitations. The goal is to avoid repeated discussion of previous decisions, formally acknowledge limitations, preserve and articulate reasons behind the decisions, and share this information with the broader community.* 

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

## Contents

[TOC]

## 2024-11-20 Level 2 return values should not return transient attributes

### Decision

Level 2 return values should not return transient attributes

### Rationale

We debated whether the `/collection?level=2` endpoint should do with transient attributes, because the level 2 representations are not stored. One train of thought was that it could return the level 1 representation; other is that it just includes nothing. We decided that the more pure approach would be include neither

Another option was something like `?level=highest`, which would return level 2 representations for everything that has one, but level 1 representations for transient attributes.

We decided that even if you don't have that information, you could just get it from the `?level=1` endpoint. Or, implementations could specify their own way


## 2024-11-20 Custom modifiers should live in the schema under the `ga4gh` key

### Decision

Any global custom modifiers should live under a `ga4gh` key in  the schemea. Right now, this includes `inherent`, `transient`, and `passthru`.
Local modifiers (currently just `collated`) will continue to live, raw, under the attribute they describe.


### Rationale

We want to follow the standard used in the other specs (VRS), and it also seems fine to have a place to lump together our custom modifiers.
We thought we could also do this for `collated`, as a local modifier, but opt not to right now because: there's only 1, it's a boolean, and it's not actually even used for anything in the spec at the moment, it is only there because it could be nice to use for a visualization of elements in a collection. The additional complexity of another layer just for this seems pointless at this point.

### Linked issues

- <https://github.com/ga4gh/refget/issues/84>

## 2024-11-13 Attributes can be designed as `passthru` or `transient`.

### Decision

We add two new attribute qualifiers: transient and passthru.

- Passthru attributes are not digested in transition from level 2 to level 1. Most attributes of the canonical (level 2) seqcol representation are digested to create the level 1 representation. But sometimes, we have an attribute for which digesting makes little sense. These attributes are passed through the transformation, so they show up on the level 1 representation in the same form as the level 2 representation. Thus, we refer to them as passthru attributes.
Transient attributes

- Transient attributes are not retrievable from the attribute endpoint. Most attributes of the sequence collection can be retrieved through the /attribute endpoint. However, some attributes may not be retrievable. For example, this could happen for an attribute that we intend to be used primarily as an identifier. In this case, we don't necessarily want to store the original content that went into the digest into the database, because it might be redundant. We really just want the final attribute. These attributes are called transient because the content of the attribute is no longer stored and is therefore no longer retrievable.

Also, a few other related decisions we finalized:
- `collection` endpoint, level 2 collection representation should exclude transient attributes.
- `attribute` endpoint wouldn't provide anything for either transient or passthru attributes.
- Can passthru or transient attributes be inherent? They could, but it probably doesn't really make sense. Nevertheless, there's no reason to state that they cannot be.

### Rationale

As we worked on more advanced attributes, and with the addition of the `/attribute` endpoint, we realized these changes necessitate a bit more power for the schema to specify behavior of the attributes. For the basic seqcol attributes (names, lengths, sequences) and original endpoint, the general algorithm and basic qualifiers (required, inherent, collated) suffice to describe the representation. But some more nuanced attributes require additional qualifiers to describe their intention and how the server should be behave for the `/attribute` endpoint. For example, sorted_name_length_pairs and sorted_sequences are intended to provide alternative tailored identifiers and comparisons, and not necessarily useful for independent attribute lookup. Similarly, custom extra attributes, like author or alias, may be simple appendages that don't need the complex digesting procedure we use for the basic attributes. In order to flag such attributes in a way that can govern slightly different server expectations, we need a couple of additional advanced attribute qualifiers. For this purpose, we added the passthru and transient qualifiers.

### Linked issues

- <https://github.com/ga4gh/refget/issues/86>


## 2024-10-02 Minimal schema should now require sequences, and lengths should not be inherent.

### Decision

We will update the minimal schema with these changes: 1. Move sequences into 'required', and 2. remove lengths from 'inherent'. So the final qualifiers would be:
- required: names, lengths, and sequences
- inherent: names, sequences


### Rationale

Originally, there was a good rationale for making sequences not required, to allow for coordinate systems to be represented as a seqcol.
But with the new `/attribute` endpoint, there's a better way to handle it, using `name_length_pairs` and `sorted_name_length_pairs` attributes.
Then, with sequences required, it does not make sense for lengths to be inherent because they are computable from sequences.
So essentially, the attribute endpoint allows us to move away from handling coordinate systems as top-level entities, and instead moves toward using the attribute endpoint for coordinate systems.

### Linked issues

- <https://github.com/ga4gh/refget/issues/72>

## 2024-10-02 The `/collection` and `/attribute` endpoints will both be `REQUIRED`

### Decision

The `/collection` and `/attribute` endpoints will both be `REQUIRED`

### Rationale

We debated whether one or both of these should drop to `RECOMMENDED`, because now we can imagine a lot of use cases that would use one but not the other. But in the end, the interoperability really needs the `/collection` endpoint, and a lot of use cases will rely on the `/attribute` endpoint, so we decided to just leave them both as `REQUIRED` to reflect their dual imporantance in an interoperable eco-system. This does not stop individual implementations from doing partial implementations, like "We only implement the `/attribute` endpoint", if that's all they need; it simply would prevent them from claiming that they are in full compliance of the spec; they'd just have a partial implementation, which is fine. Those services would have some level of interoperability, but would not rise to the level needed to do some of the meta-aggregation we can imagine, so we feel it's appropriate for them to only claim partial compliance.

## 2024-10-02 The `object_type` should be singular all the time

### Decision

The endpoints that can be moduled by `object_type`, `/list/:object_type` and `/attribute/:object_type`, should always use the singular form of the object_type.

### Rationale

It's easier if we have this be uniform, instead of having `/list/collections` and then `/attribute/collection` and then defining these both as `object_type`; to be strictly accurate here we'd need to define a second variables, like `plural_object_type`, so that the spec would be internally consistent. Instead, we don't really see a disadvantage to just making `object_type` have a consistent definition, so that it can be reused throughout the spec. So the end point should change to `/list/collection`.


## 2024-10-02 We should use query parameters for the filtered list endpoint

### Decision

The filtered list endpoint should filter by adding query parameters to the unfiltered endpoint, like `/list/:object_type?:attribute1=:attribute_digest1&attribute2=:attribute_digest2`.

### Rationale

Originally, we had defined two path-based variants of the list endpoint; unfiltered as `/list/collection` and filtered as `/list/collection/:attribute/:attribute_digest`. We realized this has some disadvantages; first, it requires us to define these as two separate endpoints, and second, it makes it so you can't enable filtering by more than one attribute digest. We didn't really see a disadvantage to just switching to optional query parameters, and we see several advantages. Now everything fits nicely under a single endpoint definition, and it's natural that without a filter parameter, you simply give the unfiltered result, but with the filter parameter, you give the filtered result. Furthermore, it sets the stage for multiple values, if this could be useful.


## 2024-08-08 The specification should require the `/attribute` endpoint

### Decision

We decided to add to the specification the `/attribute` endpoint, which would retrieve values of given collection attributes given attribute digests. This is parallel to the `/collection` endpoint, which retrieves the whole collection given a top-level digest.

### Rationale

Several use cases have re-emphasized the value of the digests of the `sorted_sequences`, the `sorted_name_length_pairs`, and other collection attributes. For many use cases, these are really the most important piece of information. However, until now, we've pushed for dealing with top-level collections, and using these as information contained within a collection. This has driven people to want to create separate schemas, which will hurt long-term interoperability. Instead, we realized that if we elevated the status of the *attributes*, such that users could retrieve these values directly, then that would allow these use cases to live within the ecosystem without needing to specify separate schemas. This change will therefore allow us to preserve some interoperability.

### Linked issues

- <https://github.com/ga4gh/refget/issues/80>
- <https://github.com/ga4gh/refget/issues/77>


## 2024-08-08 The `/list` endpoint will provide global and filtered listing of collections

### Decision

We decided to include the `/list` endpoint in two variants, a global one that just lists all available collections, and a filtered one, that allows users to list any collections that have a certain attribute. It should be `/list/collections`, in anticipation of future endpoints that could list entities of other types (like pangenomes or attributes)

### Rationale

We had been brainstorming about listing and filtered listing endpoints for several years, and it was always on the roadmap. We could think of clear use cases. For example, it would be necessary for a meta-service that would aggregate across sequence collections as a way to discover what is contained in one. We had also for along time debated a discovery endpoint that would allow searching through sequence collections. We were originally going to postpone this to v1.1, but in recent months it's become clear that these features are really important to drive uptake of the standard.

### Linked issues

- <https://github.com/ga4gh/refget/issues/61>
- <https://github.com/ga4gh/refget/issues/28>
- <https://github.com/ga4gh/refget/issues/27>


## 2024-05-16 The `sorted_sequences` attribute will be in the spec as an optional ancillary attribute

### Decision

We decided to add `sorted_sequences` to the spec as OPTIONAL.

### Rationale

When digested, this attribute provides a digest representing an order-invariant set of unnamed sequences.
It provides a way to compare two sequence collections to see if their sequence content is identical, but just in a different order.
Such a comparison can be made by the comparison function, so why might you want to include this attribute as well?
In some large-scale use cases, comparing the sequence content without considering order is something that needs to be done repeatedly and for a huge number of collections.
In these cases, using the comparison function could be computationally prohibitive.
This digest allows the comparison to be pre-computed, and more easily compared.

This attribute has been suggested by users for different use cases, and it provides a good example of an ancillary attribute that could be useful for a specific use case where you want to pre-compute this comparison instead of relying on the comparison function.
Thus, it makes sense to include as an example, but made optional since many use cases will not need it.

In the future if the number of proposed ancillary attributes grows, it could move to a separate document together with other ideas for ancillary attributes.

### Linked issues

- <https://github.com/ga4gh/refget/issues/71>


## 2024-02-21 We will specify core sequence collection attributes and a process for adding new ones

### Decision

The sequence collection specification will sanction a number of attributes for which a clear and commonly accepted definition will be provided.
These attributes will be defined in the sequence collection schema and will be part of the specification.

Additional attributes can be requested to be added to the schema via opening an issue on the sequence collection specification GitHub repo. These will be labeled "schema-term".

The set of open issues with this tag can be viewed as an extended seqcol schema that includes all attributes proposed by the community. We RECOMMEND implementers monitor this set to increase forward interoperability.

### Rationale

It is important for the interoperability of services that attributes used in different implementations have the same definition.
To ensure this, the centrally defined schema will provide clear definitions of the most important attributes. 
However it is clear that the maintainers cannot define all possible attributes that implementations might need, so it became apparent that an extended list of attributes that have not been fully defined yet would be useful.

Choosing to host this list as a list of issues allows the list to always be up to date and also contain comment threads where the community can discuss the definition and approval of each attribute.

### Linked issues

 - <https://github.com/ga4gh/refget/issues/50>
 - <https://github.com/ga4gh/refget/issues/46>
 - <https://github.com/ga4gh/refget/issues?q=is%3Aissue+is%3Aopen+label%3Aschema-term>

## 2024-01-10 Clarifications on the purpose and form of the JSON schema in service-info

### Decision

We made a series of decisions regarding how the JSON-schema should be used to specify the data that a seqcol server will serve.

- you MUST provide a single schema.
- the provided schema MUST include all possible attributes your service may provide (you cannot have a collection with an attribute that is not defined in your schema).
- the seqcol spec will include a central, approved seqcol schema
- for any terms defined by the central, approved seqcol schema, any implementations MUST use definitions provided, either by using refs or by duplicating the terms
- we RECOMMEND your schema use property-level refs to point to terms defined by a central, approved seqcol schema, rather than duplicating terms
- we RECOMMEND your schema only define terms actually used in at least one collection you serve.

### Rationale
This set of decisions is oriented around solving a series of related problems.

A JSON schema really serves multiple purposes: 1. validation; 2. configuration of a server; 3. providing information to users about what a server does. Here, the JSON-schema in the service info is really primarily for the third point.

Allowing the JSON schema to use refs introduces some challenges, because now a third-party using that schema will need to resolve those refs. While an existing JSON-schema validator would have a built-in dereferencer, currently, these are only useful if you're using the JSON-schema for validation, which may not be the goal. So, we acknowledge that allowing refs in the JSON schema has potential to make it a bit harder use; but the benefit of being able to share definitions is too great to ignore. The goal here is to increase sharability, and this will be done most effectively if users can easily point to other JSON schemas -- in particular, a central, approved one that is released with recommended terms as part of the seqcol spec.

Another issue is that we wanted the schema to be a place where a user could see what the shape of the data in the server would look like, but we realized this is basically impossible, because different collections in a given server could possibly have different attributes, and therefore, different schemas. Therefore, the only thing that makes sense is for the schema served by the service-info endpoint to have all possible attributes that could be included in any collection hosted by a particular server. This way, you're at least guaranteed that you won't encounter an attribute that is not defined by the schema, though we cannot guarantee that all sequence collections will contain all attributes defined in the schema.

### Linked issues

- <https://github.com/ga4gh/refget/issues/50>
- <https://github.com/ga4gh/refget/issues/39>

## 2024-01-06 The comparison function use more descriptive attribute names

### Decision

This decision complements and updates the one taken on 2022-06-15.
It changes the attribute names to describe specifically what is being returned.
In the top level object:
 - rename `arrays` to `attributes`, and it should describe all attributes regardless of their types.
 - rename `elements` to `array_elements`, and it should describe only attribute of type arrays regardless of the `collated` attribute

In the `array_elements` (previously `elements`):
- remove the `total` and replace it with `a_count` and `b_count` where `a_count` list all the arrays from `a` and the number of element they contain and `b_count` does the same for `b`.
- replace `a_and_b` with `a_and_b_count` -- the content stay the same

### Rationale

The comparison function is designed to compare two sequence collections by interrogating the content of the collated arrays. The initial attribute names were not specifically stating that they only applied to arrays, since originally, we had only been envisioning array attributes. Now that it's more clear how non-array attributes could be included, these updates to the comparison return value clarify which attributes are being referenced.

### Linked issues

- <https://github.com/ga4gh/refget/issues/57>


## 2023-08-25 The user-facing API will neither expect nor provide prefixes

### Rationale

We have debated whether the digests returned by the API should be prefixed in any way (either a namespace-type prefix, or a type prefix). We have also debated whether the API should *accept* prefixed versions of digests. We decided (for now) that neither should be true; our protocol is simply to use and provide the seqcol digests straight up. We view the prefixes as being something useful for an external context to determine where a digest came from or belongs; but this seems external to how our service should behave internally. Therefore, any sort of prefix should be happening by whatever is *using* this service, not by the service itself. For example, a ga4gh-wide broker that could disambiguate between different types of digests may require an incoming digest to have a type prefix, but this would be governed by such a context-oriented service, not by the seqcol service itself.

In the future, when it becomes more clear how this service will fit in with other services in the ga4gh ecosystem, we could revisit this decision.


## 2023-08-22 - Seqcol schemas MUST specify collated attributes with a local qualifier

### Decision

Collated attributes are seqcol attributes where the values of the attribute are 1-to-1 with sequences in the collection, and represented in the same order as the sequences in the collection. Names, lengths, and sequences are examples of collated attributes. While not strictly required for the core functionality of sequence collections, we anticipate downstream applications will benefit if the seqcol JSONschema specifies which attributes are collated. We therefore REQUIRE the seqcol JSONschema to specify which attributes are collated, and these will be specified using a local boolean qualifier (defined below). 

### Rationale

For applications that will visualize or present to the user a representation of a sequence collection, it will be useful to know if any attributes have one-value-per-sequence, or something else. The collated attributes are those that belong to each sequence. It is possible that other attributes will exist on the seqcol object, but a generic visualization engine or other processor will benefit from knowing which ones are collated.

But how to specify them in the JSONschema? In jsonschema, there are 2 ways to qualify properties: 1) a local qualifier, using a key under a property; or 2) an object-level qualifier, which is specified with a keyed list of properties up one level. For example, you annotate a property's `type` with a local qualifier, underneath the property, like this:

```
properties:
  names:
    type: array
```

However, you specify that a property is `required` by adding it to an object-level `required` list that's parallel to the `properties` keyword:

```
properties:
  names:
    type: array
required:
  - names
```

In sequence collections, we chose to use define `collated` as a local qualifier. Local qualifiers fit better for qualifiers independent of the object as a whole. They are qualities of a property that persist if the property were moved onto a different object. For example, the `type` of an attribute is consistent, regardless of what object that attribute were defined on. In contrast, object-level qualfier lists fit better for qualifiers that depend on the object as a whole. They are qualities of a property that depend on the object context in which the property is defined. For example, the `required` modifier is not really meaningful except in the context of the object as a whole. A particular property could be required for one object type, but not for another, and it's really the object that induces the requirement, not the property itself.

We reasoned that `inherent`, like `required`, describes the role of an attribute in the context of the whole object; An attribute that is inherent to one type of object need not be inherent to another. Therefore, it makes sense to treat this concept the same way jsonschema treats `required`.  In contrast, the idea of `collated` describes a property independently: Whether an attribute is collated is part of the definition of the attribute; if the attribute were moved to a different object, it would still be collated.

For example, here the `lengths` attribute is maraked as collated using a local qualifier. The `author` attribute is marked as *not* collated in the same way:

```
description: "A collection of biological sequences."
type: object
properties:
  lengths:
    type: array
    collated: true
    description: "Number of elements, such as nucleotides or amino acids, in each sequence."
    items:
      type: integer
  author:
    type: string
    collated: false
    description: "The author of this sequence collection"
...
```


### Linked issues
- https://github.com/ga4gh/refget/issues/40


## 2023-07-26 There will be no metadata endpoint

### Decision

We have no need for a `/metadata` endpoint

### Rationale

At one point (issue #3), we debated whether there should be a `/metadata` endpoint or something like that as a way to retrieve information about a sequence that might not be part of the digested sequence. However, after we distinguished between `inherent` and `non-inherent` attributes, we have realized that this satisfies the earlier requirement for a `/metadata` endpoint; in fact, the metadata can be returned to the user through the normal endpoint, and just flagged as `non-inherent` in the schema to indicate that it's not digested, and therefore not part of the identity of the object

We distinguished between two types of metadata:

- server-scoped metadata, like the schema we described above, should be served by `/service-info`
- collection-scoped or sequence-scoped metadata don't fit under `/service-info`. For these, they will be served by the primary `/collection` endpoint, rather than by a separate `/metadata` endpoint.

### Linked issues

- <https://github.com/ga4gh/refget/issues/3>
- <https://github.com/ga4gh/refget/issues/39>
- <https://github.com/ga4gh/refget/issues/40>

## 2023-07-12 - Required attributes are: lengths and names

### Decision

A sequence collection consists of a set of arrays. The only arrays that MUST be included for a valid sequence collection are *lengths* and *names*. All other possible arrays, including *sequences* and other controlled vocabulary arrays, are not required.

### Rationale

Debate around what should be mandatory as centered on 3 specific attributes: sequences, names, and lengths:

At first, it feels like sequences are fundamental components of sequence collections, and therefore, the *sequences* array should be mandatory, and names and lengths may be superfluous. For reference genomes, for example, it's clear that collections of sequences are the main function of sequence collections. However, analysis of reference genome data also includes many analyses for which the sequences themselves do not matter, and the critical component is simply the name and length of the sequence. An array of names and lengths can be thought of as a *coordinate system*, and we have realized that the sequence collection specification is *also* extremely useful for representing and uniquely identifying coordinate systems. From this perspective, we envision a coordinate system as a sequence collection in which the actual sequence content is irrelevant, but in which the lengths and names of the sequences are critical. Analysis of coordinate systems like this is very frequent. For example, any sort of annotation analysis looking at genomic regions will rely on the lengths of the sequences to enforce that coordinates refer to the same thing, but do not rely on the underlying sequences. This is why "chrom-sizes" files are used so frequently (*e.g.* across many UCSC tools).

This leads us to the conclusion that *sequences* should be optional, and *names* and *lengths* should be the only mandatory components. *Lengths* makes sense because if you have a sequence, you can always compute it's length, but if you don't have a sequence (all you have is a coordinate system), you may only have a length. We debated extensively whether *names* should be mandatory, and in the end, decided that it's unlikely to pose much of a difficulty to make it mandatory, and provides a lot of convenience. If sequences lack names altogether, it is trivial to name them by index of the order of the sequences. We reason that downstream use cases are very likely to require at least *some* type of identifier to refer to each of the sequences, even if it's just the index of the sequence in the list. While it may be possible to imagine a use case where an identifier for each sequence is not required, it's not difficult at all to just assign indexes. By making it required, we ensure that implementations will always have the same possible way to reference the sequences in the collection. Also, one potential use case for dropping the *names* array, namely to provide name-invariant sequence records for mapping purposes, will instead be possible to solve through defining an extra *non-inherent* and name-invariant attribute.

## 2023-07-12 Implementations SHOULD provide sorted_name_length_pairs and comparison endpoint

### Decisions

1. Name of the "names-lengths" attribute should be `sorted_name_length_pairs`.
2. The `sorted_name_length_pairs` is RECOMMENDED.
3. The `/comparison` endpoint is RECOMMENDED.
4. The algorithm for computing the `sorted_name_length_pairs` attribute should be as follows:

### Algorithm for computing `sorted_name_length_pairs`

1. Lump together each name-length pair from the primary collated `names` and `lengths` into an object, like `{"length":123,"name":"chr1"}`.
2. Canonicalize JSON according to the seqcol spec (using RFC-8785).
3. Digest each name-length pair string individually.
4. Sort the digests lexographically.
5. Add as an undigested, uncollated array to the sequence collection.


### Rationale and alternatives considered

1. We considered `names_lengths`, `sorted_names_lengths`, `name_length_pairs`. In the end we are trying to strike a balance between descriptiveness and conciseness. We decided the idea of "pairs" is really critical, and so is "sorted", so this seemed to us to be a minimal set of words to capture the intention of the attribute, though it is a bit long. But in the end the name itself just has to be *something* standardized, and nothing seems perfect.

2. We debated whether it should be required or optional to provide the `sorted_name_length_pairs` attribute. We think it provides a lot of really nice benefits, particularly if everyone implements it; however, we also acknowledge that there are some use cases for seqcols (like just being a provider of sequence collections) where every collection will have sequences, and comparing among coordinate systems is not really in scope. For this use case, we acknowledge that the sorted_name_length_pairs may not have utility, so we make it RECOMMENDED.

3. Similarly, we envisioned the possibility of a minimal implementation built using object storage that could fulfill all the other specifications. So while we think that the comparison function will be very helpful, particularly if it's implemented everywhere, for a minimal implementation that's sole purpose is to provide sequences, it might make sense to opt out of this. Therefore, we call it recommended.

### Linked issues

- <https://github.com/ga4gh/refget/issues/40>


## 2023-06-14 - Internal digests SHOULD NOT be prefixed

### Background

In some situations, digests are prefixed. For example, these may be CURIEs, which specify namespaces or provide other information about what the digest represents. This raises questions about when and where we should expect or use prefixes. This has to be determined because including prefixes in the content that gets digested changes it, so we have to be consistent.

### Decision

We determined that *internally*, we will not append prefixes to the strings we are going to digest. However, if a particular identifier defines some kind of a prefix *as part of the identifier* (*e.g.* a refget sequence identifier), then it's of course no problem, we take that identifier at face value. To summarize:

- for internal identifiers (those generated within seqcol), we digest only digests, not prefixes of any kind
- for external identifiers (like refget identifiers), we accept them at face value, so we wouldn't remove a prefix if you declare it is was part of your sequence identifier
- the seqcol specification should RECOMMEND using refget identifiers

More specifically, for refget, there are two types of prefix: the namespace prefix (`ga4gh:`) and type type prefix (`SQ.`).
Right now, the refget server requires you to have the type prefix to request a lookup; the refget protocol declares that this type prefix is *part of the identifier*.
However, the `ga4gh:` prefix is more of a namespace prefix and is *not* required, and therefore not considered part of the identifier.
Therefore, the seqcol `sequence` values would *include* the `SQ.` but not the `ga4gh:`.

### Rationale

According to the definition of CURIEs:

> A host language MAY declare a default prefix value, or MAY provide a mechanism for defining a defining a default prefix value. In such a host language, when the prefix is omitted from a CURIE, the default prefix value MUST be used.

We see no need to add prefixes to the identifiers we use internally, which we just assume belong to our namespace.
Adding prefixes will complicate things and does not add benefits. Prefixes may be added to our identifiers by outside entities as needed to define for them the scope of our local digests.

### Linked issues

- <https://github.com/ga4gh/refget/issues/37>


## 2023-06-28 - SeqCol JSON schema defines reserved attributes without additional namespacing

### Decision

One potential issue may arise if a custom implementation uses an attribute that a future version of seqcol adds to the schema, and if these attributes are defined differently. This will create a name clash and the custom implementation wouldn't be compatible with the future seqcol schema. It would be nice to prevent such future clashes, which would require ensuring that future seqcol attributes mean the same thing across collections so they are compatible. One way to solve this would be to define an namespace reserved by the specification, so that custom attributes look different and will therefore guarantee that custom attribute never clashes with seqcol attributes, thereby ensuring that custom implementations will be compatible with any future schema updates.

Despite the potential issue for custom attribute clashes, we decided:

1. We will not use any additional namespacing. Instead, the SeqCol schema declares and defines the specific attributes of a sequence collection. We will "claim" any reserved keywords in the secqol schema we publish, not by defining a style or namespace of reserved keywords.

2. We will try to add to this many things that we forsee as possible attributes that could be defined in a seqcol. Thus, we will provide an official set of definitions that should prevent many possible future clashes.

3. We will specify that for custom attributes, you can do what you want outside the reserved keywords; but you should be aware that if a word becomes part of the official schema in the future, this could require a change of your custom attribute to maintain backwards compatibility. We will advise that if possibility of future clashes is important for an external schema, they could prevent that by prefixing custom attributes. However, this also means that if a future attribute *is* added to the schema to represent that concept, it would not follow the custom name.

4. Third-party implementers may propose attributes that should be moved into the primary seqcol schema for subsequent release. These proposals could happen via raising an issue in the specification repository.

### Rationale

Several reasons led us to this decisions:

1. The likelihood of wanting to add custom attributes that will clash and have different definition seems low, so we questioned whether it is worth the cost of defining separate namespaces.
2. In the event that there is a clash in the future, this is not really a major problem. A new version of the official schema that adds new reserved keywords will basically mean a new major release of seqcol, which could potentially introduce backwards incompatibility with an existing custom attribute. This just means the custom implementation would need to be updated to follow the new schema, which is possible.
3. It seems more likely that we would "claim" an official attribute that someone else had already used that *does* match the intended semantics of the word. In that case, our effort to prevent clashes would have actually created clashes, because it would have forced the custom attribute to use a different attribute name. Instead, it seems more prudent to just allow the custom implementations to use the same namespace of attribute names, and deal with any possible backwards incompatibilities if they ever actually arise in the future.
4. Since we expect the major implementations to be few and driven by people connected with the project, it seems more likely that we would just adopt the custom attribute with its definition as an official attribute. We would not be able to do this if we enforced separate namespaces, which would create backwards compatibility.

In other words, in short: the idea to prevent future backwards-incompatibility by creating a reserved word namespace seems, paradoxically, more likely to actually *create* a future backwards compatibility than to prevent one.

## 2023-06-28 Details of endpoints

### Decisions

1. The specification for how to retrieve different representations of a sequence collection should be specified to the `/collection` endpoint with `?level=<level>`, where `<level>` interpretations are:
	- `level` <= 0 is undefined
	- the return value is JSON for all 
	- `?level=1` MUST be allowed, and must return the level 1 seqcol representation
	- `?level=2` MUST be allowed, and must return the level 2 seqcol representation
	- `?level` is OPTIONAL, and when not provided, `level=2` is assumed

2. The `/comparison` endpoint is RECOMMENDED.

### Rationale

1. The different levels of representation are useful for different things and it makes sense to make it possible to retrieve them. We debated about the best way to do this, and also considered using names instead of numbers.

2. The comparison endpoint is very useful, but we can imagine use cases where it can cause problems or may not be needed. First, it will preclude the ability of creating an S3-only implementation. Since it's possible and useful to create an implementation that only implements the `/collection` endpoint, it makes sense that `/comparison` should not be required. Second, some services may view themselves as solely providing content, and nothing more. We recommend these services still implement `/comparison`, but acknowledge that the `/collection` endpoint will still be useful even without it, so this again fits with a `RECOMMEND` status.



## 2023-03-22 - Seqcol schemas MUST specify inherent attributes

### Decision

A seqcol schema provided by a seqcol API `MUST` define an `inherent` section. This section specifies a list of attributes, indicating the attributes that **do** contribute to the identity of the collection. As a corollary, attributes of a seqcol that are *not* listed in `inherent` `MUST NOT` contribute to the identifier; they are therefore excluded from the digest calculation.

### Rationale

We have found a lot of useful use cases for information that should go along with a seqcol, but should not contribute to the *identity* of that seqcol. This is a useful construct as it allows us to include information in a collection that does not affect the identifier that is computed for that collection. One simple example is the "author" or "uploader" of a reference sequence; this is useful information to store alongside this collection, but we wouldn't want the same collection with two different authors to have a different identifier! Similarly, the 'sorted_name_length_pairs' idea provides lots of utility, but it doesn't change anything about the identity of the collection, so it would be nice to exclude it because then an implementation that didn't implement 'sorted_name_length_pairs' would end up with the same identifier, improving interoperability across servers.

Thus, we introduce the idea of *inherent* vs *non-inherent attributes*. Inherent attributes contribute to the identifier; *non-inherent* attributes are not considered in computing the top-level digest. We previously called these *digested* and *non-digested* attributes, but this is not really a good name because, while these non-inherent attributes may not be part of the top-level digest calculation, they are still going to be digested at level 2.

### Linked issues

- <https://github.com/ga4gh/refget/issues/40>

### Alternatives considered

We considered using `extrinsic` to define the opposite of `inherent`, which would change it so that attributes were inherent by default; but we decided we liked the explicitness of forcing the schema to specify which attributes are to be included in the digest, because this brings clarity over the alternative, which is to assume everything is included unless it's excluded. We also liked that this makes the `inherent` keyword behave similarly to the `required` keyword in JSON-schema; if left off, we assume nothing is required. This means that in order for a seqcol schema to be valid, it must have at least one inherent attribute specified.

## 2023-02-08 - Array names SHOULD be ASCII

### Decision

Custom array names SHOULD be ASCII characters. We expect most implementations will require this; nevertheless, implementers may choose to allow UTF-8 characters as an extension to the spec. Implementing UTF-8 is not required for an implementation. In this extension, array names MUST at least follow UTF-8.

### Rationale

The sequence collection is a group of named arrays. These array names include built-in, defined arrays, like names, lengths, and sequences, but users may also use custom array names. Our spec-defined array names are all lowercase ASCII characters, but this doesn't mean we must restrict custom array names in the same way.

While non-ASCII array names would be compatible with our current specification, we identified 3 issues that could arise if someone uses non-ASCII: 1) Normalization. We would probably need to define in the specification some normalization scheme to make sure things a user expects to be identical will hash to the same digest. 2) Sort order. However, this problem will be solved by following a JSON canonicalization standard. 3) Use of array names in other places will be restricted. For example, it seems natural to want to create API endpoints or table names or in columns in a database that correspond to array names. If array names are non-ASCII, it may preclude this, increasing implementation complexity and may make some things impossible.

### Linked issues

- <https://github.com/ga4gh/refget/issues/33>


## 2023-01-25 - The digest algorithm will be the GA4GH digest

### Decision

`sha512t24u` digests must be used instead of `md5` for sequence collection digests.

### Rationale 

`sha512t24u` was created as part of the [Variation Representation Specification standard](https://vrs.ga4gh.org/en/stable/impl-guide/computed_identifiers.html) and used within VRS to calculate GA4GH identifiers for high-level domain objects in combination with a type prefix map. The `sha512t24u` function ([Hart _et al_. 2020](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0239883)) is described as:

- performing a SHA-512 digest on a binary blob of data
- truncate the resulting digest to 24 bytes
- encodes the 24 bytes using `base64url` ([RFC 4648](https://datatracker.ietf.org/doc/html/rfc4648#section-5)) resulting in a 32 character string

Under this scheme the string `ACGT` will result in the `sha512t24u` digest `aKF498dAxcJAqme6QYQ7EZ07-fiw8Kw2`. This digest can be converted into a valid refget identifier by prefixing `SQ.`. 

`sha512t24u` was envisaged as a fast digest mechanism with a space-efficient representation that can be used for any data with low collision probability. Collisions have been (documented in `md5`)[https://en.wikipedia.org/wiki/MD5#Collision_vulnerabilities] leading to the belief MD5 was insufficient for our needs.

`sha512t24u` must be used for any digest of data **by** the sequence collections standard. This decision does not disallow the use of `md5` sequence checksums.

### Limitations

`MD5` is easier to calculate and familiar as many systems ship with a command line `md5` binary. `sha512t24u` needs to be typed when used outside of an implementation to avoid issues of collision.

### Linked issues

- [https://github.com/ga4gh/refget/issues/30](https://github.com/ga4gh/refget/issues/30)


## 2023-01-12 - How sequence collection are serialized prior to digestion

The serialisation in this context is the conversion of the sequence collection object into a string that can be digested.

### Decision

The serialisation of a sequence collection will use the following steps

 1. Apply RFC-8785 on each array of level 2
 2. Digest the canonical representation of each array
 3. Create object representation of the seq-col using array names and digested arrays
 4. Apply RFC-8785 on the object representation
 5. Digest the final canonical representation


#### 1. Apply RFC-8785 for converting from level 2 to level 1

For example the length array at level 2:
```json
[248956422, 242193529, 198295559]
```

Will be serialised using RFC-8785 and digested as a binary string. Here the output of the python implementation: 

```python
b'[248956422,242193529,198295559]'
```

It would also support any UTF-8 character. For example this array of names
```json
["染色体-1","染色体-2","染色体-3"]
```

Would create the following serialisation:

```python
b'["\xe6\x9f\x93\xe8\x89\xb2\xe4\xbd\x93-1","\xe6\x9f\x93\xe8\x89\xb2\xe4\xbd\x93-2","\xe6\x9f\x93\xe8\x89\xb2\xe4\xbd\x93-3"]'
```

#### 2. Digest of the canonical representation

The canonical string representation is then digested. Assuming the use of GA4GH (sha512 trim to 24) digest, the following array of length

```python
b'[248956422,242193529,198295559]'
```

would be converted to 

```json
"5K4odB173rjao1Cnbk5BnvLt9V7aPAa2"
```

#### 3. Creation of an object composed of the array names and the digested arrays
An object is created with the array name as properties and the digest as value.
Example the following collection: 
```json
{
    "sequences": "EiYgJtUfGyad7wf5atL5OG4Fkzohp2qe",
    "lengths": "5K4odB173rjao1Cnbk5BnvLt9V7aPAa2",
    "names": "g04lKdxiYtG3dOGeUC5AdKEifw65G0Wp"
}
```

#### 4. Use RFC-8785 on the object
This will create a canonical representation of the object 

```python
b'{"lengths":"5K4odB173rjao1Cnbk5BnvLt9V7aPAa2","names":"g04lKdxiYtG3dOGeUC5AdKEifw65G0Wp","sequences":"EiYgJtUfGyad7wf5atL5OG4Fkzohp2qe"}'
```

#### 5. Digest the final canonical representation
Finally the canonical, representation is digested again to produce the identifier

```json
"S3LCyI788LE6vq89Tc_LojEcsMZRixzP"
```

### Rationale
The decision to use the serialisation of array and object provided in RFC-8785 allows sequence collection to support any type of characters and rely on a documented standard that offer implementation in multiple languages.
It also future-proofs the serialisation method if we ever allow complex object to be element of the array.
 
### Linked issues

 - [https://github.com/ga4gh/refget/issues/1](https://github.com/ga4gh/refget/issues/1)
 - [https://github.com/ga4gh/refget/issues/25](https://github.com/ga4gh/refget/issues/25)
 - [https://github.com/ga4gh/refget/issues/33](https://github.com/ga4gh/refget/issues/33)


### Known limitations

The JSON canonical serialisation defined in RFC-8785 has a limited set of reference implementation. It is possible that its implementation makes sequence collection implementation more difficult in languages where the RFC is not implemented. In this cases it is valuable to note that the current specification of Sequence Collection do not require that all the features of RFC-8785 be implemented. 

### Alternatives considered

We spent a significant amount of time discussing approaches for what essentially amounts to a custom standard for creating the string-to-digest. A lot of this revolved around what delimiters to use. We made a lot of progress there and came up with some really interesting encoding schemas, which had many desirable characteristics. However, ultimately we decided that the value derived from using a comprehensive and well-developed third-party solution would trump the elegance, efficiency, and other benefits we received from our custom encoding schema. In particular, adopting the RFC-8785 would make developers more likely to be able to rely on third-party implementations, reducing the burden to implement our standard. Also, this solution accommodates other sources that we had struggled with a bit, such as UTF-encoding.

## 2022-10-05 - Terminology decisions

### Decision

We refer to representations in "levels". The level number represents the number of "lookups" you'd have to do from the "top level" digest. So, we have:

#### Level 0 (AKA "top level")

Just a plain digest. This corresponds to **0 database lookups**. Example:
```
a6748aa0f6a1e165f871dbed5e54ba62
```

#### Level 1

What you'd get when you look up the digest with **1 database lookup** and no recursion. Previously called "layer 0" or "reclimit 0" because there's no recursion. Also sometimes called the "array digests" because each entity represents an array.

Example:
```
{
  "lengths": "4925cdbd780a71e332d13145141863c1",
  "names": "ce04be1226e56f48da55b6c130d45b94",
  "sequences": "3b379221b4d6ea26da26cec571e5911c"
}
```

#### Level 2

What you'd get with **2 database lookups** (equivalently, 1 recursive call). This is the most common representation, more commonly used than either the "level 1" or the "level 3" representations.

```
{
  "lengths": [
    "1216",
    "970",
    "1788"
  ],
  "names": [
    "A",
    "B",
    "C"
  ],
  "sequences": [
    "76f9f3315fa4b831e93c36cd88196480",
    "d5171e863a3d8f832f0559235987b1e5",
    "b9b1baaa7abf206f6b70cf31654172db"
  ]
}
```

#### Level 3

What you'd get with **3 database lookups** (equivalently, 2 recursive call). The only field that can be further populated is `sequences`, so the level 3 representation provides the complete data. This layer:
- can potentially be very large
- is the only level that requires outsourcing a query to a refget server
- may reasonable be disabled on my seqcol server, since the point is not to retrieve actual sequences; 

Example (sequences truncated for brevity):
```
{
  "lengths": [
    "1216",
    "970",
    "1788"
  ],
  "names": [
    "A",
    "B",
    "C"
  ],
  "sequences": [
    "CATAGAGCAGGTTTGAAACACTCTTTCTGTAGTATCTGCAAGCGGACGTTTCAAGCGCTTTCAGGCGT...",
    "AAGTGGATATTTGGATAGCTTTGAGGATTTCGTTGGAAACGGGATTACATATAAAATCTAGAGAGAAGC...",
    "GCTTGCAGATACTACAGAAAGAGTGTTTCAAACCTGCTCTATGAAAGGGAATGTTCAGTTCTGTGACTT..."
  ]
}
```
#### Summary

We should be consistent by using these terms to refer to the above representation:
-  "level 0 representation", "level 0 digest", top-level digest", or "primary digest";
-  "level 1 representation" or "level 1 digests";
-  "level 2 representation"; 
-  "level 3 representation" of a sequence collection. 


### Linked issues
- <https://github.com/ga4gh/refget/issues/25>


## 2022-06-15 - Structure for the return value of the comparison API endpoint

### Decision

The compare function return value MUST be an object following the REQUIRED format specified below.

**REQUIRED**: The endpoint MUST return, in JSON format, an object with these 3 keys: "digests", "arrays", "elements". 

- *digests*: an object with 2 elements, with keys *a* and *b*, and values either the level 0 seqcol digests for the compared collections, or *undefined (null)*. The value MUST be the level 0 seqcol digest for any digests provided by the user for the comparison. However, it is OPTIONAL for the server to provide digests if the user provided the sequence collection contents, rather than a digest. In this case, the server MAY compute and return the level 0 seqcol digest, or it MAY return *undefined (null)* in this element for any corresponding sequence collection.
- *arrays*: an object with 3 elements, with keys *a_only*, *b_only*, and *a_and_b*. The value of each element is a list of array names corresponding to arrays only present in a, only present in b, or present in both a and b.
- *elements*: An object with 3 elements: *total*, *a_and_b*, and *a_and_b-same-order*. *total* is an object with *a* and *b* keys, values corresponding to the total number of elements in the arrays for the corresponding collection. *a_and_b* is an object with names corresponding to each array present in both collections (in *arrays.a_and_b*), with values as the number of elements present in both collections for the given array. *a_and_b-same-order* is also an object with names corresponding to arrays, and the values a boolean following the same-order specification below.

Example: 

```
{
  "digests": {
    "a": "514c871928a74885ce981faa61ccbb1a",
    "b": "c345e091cce0b1df78bfc124b03fba1c"
  },
  "arrays": {
    "a_only": [],
    "b_only": [],
    "a_and_b": [
      "lengths",
      "names",
      "sequences"
    ]
  },
  "elements": {
    "total": {
      "a": 195,
      "b": 25
    },
    "a_and_b": {
      "lengths": 25,
      "names": 25,
      "sequences": 0
    },
    "a_and_b-same-order": {
      "lengths": false,
      "names": false,
      "sequences": null
    }
  }
}
```

#### Same-order specification

The comparison return includes an *a_and_b-same-order* boolean value for each array that is present in both collections. The defined value of this attribute is:

- *undefined (null)* if there are fewer than 2 overlapping elements
- *undefined (null)* if there are unbalanced duplicates present (see definition below)
- *true* if all matching elements are in the same order in the two arrays
- *false* otherwise.

An *unbalanced duplicate* is used in contrast with a *balanced duplicate*. Balanced means the duplicates are the same in both arrays. When the duplicates are balanced, order is still defined; but if duplicates are unbalanced, this means an array has duplicates not present in the other, and in that case, order is not defined.

### Rationale

The primary purpose of the compare function is to provide a high-level view of how two sequence collections match and differ. The primary use cases are to see if collections are identical or subsets, and to assess the degree of overlap in each attribute (such as sharing all sequence digests, sequence names, or lengths). If more details are needed, the user will need to look in more depth at the raw elements of the sequence collection. It's important to have a fast, easy-to-implement, and minimal payload function to provide answers to the common question about "how compatible are these two collections".

### Linked issues

- <https://github.com/ga4gh/refget/issues/21>
- <https://github.com/ga4gh/refget/issues/7>

### Alternatives considered

We considered a simpler arrangement that would only return true/false values as to whether the arrays matched but in the different order, or contained any matching elements vs no matching elements. While this would have been faster to compute than the counting approach we settled on, there was concern that it would not be enough information to interpret the comparison. We also considered more information-rich values that would enumerate overlapping or non-overlapping elements. We finally concluded that the most useful would be the middle ground proposed here, where you get counts but no enumerated elements. This provides sufficient information to make a pretty detailed comparison, and can still be computed relatively quickly and keeps the payload size small and predictable.

### Known limitations

Someone may want to return more information than this, such as enumerating the specific elements in each category. However, this use case would be problematic for large collections, like a transcriptome. We may in the future provide an update to the specification that defines how this information should be returned, but for now, we leave the specification at this minimum requirement.

## 2022-06-15 - We will define the elements of a sequence collections using a schema

### Decision

The elements of a sequence collection will be defined and described using JSON Schema. The exact contents of the JSON Schema will be determined later. Here is an example JSON Schema as an illustration of the concept (encoded in YAML for readability):

```yaml

description: "A collection of sequences, representing biological sequences including nucleotide or amino acid sequences. For example, a sequence collection may represent the set of chromosomes in a reference genome, the set of transcripts in a transcriptome, a collection of reference microbial sequences of a specific gene, or any other set of sequences."
type: object
properties:
  lengths:
    type: array
    description: "Number of elements, such as nucleotides or amino acids, in each sequence."
    items:
      type: integer
  masks:
    type: array
    description: "Digests of subsequence masks indicating subsequences to be excluded from an analysis, such as repeats"
    items:
      type: string
  names:
    type: array
    description: "Human-readable identifiers of each sequence, commonly called chromosome names."
    items:
      type: string
  priorities:
    type: array
    description: "Annotation of whether each sequence is a primary or secondary component in the collection."
    items:
      type: boolean
  sequences:
    type: array
    description: "Digests of sequences computed using the GA4GH digest algorithm (sha512t24u)."
    items:
      type: string
      description: "Actual sequence content"
  topologies:
    type: array
    description: "Annotation of whether each sequence represents a linear or other topology."
    items:
      type: string
      enum: ["circular", "linear"]
      default: "linear"
required:
  - lengths
digested:
  - lengths
  - names
  - masks
  - priorities
  - sequences
  - topologies
```



### Rationale

We need a formal definition of a sequence collection. The schema provides a machine-usable definition that specifies the names and types of what we envision as a sequence collection. It also provides a convenient way to describe those elements in human-understandable ways through the `description` field. Thus, this schema solves a number of issues. Most importantly, it answers the question of *what are the elements of the sequence collection, and how are they defined?* For example, this schema answers the more specific question: *what are the allowable or expected algorithms for items included in the sequence array (or possibly other, custom digested arrays?)*. The schema establishes a standard expectation for elements that will go into general sequence collections. Implementations are free to add to this schema for their instances as needed.

### Linked issues

- <https://github.com/ga4gh/refget/issues/8>
- <https://github.com/ga4gh/refget/issues/6>


## 2021-12-01 - Endpoint names and structure

### Decision

The endpoint names will be:

- `GET /service-info` for GA4GH service info
- `GET /collection/{digest}` for retrieving a sequence collection
- `GET /comparison/{digest1}/{digest2}` for comparing two collections in the database
- `POST /comparison/{digest1}` for comparing one database collection to a local user-provided collection.

The POST body for the local comparison is a "level 2" sequence collection, like this:

```
{
  "lengths": [
    "248956422",
    "242193529",
    "198295559"
  ],
  "names": [
    "chr1",
    "chr2",
    "chr3"
  ],
  "sequences": [
    "a004bc1b0bf05fc668cab6bbfd93d3eb",
    "0ccf3a67666ac53f99fcad19768f2dde",
    "bda7b228789169ae811dd8d676d517ca"
  ]
}
```

### Rationale

We wanted to stick with the REST guideline of noun endpoints with GET that describe what you are retrieving. As recommended in the [service-info specification](https://github.com/ga4gh-discovery/ga4gh-service-info#how-do-i-describe-a-service-implementing-multiple-specifications), a prefix, like `/seqcol/...` could be added by a service that implemented multiple specifications, but this kind of namespace it outside the scope of the specification itself. We considered doing `/{digest1}/compare/{digest2}` and that would have been fine. In the end we liked the symmetry of `/comparison` and `/collection` as parallel endpoints. For the retrieval endpoint we considered `/seqcol` or `/sequence-collection` or `/seqCol`, but wanted to keep structure parallel to the refget `/sequence` endpoint.

### Limitations

For the `POST comparison` endpoint, we made 2 limitations to simplify the implementation of the function. First, we do not require it to allow comparing 2 local collections, which could be enabled, but we reason that users should always be comparing against something in the database, and this prevents abusing the system as a computing engine. We also disallowed (or at least don't explicitly require) comparing a level 1 collection (which consists of a named list of array digests), as we figured that most frequently the user will have the array details, and if not, they could look them up.

### Linked issues

- [https://github.com/ga4gh/refget/issues/21](https://github.com/ga4gh/refget/issues/21)
- [https://github.com/ga4gh/refget/issues/23](https://github.com/ga4gh/refget/issues/23)

## 2021-09-21 - Order will be recognized by digesting arrays in the given order, and unordered digests will be handled as extensions through additional attributes

### Decision

The final sequence collection digests will reflect the order by digesting the arrays in the order provided. We will employ no additional 'order' array, and no additional unordered digests *in the string-to-digest*. Any additional attributes designed to handle questions with order, such as `sorted_name_length_pairs`, will not contribute to the digest. Thus, to determine whether two sequence collections differ only in order will require either 1. using the comparison API; or 2. implementing additional functionality via digests outside the inherent attributes.

### Rationale

Our earlier decision determined that order *must* be reflected in the sequence digests, but did not determine the way to ensure that. After months of debate we came up with 4 competing ideas that could do this:

A. Digest arrays in given order. 

B. Reorder all given arrays according to a single canonical order, and encode order in a separate 'order' array that provides an index into the canonically ordered arrays.

C. Reorder each given array individually, and then provide a separate 'order_ATTR' array as an index for each array.

D. Store each array in both ordered and unordered form.

After lots of initial enthusiasm for option B, we determined that it fails to deliver on the promise of staying invariant when order changes, because if there is a change in any array on which the canonical order is based, this changes the canonical ordering, which in turn changes all the array digests. So these 'unordered' (or canonically ordered) digests are in fact not fit for their main purpose. We therefore agreed to discard this option.

While options C/D skirt this issue by having a separate order for each array, so that changes in one array do not affect the digest of another, they add significant complexity as everything needs to be stored twice.

To conclude, option A seems simple and straightforward, satisfies for a basic implementation. We thus defer the question of determining whether two sequence collections differ only in order to the comparison API, or to some other future way to do it that will not affect the actual digests (*e.g.* the 'sorted_name_length_pairs' attribute).

### Linked issues

- https://github.com/ga4gh/refget/issues/5

### Known limitations

For use cases that require determination of whether two sequence collections differ only in element order, option A will not provide an answer based on digest comparison alone. Instead, the query will be required to use the compatibility API, which means retrieving the contents of the array to compare them.

Therefore, to answer this 'order-equivalence' question will require a bit more work than if unordered digests were available; however, this functionality can be easily implemented on top of the basic functionality in a number of ways, which we are continuing to consider.


## 2021-08-25 - Sequence collection digests will reflect sequence order

### Decision

The final sequence collection digests must reflect the order of the sequences given. In other words, changing the order of the sequences will change the identifier.

### Rationale

In some scenarios, the order of the sequences in a collection is irrelevant, and therefore, two collections with identical content but a different order should be considered equivalent. This could lead to an approach of first lexographically sorting input sequences before digesting, so that the final identifier is identical regardless of input order.

However, there are also scenarios for which the order of sequences in a collection matters; for example, some aligners output different results based on the input order of the sequences. Or, order may be used to encode sequence priority. Therefore, it is critical that the final identifiers be able to uniquely identify sequence collections with different orders. Because some use cases require order-aware digests, the final algorithm will have to accommodate this, and we will need to come up with another way to identify two collections that are identical in content but with different order, without relying on the digests being identical

### Linked issues

- [https://github.com/ga4gh/refget/issues/5](https://github.com/ga4gh/refget/issues/5)

### Known limitations

This decision will lead to a bit more complexity for use cases that do not care about sequence order, because these use cases will no longer be able to rely on simple, identical digest matching. However, this is a necessary increase in complexity to accommodate other use cases where order matters.


## 2021-06-30 - Use array-based data structure and multi-tiered digests

### Decision

Our original formulation structured data with groups of a sequence plus its  annotation, like `chr1|248956422|2648ae1bacce4ec4b6cf337dcae37816/chr2|242193529|4bb4f82880a14111eb7327169ffb729b|`. Instead, we decided to switch to an array-based model, in which a sequence collection will be constructed as a dictionary object, with attributes as named arrays, like this:

```
seqcol = {'names': ['chrUn_KI270742v1',   'chrUn_GL000216v2',   'chrUn_GL000218v1'],
  'lengths': ['186739', '176608', '161147'],
  'sequences': ['2f31c013a4a8301deb8ab7ed1ca1cd99',   '725009a7e3f5b78752b68afa922c090c',   
'1d708b54644c26c7e01c2dad5426d38c']}
```

Digests of this object will be the unique identifier of the sequence collection. To compute the digest of this object, we will use a multi-tiered digest approach, where each array will first be digested separately, and then the final identify will be a digest of digests. A sequence collection may thus be represented as a dictionary of digests:

```
{ 'sequences': '8dd93796fa0225e92eb159a8779f1b254776557f748f8bfb',
 'lengths': '501fd98e2fdcc276c47306bd72c9155489ed2b23123ddfa2',
 'names': '7bc90a07cf25f2f64f33baee3d420ad1ae5f442055280d43',
}
```

This will allow retrieving individual attributes, and testing for identity of individual attributes without retrieving more data.

### Rationale

- This makes it straightforward to mix-and-match components of the collection. Because each component is independent, and not integrated in with the sequence, it is simpler to select and build subsets and permutations.
- We can add a new component without eliminating backwards compatibility with previous digests that did not include it, because leaving out an array doesn't change the string to digest.
- This makes it easier to test for matching sets of sequences, or matching coordinate systems, using the individual component digests. This way we don't have to traverse down a layer deeper, to the individual elements, to establish identity of individual components.
- It requires only a single delimiter, rather than multiple, which has been a sticking point.

### Linked issues

- [https://github.com/ga4gh/refget/issues/8#issuecomment-773489450](https://github.com/ga4gh/refget/issues/8#issuecomment-773489450)
- [https://github.com/ga4gh/refget/issues/10](https://github.com/ga4gh/refget/issues/10)

### Known limitations

- We may need to enforce that arrays be the same length, at least for attributes that provide one value per sequence. Also, the order of items within each array must match in order for the attributes to correctly collate to a specific sequence.


## 2021-01-20 - Use the SAM specification v1 description of sequence names

### Decision

Sequence collections will use the [v1 SAM specification (subsection 1.2.1)](https://samtools.github.io/hts-specs/SAMv1.pdf#subsubsection.1.2.1) to define the allowable characters in a sequence name. As of January 2021, this is:

> Reference sequence names may contain any printable ASCII characters in the range[!-~]apart from backslashes, commas, quotation marks, and brackets—i.e., apart from ‘\ , "‘’ () [] {} <>’—and may notstart with ‘*’ or ‘=’.4

Should a wider GA4GH standard appear from [TASC issue 5](https://github.com/ga4gh/TASC/issues/5) the sequence collection spec will review this decision. The long-term vision of the sequence collections specification is to comply with any eventual GA4GH standard for sequence names.

### Linked issues

- [https://github.com/ga4gh/refget/issues/2](https://github.com/ga4gh/refget/issues/2)

### Known limitations

- There have been a handful of reports of old sequences with disallowed characters in the sequence name rows (`>`) of FASTA files, particularly from the microbiome community. These sequence collections would have to be changed to include only SAM-compatible ASCII characters, which could restrict usage of the sequence collections protocols and delay uptake.
