# **GA4GH seqcol user stories: Practical applications for sequence collection management**

## **Introduction**

The GA4GH Sequence Collections (seqcol) specification provides a standardized solution for identifying and managing collections of reference sequences in genomics workflows. This document presents user stories for eleven key use cases, demonstrating how seqcol addresses real-world challenges in genomic data management, analysis, and sharing.

## **Use case 1: Find all sequences within a collection**

**What this accomplishes**

Researchers and bioinformaticians need to retrieve complete information about all sequences in a reference genome or collection. This includes chromosome names, lengths, and sequence digests that comprise a specific genome assembly.

**User story**

**As a** bioinformatician preparing an analysis pipeline  
 **I want** to retrieve all sequences contained in a specific genome assembly  
 **So that** I can configure my tools with the correct chromosome names and sizes for proper alignment and analysis

**Concrete examples**

* Setting up a GATK variant calling pipeline that needs to know all chromosomes in GRCh38  
* Configuring a STAR aligner index build that requires chromosome names and lengths  
* Quality control scripts verifying that all expected sequences are present in a reference

**How seqcol solves this**

The seqcol specification provides the `/collection/{digest}` endpoint with different detail levels. When called with `level=2`, it returns the complete canonical representation of the collection, including:

* `names`: An array of sequence names (e.g., \["chr1", "chr2", "chr3"\])  
* `lengths`: An array of sequence lengths in base pairs  
* `sequences`: An array of refget sequence identifiers

These arrays are collated, meaning the first element of each array corresponds to the same sequence. In other words, the first item in the names list matches the first length and the first sequence ID. This provides all the information needed to understand the complete content of a reference genome.

Sequence collections are identified by content-derived digests, meaning each collection gets a unique fingerprint based on what's actually in it. If you have that fingerprint, you always get back exactly the same collection with no mistakes or missing pieces.

## **Use case 2: Compare between collections to describe compatibility**

**What this accomplishes**

Determines whether two sequence collections are compatible for joint analysis, identifying shared and unique sequences between different genome versions or assemblies. This is critical for data integration and cross-study comparisons.

**User story**

**As a** genomics researcher analyzing data from multiple studies  
 **I want** to compare sequence collections used in different datasets  
 **So that** I can determine if the data can be merged or requires coordinate conversion

**Concrete examples**

* Comparing UCSC vs NCBI versions of GRCh38 (same assembly, different providers)  
* Validating that collaborators are using the exact same flavor of a reference genome  
* Checking if a "cleaned" genome (with ambiguity codes removed) matches the original

**How seqcol solves this**

The specification provides the `/comparison/{digest1}/{digest2}` endpoint that returns detailed compatibility information:

**The comparison response provides 3 key pieces of information:**

* `attributes`: Shows which attributes (names, lengths, sequences, *etc.*) are present in collection A only, B only, or both  
* `array_elements`: Reports how many items are in each collection and how many they have in common  
* `a_and_b_same_order`: Indicates whether shared elements appear in the same order in both collections

This structured comparison allows researchers to determine multiple levels of compatibility:

* **Identical collections**: All attributes match exactly  
* **Subset relationships**: One collection contains all sequences from another  
* **Partial overlap**: Some sequences are shared but collections differ  
* **Order compatibility**: Shared sequences are in the same or different order

The comparison function provides the granular detail needed to make informed decisions about data integration strategies.

## **Use case 3: Find the "important" sequences**

**What this accomplishes**

Identifies primary chromosomes versus alternate contigs, unplaced scaffolds, or other ancillary sequences. This helps researchers focus on canonical sequences for their analysis while being aware of additional sequences.

**User story**

**As a** clinical genomicist performing variant interpretation  
 **I want** to identify the primary chromosomes (chr1-22, X, Y, M) versus alternate loci  
 **So that** I can focus clinical analysis on medically relevant sequences while excluding technical artifacts

**Concrete examples**

* Clinical exome analysis focusing on canonical chromosomes  
* Excluding alternate loci from germline variant calling  
* Identifying mitochondrial sequences for specialized analysis

**How seqcol solves this**

The base seqcol specification doesn't directly label sequences as "important," but it enables this use case through ancillary attributes:

1. **Non-inherent attributes**: Implementations can extend the schema with custom attributes like `is_primary` or `priority` that don't affect the digest, but provide this metadata

2. **Multiple collection definitions**: Providers could define separate collections for different subsets (e.g., "GRCh38\_primary" containing only primary chromosomes)

This allows each implementation to define importance according to their specific needs while maintaining interoperability through the core attributes.

## **Use case 4: Bootstrap a process**

**What this accomplishes**

Initializes bioinformatics pipelines with the correct reference sequences, automatically configuring tools and workflows based on a seqcol identifier. This eliminates manual configuration errors and ensures reproducibility.

**User story**

**As a** bioinformatics pipeline developer  
 **I want** to automatically configure my pipeline using a seqcol identifier  
 **So that** all tools use consistent reference sequences without manual intervention

**Concrete examples**

* Automatically downloading and indexing reference sequences for a new project  
* Initializing cloud-based analysis with the correct genome version  
* Setting up containerized workflows with reference data

**How seqcol solves this**

The seqcol specification enables automated pipeline bootstrapping through:

1. **Unambiguous identification**: A single seqcol digest uniquely identifies the exact set of sequences needed

2. **Structured retrieval**: The `/collection/{digest}` endpoint provides all necessary information:

   * Sequence names for creating interval files  
   * Lengths for generating size files  
   * Refget sequence digests for retrieving actual sequences  
3. **Integration with refget**: The sequence digests can be used with refget servers to retrieve the actual sequence content

4. **Validation**: The content-based digests ensure the downloaded sequences match exactly what was expected

This allows pipeline systems to take a seqcol identifier as input and automatically prepare all reference-dependent components without human intervention.

## **Use case 5: Validate analysis/submission input**

**What this accomplishes**

Ensures that submitted data or analysis inputs use the expected reference sequences, catching mismatches before they cause errors or incorrect results. This is crucial for maintaining data integrity in repositories and analysis platforms.

**User story**

**As a** database curator at NCBI/EBI  
 **I want** to validate that submitted genomic data matches declared reference sequences  
 **So that** I can prevent data corruption and ensure accurate downstream analysis

**Concrete examples**

* Validating VCF files before submission to ClinVar  
* Checking BAM file headers match the stated reference genome  
* Verifying CRAM reference compatibility before archival

**How seqcol solves this**

The seqcol specification enables validation through:

1. **Digest computation**: Files can be processed to extract their reference information and compute a seqcol digest

2. **Comparison endpoint**: The `/comparison` endpoint (both GET and POST variants) allows comparing:

   * Expected seqcol digest vs observed digest from the file  
   * A digest vs a user-provided collection structure  
3. **Detailed mismatch reporting**: The comparison response identifies:

   * Missing sequences (`a_only`)  
   * Extra sequences (`b_only`)  
   * Sequences with matching names but different lengths or content  
4. **Partial compatibility assessment**: Even if not identical, the comparison can determine if the submission might still be usable (e.g., uses a subset of expected sequences)

This systematic validation prevents common errors like using the wrong genome version or modified reference sequences.

## **Use case 6: Allow human-readable metadata lookup for sequence collections**

**What this accomplishes**

Provides human-friendly information about sequence collections, including organism, assembly version, provider, and other descriptive metadata. This makes it easier for researchers to understand and select appropriate references.

**User story**

**As a** researcher browsing available reference genomes  
 **I want** to see descriptive information about sequence collections  
 **So that** I can select the appropriate reference for my species and analysis needs

**Concrete examples**

* Displaying genome information in a web portal  
* Generating reports about reference genomes used in studies  
* Creating reference genome catalogs for institutional resources

**How seqcol solves this**

The seqcol specification addresses this through non-inherent attributes and schema extensibility:

1. **Non-inherent attributes**: These can include human-readable metadata that doesn't affect the digest:

   * Assembly name and version  
   * Organism/species information  
   * Provider or source  
   * Release dates  
   * Aliases (e.g., "hg38", "GRCh38")  
2. **Service info endpoint**: The `/service-info` endpoint provides information about what attributes the server supports (through the provided data model JSON schema)

3. **Collection retrieval**: When fetching a collection with `/collection/{digest}`, non-inherent attributes are returned alongside the core sequence information

4. **Schema flexibility**: Implementations can extend the schema to include whatever metadata fields are relevant to their users

There is no dedicated `/metadata` endpoint in the specification. Metadata is included directly in the collection objects as non-inherent attributes.

## **Use case 7: Reverse lookup**

**What this accomplishes**

Finds sequence collections based on characteristics like sequence names or content, rather than starting with a known collection identifier. This helps researchers identify which reference genomes include particular sequences of interest.

**User story**

**As a** researcher studying a specific genomic region  
 **I want** to find all reference assemblies containing my sequence of interest  
 **So that** I can identify appropriate references for comparative analysis

**Concrete examples**

* Finding all assemblies that include a specific alternate haplotype  
* Identifying genomes containing a particular mitochondrial sequence  
* Locating references with specific chromosome arrangements

**How seqcol solves this**

The seqcol specification provides limited support for this use case through the `/list/collection` endpoint:

**Current limitations:** The `/list` endpoint can only filter by level 1 digests of entire attributes, not individual elements. This means:

* You cannot search for collections containing a specific sequence  
* You can only find collections with an exact match of an entire attribute array

**What is possible:**

* If you have a complete set of sequences and compute their level 1 digest, you can find all collections with exactly that set of sequences  
* If you know the exact `names` array you're looking for, you can find collections with those exact names in that exact order

**What is NOT possible with current spec:**

* Finding all collections that contain chromosome 21  
* Finding all collections that include a specific mitochondrial sequence variant  
* Finding collections that contain any subset of sequences

## **Use case 8: Containing collection lookup**

**What this accomplishes**

This use case aims to identify parent or superset collections that contain all sequences from a given collection, helping researchers find more complete assemblies or understand relationships between genome versions.

**User story**

**As a** researcher working with a minimal reference set  
 **I want** to find complete genome assemblies that include all my sequences  
 **So that** I can upgrade to a more comprehensive reference while maintaining compatibility

**Concrete examples**

* Finding full genome assemblies that contain a clinical gene panel reference  
* Identifying complete assemblies that include specific organellar genomes  
* Discovering extended references that encompass core chromosome sets

**Current limitations**

**The current seqcol specification does not directly support this use case.**

What the specification provides:

* The `/comparison` endpoint can identify overlapping sequences between two known collections  
* The `/list` endpoint can only filter by complete attribute digests (not individual sequences)

What would be needed but is missing:

* No element-level search capability (e.g., "find collections containing these specific sequences")  
* No built-in subset/superset relationship detection

Implementing this use case with the current specification would require:

1. Using `/list/collection` to retrieve all available collections  
2. Fetching each collection's full details with `/collection/{digest}`  
3. Comparing each collection individually using the `/comparison` endpoint  
4. Analyzing the comparison results to identify subset relationships

## **Use case 9: Computing digests (checksums)**

**What this accomplishes**

Generates and validates digests (checksums)  for sequence collections, ensuring data integrity and enabling content-based verification. This provides a cryptographic guarantee that sequences haven't been corrupted or modified.

**User story**

**As a** data manager responsible for reference genome integrity  
 **I want** to compute and verify digests for sequence collections  
 **So that** I can ensure data hasn't been corrupted during storage or transfer

**Concrete examples**

* Validating reference genomes after downloading from repositories  
* Ensuring integrity of references in long-term storage  
* Verifying consistency across mirrored sites

**How seqcol solves this**

The seqcol specification is fundamentally built on content-based identifiers:

1. **Deterministic digest algorithm**: The specification defines a precise algorithm for computing digests:

   * Uses GA4GH standard sha512t24u digest algorithm  
   * Follows RFC-8785 JSON canonicalization  
   * Computes nested digests for attributes and top-level collection  
2. **Content-based verification**: Any change to the sequences, their names, or their order results in a different digest

3. **Multi-level validation**:

   * Individual sequence integrity via refget digests  
   * Attribute integrity via level 1 digests  
   * Collection integrity via top-level digest  
4. **Standardized computation**: The specification ensures any implementation computes identical digests for identical content

This built-in digest system means that if two collections have the same digest, they are guaranteed to have identical content. No additional digest system is needed beyond the seqcol identifier itself. Tools implementing seqcol include reference implementations in Python (e.g., the refget-py package) that can compute these digests from local sequence files.

## **Use case 10: Embed identifiers for provenance and reproducibility**

**What this accomplishes**

Enables tools and pipelines to embed seqcol identifiers in their outputs for provenance tracking, and allows annotation of data files with computed seqcol digests when the reference is unknown. This ensures that the exact reference sequences used in analysis are documented and traceable.

**User story**

**As a** software developer building genomics tools
**I want** to embed seqcol identifiers in my tool's outputs and compute them from input files
**So that** downstream tools can identify the exact reference used and maintain complete provenance

**Concrete examples**

* Embedding seqcol digest in VCF/BAM/CRAM file headers during variant calling or alignment
* Computing seqcol digest from an input file with unknown or undocumented reference
* Pipeline outputs that automatically track reference provenance across analysis steps
* Annotating legacy datasets with seqcol digests for improved data management

**How seqcol solves this**

The seqcol specification enables provenance tracking through multiple mechanisms:

1. **Lightweight identifiers**: Seqcol digests are compact strings that can be easily embedded in file headers, metadata fields, or database records

2. **Bidirectional workflow**:
   * **Forward direction**: Tools can embed a known seqcol digest when creating output files
   * **Reverse direction**: Tools can compute a seqcol digest from reference information in existing files (e.g., extracting chromosome names and lengths from BAM headers)

3. **Standard format**: The consistent digest format means any tool that encounters a seqcol identifier can query a seqcol server to retrieve complete reference information

4. **No central authority required**: Digests can be computed locally from reference files without needing to register with a central database, making it suitable for custom or private references

This allows developers to build tools that maintain reference provenance automatically, and enables users to document references for data that previously lacked clear provenance.
Downstream users who then make use of these pipeline outputs will be able to retrieve the exact sequences used (assuming it is a standardized collection hosted by a refget service).

## **Use case 11: Assess coordinate system compatibility**

**What this accomplishes**

Determines whether different datasets, annotation files, or visualizations can be used together based on coordinate system (name/length) compatibility, even if underlying sequences differ.
This is particularly important for genome browsers, annotation curation, and data integration where exact sequence content is less critical than positional consistency.

**User story**

**As a** genome browser developer or data curator
**I want** to check if my coordinate system is compatible with annotation files or other datasets
**So that** I can determine if annotations can be displayed, merged, or compared without coordinate transformation

**Concrete examples**

* Checking if a BED file can be displayed on a genome browser's reference genome
* Validating SNP annotations against target reference genome coordinates before import
* Determining if two datasets with different sequence content (e.g., soft-masked vs hard-masked) share coordinate systems
* Assessing whether published annotations can be applied to a locally-modified reference

**How seqcol solves this**

The seqcol specification enables coordinate system compatibility checking through specialized attributes and comparison capabilities:

1. **Coordinate-focused attributes**: The recommended `name_length_pairs` and `sorted_name_length_pairs` attributes capture just the coordinate system information:
   * `name_length_pairs`: Ordered coordinate system (preserves chromosome order)
   * `sorted_name_length_pairs`: Order-invariant coordinate system (for order-agnostic compatibility)

2. **Attribute-level comparison**: The `/comparison` endpoint provides detailed information about:
   * Whether names match between collections
   * Whether lengths match between collections
   * How many sequences are shared vs unique
   * Whether shared sequences are in the same order

3. **Flexible compatibility levels**: Users can determine different levels of compatibility:
   * **Strict compatibility**: Identical `name_length_pairs` digest means identical coordinate systems in same order
   * **Relaxed compatibility**: Matching `sorted_name_length_pairs` digest means same coordinates, possibly different order
   * **Partial compatibility**: Comparison shows subset relationships or overlapping coordinates

4. **Sequence-agnostic**: Because coordinate attributes are separate from sequence content, two references with different underlying sequences (e.g., with/without alternate contigs, soft-masked vs unmasked) can be recognized as coordinate-compatible

This is particularly valuable for genome browsers that need to quickly determine if annotation tracks can be displayed, and for data curators who need to assess whether annotations created on one reference can be safely applied to another.