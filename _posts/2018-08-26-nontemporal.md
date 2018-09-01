---
layout: post
title: Optimizing Cache Usage with Nontemporal Memory Access, part 1
subtitle: Nontemporal stores
toc: true
---

Have you ever looked at code reading/writing to a large or infrequently used datastructure and thought "What a waste of the cache?".
Look no further than nontemporal memory operations for all your cache-bypassing needs.

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

  * The costs scale about linearly with the number of stores in either dimension, and there's no performance wall or major stall
  * Nontemporal stores have some slight throughput penalty over normal store operations, albeit for rather limited store sequences

Importantly, there doesn't seem to be any cost difference to subsequent stores after a series of nontemporal stores is executed. Contrast this with the case where
we emit an sfence (more on this later, but enforces store ordering) between the nontemporal and temporal stores:
<a id="fence">![normal_sfence]({{ "/img/nontemporal_sfence_d1.png" }})</a>

This won't be relevant except when writing multicore code, but this is a great example of what happens when nontemporal stores block normal stores.
Eventually, normal stores can't issue any more since the store buffer fills up and the processor just stalls.

#### Write-combining buffers

Write combining buffers exist on the cpu to help coalesce stores into a single operation. On normal stores, this allows reads-for-ownership to by coalesced
on consecutive stores to the cache line, but otherwise is not fundamental to performance. For nontemporal stores, they play a much more fundamental role.

The memory bus will only transact in 8 or 64 byte blocks, so store blocks that don't write an entire cache line will themselves get split into many
bus transactions. One should must ensure that whenever writing data nontemporally, the whole cache line is written to in a block. We can run some benchmarks
on what sort of leeway one has when defining 'in a block', but first let's see how bad it is for performance to run many.

For this benchmark, we'll be running this code in the inner loop:

``` c
void force_nt_store(cache_line *a) {
    __m128i zeros = {0, 0}; // chosen to use zeroing idiom;
    // do 4 stores to hit whole cache line
    __asm volatile("movntdq %0, (%1)\n\t"
#if BYTES > 16
                   "movntdq %0, 16(%1)\n\t"
#endif
#if BYTES > 32
                   "movntdq %0, 32(%1)\n\t"
#endif
#if BYTES > 48
                   "movntdq %0, 48(%1)"
#endif
                   :
                   : "x" (zeros), "r" (&a->vec_val)
                   : "memory");
}
 
uint64_t run_timer_loop(void) {

    mfence();
    uint64_t start = rdtscp();

    for (int i = 0; i < 32; i++) {
        force_nt_store(&large_buffer[i]);
    }

    mfence();

    uint64_t end = rdtscp();
}
```

We initiate a series of nontemporal stores (possibly to partial cache lines), and then issue a strong fence to time how long they take to complete.
As expected, writing partial cache lines seriously hurts performance:

![write_combine]({{ "/img/write_combine.png" }})

### What can we do with this?

Since we'll need a special kernel module for performing proper nontemporal reads, there's not a whole lot that we can do with this.
Nontemporal stores could be useful for writing to low-priority threads on a different socket, but ordering via fences and <a href="#fence">the performance implications</a>
of them are tricky enough to deserve their own article. For the sake of this article, I'll write a fairly contrived example but the juicy applications don't come until we've got nontemporal
loads and multicore ordering sorted out.

Consider this situation: One has some function which receives a long message containing a user ID, looks up that user in some datastructure,
and forwards the result on to some different processing pipeline.
Furthermore, one must keep the last 200MB of received messages in a buffer to dump upon failure. We'll benchmark this scenario and see how using nontemporal stores can greatly reduce the cache pressure of this buffer and keep the lookup datastructure in cache. The basic outline of our test code will be:

``` c++
struct message {
    uint64_t id;
    char data[1024*8 - sizeof(uint64_t)];
};

std::map<uint64_t, uint64_t> lookup_map;
std::vector<message> message_buffer;
size_t message_buffer_ind;

void process_message(const message &m) {

    message &cpy_to = message_buffer[message_buffer_ind];
    message_buffer_ind++;
    if (message_buffer_ind == message_buffer.size()) {
        message_buffer_ind = 0;
    }

#ifdef NONTEMPORAL_COPY   
    nontemporal_cpy_message(m, cpy_to);
#else
    cpy_message(m, cpy_to);
#endif

   process_message_farther(m, lookup_map[m.id]);
}
```

We'll adjust the number of ids in the map (and in the message) and see how each performs. The results are:
![example]({{ "/img/example.png" }})

Exactly as we hoped. The nontemporal operations reduce cache pressure in the pointer-chasing tree lookup and we see a performance improvement.
Hopefully, this was enough to demonstrate that nontemporal operations can truly have a use in high performance applications, and we'll see how
much more one can use them for when we can load past the cache as well.




<sup id="fnsse">1. Nontemporal stores were introduced in SSE, and loads in SSE 4.1<a href="#ref_sse" title="Jump back to footnote 1 in the text.">↩</a></sup> 

<sup id="fnlist">2. Iterating a list help reduce variability by having to hit multiple lines in a prefetcher-invisible way<a href="#ref_list" title="Jump back to footnote 2 in the text.">↩</a></sup> 
