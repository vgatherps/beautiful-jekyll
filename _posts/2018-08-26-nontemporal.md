---
layout: post
title: Nontemporal Memory Access, part 1
subtitle: Nontemporal stores
toc: true
---

Have you ever looked at code reading/writing to a large or infrequently used datastructure and thought "What a waste of the cache?".
Look no further than nontemporal memory operations for all your cache-bypassing needs.

Still in progress, but will still contain interesting code and charts while I'm battling my nemesis web-dev trying to make this pretty

### Nontemporal memory access in intel-x86
Modern x86 chips <sup><a href="#fnsse" id="ref_sse">1</a></sup> contain load and store operations which completely bypass the cache, usually described as nontemporal
memory operations. 

Nontemporal loads require that one is loading from write-combining memory, which requires a kernel module to easily access
in linux userspace. For this article, I'll just focus on nontemporal stores and later talk about the module and ways to use
nontemporal loads as well

### Mechanics of nontemporal stores

While the Intel documentation is flush with details of how normal stores work, much less attention is given to the characteristics
of nontemporal stores. As a result, most of the findings here are inferred from timing data.
The full code is available [in this repo](https://github.com/vgatherps/nontemporal_stores), test features and size
are `#define` gated.

#### Basic cache tests

Before embarking on any more advanced analysis, it's useful to confirm that the nontemporal stores won't write-allocate
and therefore won't pollute the cache. Our benchmark loop will be:

``` c
uint64_t run_timer_loop(void) {

    shuffle_list();

    mfence();

    for (int i = 0; i < STORES_BEFORE; i++) {
        force_store(&large_prestore_buffer[i]);
    }

    for (int i = 0; i < NT_STORES_BEFORE; i++) {
        force_nontemporal_store(&large_prestore_buffer[i]);
    }

    mfence();

    uint64_t start = rdtscp();
    iterate_list();
    uint64_t end = rdtscp();
}
```
We'll try writing between 0 and 6000 cache lines of either normal stores or nontemporal stores, and time iterating a shuffed linked list <sup><a href="#fnlist" id="ref_list">2</a></sup>.
The results are:
![write_allocate]({{ "/img/write_allocate.png" }})

This is what we hoped for - write allocation from normal stores evicts our list from the cache, but nontemporal
stores don't write-allocate and hence don't evict our list. If this weren't true, none of this would really be worth anything.
The results might seem obvious, but it's worth confirming that nontemporal stores truly behave as expected on standard instead of
write-combining memory.

#### Interactions with normal stores

Nontemporal stores are documented as being unordered with respect to other stores as a performance optimization. How this plays out in practice
is important since nontemporal stores have a huge, albeit usually unobserved latency. If it's possible for nontemporal stores to block
normal store behavior outside of cases with hardware fences, one needs to be much more careful regarding their use as opposed to the case
where the stores simply disappear off into the memory bus.

To test this, we'll execute a series of nontemporal stores before a series of temporal stores, and measure any interference or performance impact. The inner timing code we run will be:

``` c
uint64_t run_timer_loop(void) {

    for (int i = 0; i < LINES; i++) {
#ifndef LINES_IN_CACHE
        clflush(&large_buffer[i]);
#else
        force_load(&large_buffer[i]);
#endif
    }

    for (int i = 0; i < LINES; i++) {
#ifndef NT_LINES_IN_CACHE
        clflush(&large_nontemporal_buffer[i]);
#else
        force_load(&large_nontemporal_buffer[i]);
#endif
    }

    mfence();

    uint64_t start = rdtscp();

    for (int i = 0; i < NT_LINES; i++) {
        force_nt_store(&large_nontemporal_buffer[i]);
    }

    for (int i = 0; i < NT_LINES; i++) {
        force_store(&large_buffer[i]);
    }

    // while rdtscp 'waits' for all preceding instructions to complete,
    // it does not wait for stores to complete as in be visible to
    // all other cores or the L3, but for the instruction to be retired
    // this makes it easy to benchmark the visible effects of nontemporal stores
    // on our stalls
    uint64_t end = rdtscp();
}
```

We can run a test trying each combination of NT_LINES and LINES from 0 to 100, increments of 5 each. The results are:
![normal_basic]({{ "/img/nontemporal_basic.png" }})

The results of that don't point to a single cliff or pointing performance gun, but confirm some rather expected results:

  * For data in the cache, we can basically store at full rate - the store buffer can clear faster than we can submit
  * Nontemporal stores have some performance penalty when executing many. In this test, we must write the whole line nontemporally which worsens throughput problems

Importantly, there doesn't seem to be any cost difference to subsequent stores after a series of nontemporal stores is executed. Contrast this with the case where
we omit an sfence (more on this later, but enforces store ordering) between the nontemporal and temporal stores:
![normal_sfence]({{ "/img/nontemporal_sfence_d1.png" }})

This won't be relevant except when writing multicore code, but this is a great example of blocking the store buffer. Since stores can't leave the buffer until
the nontemporal stores complete, they simply just stall.

#### Write-combining buffers

TODO, tl;dr is that you better write 64 bytes at once nontemporally

<sup id="fnsse">1. Nontemporal stores were introduced in SSE, and loads in SSE 4.1<a href="#ref_sse" title="Jump back to footnote 1 in the text.">↩</a></sup> 

<sup id="fnlist">2. Iterating a list help reduce variability by having to hit multiple lines in a prefetcher-invisible way<a href="#ref_list" title="Jump back to footnote 2 in the text.">↩</a></sup> 
