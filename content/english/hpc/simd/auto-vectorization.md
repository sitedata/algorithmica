---
title: Auto-Vectorization
weight: 10
---

SIMD-parallelism is most often used for *embarrassingly parallel* computations: the kinds where all you do is apply some elementwise function to all elements of an array and write it back somewhere else. In this setting, you don't even need to know how SIMD works: the compiler is perfectly capable of optimizing such loops by itself — you just need to be aware that such optimization exists and that it usually yields a 5-10x speedup.

Doing nothing and relying on auto-vectorization is actually the preferred way of using SIMD. Whenever you can, you should always stick with the scalar code for its simplicity and maintainability. But often even the loops that seem straightforward to vectorize are not optimized because of some technical nuances. [As in many other cases](/hpc/compilation/contracts), the compiler may need some additional input from the programmer as he may know a bit more about the problem than what can be inferred from static analysis.

### Potential Problems

Consider the "a + b" example:

```c++
void sum(int *a, int *b, int *c, int n) {
    for (int i = 0; i < n; i++)
        c[i] = a[i] + b[i];
}
```

Let's step into a compiler's shoes and think about what can go wrong when this loop is vectorized.

**Array size.** If the array size is unknown beforehand, it may be that it is too small for vectorization to be beneficial in the first place. Even if it is sufficiently large, we need to insert an additional check for the remainder of the loop to process it scalar, which would cost us a branch.

To eliminate these runtime checks, use array sizes that are compile-time constants, and preferably pad arrays to the nearest multiple of the SIMD block size.

**Memory aliasing.** Even when array size issues are out of the question, vectorizing this loop is not always technically correct. For example, the arrays `a` and `c` can intersect in a way that their beginnings differ by a single position — because who knows, maybe the programmer wanted to calculate the Fibonacci sequence through a convolution this way. In this case, the data in the SIMD blocks will intersect and the observed behavior will differ from the one in the scalar case.

When the compiler can't prove that the function may be used for intersecting arrays, it has to generate two implementation variants — a vectorized and a "safe" one — and insert runtime checks to choose between the two. To avoid them, we can tell the compiler that we are that no memory is aliased by adding the `__restrict__` keyword:

```cpp
void add(int * __restrict__ a, const int * __restrict__ b, int n) {
    for (int i = 0; i < n; i++)
        a[i] += b[i];
}
```

The other way, specific to SIMD, is the "ignore vector dependencies" pragma. It is a general way to inform the compiler that there are no dependencies between the loop iterations:

```c++
#pragma GCC ivdep
for (int i = 0; i < n; i++)
    // ...
```

**Alignment.** The compiler also doesn't know anything about the alignment of these arrays and has to either process some elements at the beginning of these arrays before starting the vectorized section or potentially lose some performance by using [unaligned memory accesses](../moving).

To help the compiler eliminate this corner case, we can use the `alignas` specifier on static arrays and the `std::assume_aligned` function to mark pointers aligned.

**Checking if vectorization happened.**  In either case, it is useful to check if the compiler vectorized the loop the way you intended. You can either [compiling it to assembly](/hpc/compilation/stages) and look for blocks for instructions that start with a "v" or add the `-fopt-info-vec-optimized` compiler flag so that the compiler indicates where auto-vectorization is happening and what SIMD width is being used. If you swap `optimized` for `missed` or `all`, you may also get some reasoning behind why it is not happening in other places.

---

There are [many other ways](https://software.intel.com/sites/default/files/m/4/8/8/2/a/31848-CompilerAutovectorizationGuide.pdf) of telling the compiler what we meant exactly, but in especially complex cases — when inside the loop there are a lot of branches or some functions are called — it is easier to go down to the intrinsics level and write it yourself.
