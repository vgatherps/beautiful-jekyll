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
Modern x86 chips[^1] contain load and store operations which completely bypass the cache, usually described as nontemporal
memory operations. 

Nontemporal loads require that one is loading from write-combining memory, which requires a kernel module to easily access
in linux userspace. For this article, I'll just focus on nontemporal stores and later talk about the module and ways to use
nontemporal loads as well

### Mechanics of nontemporal stores

While the Intel documentation is flush with details of how normal stores work, much less attention is given to the characteristics
of nontemporal stores. As a result, most of the findings here are inferred from timing data.

### Ordering and occupancy in the store buffer

Nontemporal stores are documented as being unordered with respect to other stores as a performance optimization. How this plays out in practice
is important since nontemporal stores have a huge, albeit usually unobserved latency. If it's possible for nontemporal stores to block
normal store behavior outside of cases with hardware fences, one needs to be much more careful regarding their use as opposed to the case
where the stores simply disappear off into the memory bus.

The store buffer exists to allow many stores, possibly speculative, to be seen as 'completed' be later loads before they're actually committed
to the cache[^2]. In short, some number of stores may reside in this buffer and be visible to subsequent loads before becoming visible to the rest
of the cache subsystem. In cases where nontemporal stores can block or ocupy the store buffer, the remaining space is how much 'leeway' we have until
the program stalls on a store.

The code we'll be using to measure various cases will be:

``` c

// I use defines here so there's never a possibility of speculation
// accidentally seeing the wrong instruction and having some
// side effect

#ifndef NT_LINES
#define NT_LINES 1
#endif

#ifndef LINES
#define LINES 1
#endif

{
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

    // cpuid is a serializing instruction, stronger than say mfence. Defined in the full code
    cpuid();

    uint64_t start = rdtscp();

    for (int i = 0; i < NT_LINES; i++) {
        force_nt_store(&large_nontemporal_buffer[i]);
    }

#ifdef FENCE
    sfence();
#endif

    for (int i = 0; i < NT_LINES; i++) {
        force_store(&large_buffer[i]);
    }

    // while rdtscp 'waits' for all preceding instructions to complete,
    // it does not wait for stores to complete as in be visible to
    // all other cores or the L3, but for the instruction to be retired
    // this makes it easy to benchmark the visible effects of nontemporal stores
    // on our stalls
    uint64_t end = rdtscp();

    cpuid();
    return  end > start ? end - start : 0;
}
```

The full code is available [in this repo](https://github.com/vgatherps/nontemporal_stores)

#### Write-combining buffers

Write combining buffers do what the name implies - they combine multiple stores to the same cache line to reduce
bandwidth in the cache subsystem. For normal stores, this means that multiple stores can be serviced by a single RFO,


[^1] Nontemporal stores were introduced in SSE, and loads in SSE 4.1

[^2] This is a massive simplification of store buffers and store-load forwarding
