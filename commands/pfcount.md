Returns the approximated cardinality computed by the HyperLogLog data structure stored at the specified variable, which is 0 if the variable does not exist.

The HyperLogLog data structure can be used in order to count **unique** elements in a set using just a small constant amount of memory, specifically 12k bytes for every HyperLogLog (plus a few bytes for the key itself).

The returned cardinality of the observed set is not exact, but approximated with a standard error of 0.81%.

For example in order to take the count of all the unique search queries performed in a day, a program needs to call `PFADD` every time a query is processed. The estimated number of unique queries can be retrieved with `PFCOUNT` at any time.

Note: as a side effect of calling this function, it is possible that the HyperLogLog is modified, since the last 8 bytes encode the latest computed cardinality
for caching purposes. So `PFCOUNT` is technically a write command.

@return

@integer-reply, specifically:

* The approximated number of unique elements observed via `PFADD`.

@examples

```cli
PFADD hll foo bar zap
PFADD hll zap zap zap
PFADD hll foo bar
PFCOUNT hll
```

HyperLogLog representation
---

Redis HyperLogLogs are represented using a double representation: the *sparse* representation suitable for HLLs counting a small number of elements (resulting in a small number of registers set to non-zero value), and a *dense* representation suitable for higher cardinalities. Redis automatically switches from the sparse to the dense representation when needed.

The sparse representation uses a run-length encoding optimized to store efficiently a big number of registers set to zero. The dense representation is a Redis string of 12288 bytes in order to store 16384 6-bit counters. The need for the double representation comes from the fact that using 12k (which is the dense representation memory requirement) to encode just a few registers for smaller cardinalities is extremely suboptimal.

Both representations are prefixed with a 16 bytes header, that includes a magic, an encoding / version fiend, and the cached cardinality estimation computed, stored in little endian format (the most significant bit is 1 if the estimation is invalid since the HyperLogLog was updated since the cardinality was computed).

The HyperLogLog, being a Redis string, can be retrieved with `GET` and restored with `SET`. Calling `PFADD`, `PFCOUNT` or `PFMERGE` commands with a corrupted HyperLogLog is never a problem, it may return random values but does not affect the stability of the server. Most of the times when corrupting a sparse representation, the server recognizes the corruption and returns an error.

The representation is neutral from the point of view of the processor word size and endianess, so the same representation is used by 32 bit and 64 bit processor, big endian or little endian.

More details about the Redis HyperLogLog implementation can be found in [this blog post](http://antirez.com/news/75). The source code of the implementation in the `hyperloglog.c` file is also easy to read and understand, and includes a full specification for the exact encoding used for the sparse and dense representations.
