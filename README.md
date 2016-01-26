# datasketch

[![Build Status](https://travis-ci.org/ekzhu/datasketch.svg?branch=master)](https://travis-ci.org/ekzhu/datasketch)

datasketch gives you probabilistic data structures that can process
vary large amount of data super fast, with little loss of accuracy.

This package contains the following data sketches:

| Data Sketch   | Usage                                |
|---------------|--------------------------------------|
| MinHash       | estimate resemblance and cardinality |
| b-Bit MinHash | estimate resemblance                 |
| HyperLogLog   | estimate cardinality                 |
| HyperLogLog++ | estimate cardinality                 |

The following indexes for data sketches are provided to support 
sub-linear query time:

| Index         | For Data Sketch  | Supported Query Type |
|---------------|------------------|----------------------|
| LSH           | MinHash          | Radius (Threshold)   |

datasketch must be used with Python 2.7 or above and NumPy.
Scipy is optional, but with it the LSH initialization can be much faster.

## Install

To install datasketch using `pip`:

    pip install datasketch -U

This will also install NumPy as dependency.

## MinHash

MinHash lets you estimate the Jaccard similarity
(resemblance)
between datasets of
arbitrary sizes in linear time using a small and fixed memory space.
It can also be used to compute Jaccard similarity between data streams.
MinHash is introduced by Andrei Z. Broder in this
[paper](http://cs.brown.edu/courses/cs253/papers/nearduplicate.pdf)

```python
from hashlib import sha1
from datasketch import MinHash

data1 = ['minhash', 'is', 'a', 'probabilistic', 'data', 'structure', 'for',
        'estimating', 'the', 'similarity', 'between', 'datasets']
data2 = ['minhash', 'is', 'a', 'probability', 'data', 'structure', 'for',
        'estimating', 'the', 'similarity', 'between', 'documents']

m1, m2 = MinHash(), MinHash()
for d in data1:
	m1.digest(sha1(d.encode('utf8')))
for d in data2:
	m2.digest(sha1(d.encode('utf8')))
print("Estimated Jaccard for data1 and data2 is", m1.jaccard(m2))

s1 = set(data1)
s2 = set(data2)
actual_jaccard = float(len(s1.intersection(s2)))/float(len(s1.union(s2)))
print("Actual Jaccard for data1 and data2 is", actual_jaccard)
```

You can adjust the accuracy by customizing the number of permutation functions
used in MinHash.

```python
# This will give better accuracy than the default setting (128).
m = MinHash(num_perm=256)
```

The trade-off for better accuracy is slower speed and higher memory usage.
Because using more permutation functions means 1) more CPU instructions
for every hash digested and 2) more hash values to be stored.
The speed and memory usage of MinHash are both linearly proportional
to the number of permutation functions used.

![MinHash Benchmark](https://github.com/ekzhu/datasketch/blob/master/minhash_benchmark.png)

You can union two MinHash object using the `merge` function.
This makes MinHash useful in parallel MapReduce style data analysis.

```python
# The makes m1 the union of m2 and the original m1.
m1.merge(m2)
```

MinHash can be used for estimating the number of distinct elements, or cardinality.
The analysis is presented in [Cohen 1994](http://ieeexplore.ieee.org/stamp/stamp.jsp?arnumber=365694).

```python
# Returns the estimation of the cardinality of
# all elements digested so far.
m.count()
```

## MinHash LSH

Suppose you have a very large collection of datasets. Giving a query, which
is also a dataset, you want to find datasets in your collection 
that have 
Jaccard similarities above certain threshold, 
and you want to do it with many other queries. 
To do this efficiently, you can create a MinHash for every dataset,
and when a query comes, you
compute the Jaccard similarities between the query MinHash and all the
MinHash of your collection, and return the datasets that
satisfy your threshold.

The said approach is still an O(n) algorithm, meaning the query cost 
increases linearly with respect to the number of datasets.
A popular alternative is to use Locality Sensitive Hashing (LSH) index.
LSH can be used with MinHash to achieve sub-linear query cost - that is
a huge improvement.
The details of the algorithm can be found in 
[Chapter 3, Mining of Massive Datasets](http://infolab.stanford.edu/~ullman/mmds/ch3.pdf),

This package includes the classic version of MinHash LSH.
It is important to note that the query does not give you the exact result,
due to the use of MinHash and LSH. There will be false positives - datasets
that do not satisfy your threshold but returned, and false negatives - 
qualifying datasets that are not returned.
However, the property of LSH assures that datasets with higher Jaccard
similarities always have higher probabilities to get returned than datasets
with lower similarities.
Moreover, LSH can be optimized so that there can be a "jump"
in probability right at the threshold, making the qualifying datasets much
more likely to get returned than the rest.

```python
from hashlib import sha1
from datasketch import MinHash, LSH

data1 = ['minhash', 'is', 'a', 'probabilistic', 'data', 'structure', 'for',
        'estimating', 'the', 'similarity', 'between', 'datasets']
data2 = ['minhash', 'is', 'a', 'probability', 'data', 'structure', 'for',
        'estimating', 'the', 'similarity', 'between', 'documents']
data3 = ['minhash', 'is', 'probability', 'data', 'structure', 'for',
        'estimating', 'the', 'similarity', 'between', 'documents']

# Create MinHash objects
m1 = MinHash(num_perm=128)
m2 = MinHash(num_perm=128)
m3 = MinHash(num_perm=128)
for d in data1:
	m1.digest(sha1(d.encode('utf8')))
for d in data2:
	m2.digest(sha1(d.encode('utf8')))
for d in data3:
	m3.digest(sha1(d.encode('utf8')))

# Create an LSH index optimized for Jaccard threshold 0.5, 
# that accepts MinHash objects with 128 permutations functions
lsh = LSH(threshold=0.5, num_perm=128)

# Insert m2 and m3 into the index 
lsh.insert("m2", m2)
lsh.insert("m3", m3)

# Using m1 as the query, retrieve the keys of the qualifying datasets
result = lsh.query(m1)
print("Candidates with Jaccard similarity > 0.5", result)
```

The Jaccard similarity threshold must be set at initialization, and cannot
be changed. So does the `num_perm` parameter.
Similar to MinHash, higher `num_perm` can improve the accuracy of LSH,
but increase 
query cost, since more processing is required as the MinHash gets bigger.
Unlike MinHash, the benefit of higher `num_perm` seems to be limited for LSH - 
it looks like when `num_perm` becomes greater than the dataset cardinality,
both precision and recall starts to decrease.
I experimented with the 
[20 News Group Dataset](http://scikit-learn.org/stable/datasets/twenty_newsgroups.html),
which has an average cardinality of 193 (3-shingles). 
The average recall, average precision, and 90 percentile query time vs.
`num_perm` are plotted below. See the `benchmark` directory for the experiment and
plotting code.

![LSH Benchmark](https://github.com/ekzhu/datasketch/blob/master/lsh_benchmark.png)

There are other optional parameters that be used to tune the index:

```python
# Use defaults: threshold=0.5, num_perm=128, weights=(0.5, 0.5)
lsh = LSH()

# `weights` controls the relative importance between minizing false positive
# and minizing false negative when building the LSH.
# `weights` must sum to 1.0, and the format is 
# (false positive weight, false negative weight).
# For example, if minizing false negative (or maintaining high recall) is more
# important, assign more weight toward false negative: weights=(0.4, 0.6).
# Note: try to live with a small difference between weights (i.e. < 0.5).
lsh = LSH(weights=(0.4, 0.6))
```

## b-Bit MinHash

[b-Bit MinHash](http://research.microsoft.com/pubs/120078/wfc0398-liPS.pdf)
is created by Ping Li and Arnd Christian König.
It is a compression of MinHash - it stores only the lowest b-bits of each
minimum hashed values in the MinHash, allowing one to trade accuracy for
less storage cost.

When the actual Jaccard similarity, or resemblance, is large (>= 0.5),
b-Bit MinHash's estimation for Jaccard has very small loss of accuracy
comparing to the original MinHash.
On the other hand, when the actual Jaccard is small, b-Bit MinHash gives
bad estimation for Jaccard, and it tends to over-estimate.

![b-Bit MinHash Benchmark](https://github.com/ekzhu/datasketch/blob/master/b_bit_minhash_benchmark.png)

To create a b-Bit MinHash object from an existing MinHash object:

```python
from datasketch import bBitMinHash

# minhash is an existing MinHash object.
bm = bBitMinHash(minhash)
```

To estimate Jaccard similarity using two b-Bit MinHash objects:

```python
# Estimate Jaccard given bm1 and bm2, both must have the same
# value for parameter b.
bm1.jaccard(bm2)
```

The default value for parameter b is 1.
Estimation accuracy can be improved by keeping more bits -
increasing the value for parameter b.

```python
# Using higher value for b can improve accuracy, at the expense of
# using more storage space.
bm = bBitMinHash(minhash, b=4)
```

**Note:** for this implementation,
using different values for the parameter b
won't make a difference in the in-memory
size of b-Bit MinHash, as the underlying storage is a NumPy integer array.
However, the size of a serialized b-Bit MinHash is determined by the parameter
b (and of course the number of permutation functions in the original MinHash).

Because b-Bit MinHash only retains the lowest b-bits of the minimum hashed
values in the original MinHash, it is not mergable.
Thus it has no `merge` function.

## HyperLogLog

HyperLogLog is capable of estimating the cardinality (the number of
distinct values) of dataset in a single pass, using a small and fixed
memory space.
HyperLogLog is first introduced in this
[paper](http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf)
by Philippe Flajolet, Éric Fusy, Olivier Gandouet and Frédéric Meunier.

```python
from hashlib import sha1
from datasketch import HyperLogLog

data1 = ['hyperloglog', 'is', 'a', 'probabilistic', 'data', 'structure', 'for',
'estimating', 'the', 'cardinality', 'of', 'dataset', 'dataset', 'a']

h = HyperLogLog()
for d in data1:
  h.digest(sha1(d.encode('utf8')))
print("Estimated cardinality is", h.count())

s1 = set(data1)
print("Actual cardinality is", len(s1))
```

As in MinHash, you can also control the accuracy of HyperLogLog by changing
the parameter p.

```python
# This will give better accuracy than the default setting (8).
h = HyperLogLog(p=12)
```

Interestingly, there is no speed penalty for using higher p value.
However the memory usage is exponential to the p value.

![HyperLogLog Benchmark](https://github.com/ekzhu/datasketch/blob/master/hyperloglog_benchmark.png)

As in MinHash, you can also merge two HyperLogLogs to create a union HyperLogLog.

```python
h1 = HyperLogLog()
h2 = HyperLogLog()
h1.digest(sha1('test'.encode('utf8')))
# The makes h1 the union of h2 and the original h1.
h1.merge(h2)
# This will return the cardinality of the union
h1.count()
```

## HyperLogLog++

[HyperLogLog++](http://research.google.com/pubs/pub40671.html)
is an enhanced version of HyperLogLog by Google with the following
changes:
* Use 64-bit hash values instead of the 32-bit used by HyperLogLog
* A more stable bias correction scheme based on experiments
on many datasets
* Sparse representation (not implemented here)

HyperLogLog++ object shares the same interface as HyperLogLog.
So you can use all the HyperLogLog functions in HyperLogLog++.

```python
from datasketch import HyperLogLogPlusPlus

# Initialize an HyperLogLog++ object.
hpp = HyperLogLogPlusPlus()
# Everything else is the same as HyperLogLog
```

## Serialization

All data sketches supports efficient serialization
using Python's `pickle` module.
For example:

```python
import pickle

m1 = MinHash()
# Serialize the MinHash objects to bytes
bytes = pickle.dumps(m1)
# Reconstruct the serialized MinHash object
m2 = pickle.loads(bytes)
# m1 and m2 should be equal
print(m1 == m2)
```

Additionally, you can check the byte size of any data sketch using the
`bytesize` function.

```python
# Print out the serialized size of m1 in number of bytes.
print(m1.bytesize())
```
