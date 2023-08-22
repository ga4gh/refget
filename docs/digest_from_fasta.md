
# Compute a seqcol digest given a sequence collection

One of the most common uses of the seqcol specification is to compute a standard, universal identifier for a particular sequence collection. There are two ways to approach this: 1. Using an existing implementation; 2. Implement the seqcol digest algorithm yourself (it's not that hard).

## 1. Using existing implementations

### Reference implementation in Python

If working from within Python, you can use the reference implementation like this:

1. Install the seqcol package with some variant of `pip install seqcol`.
2. Build up your canonical seqcol object
3. Compute its digest:

```
seqcol.digest(seqcol_obj)
```



#### From a Canonical Sequence Collection

If you have a sequence collection in canonical structure, you can get its digest like this:



```
import seqcol

seqcol.digest()

