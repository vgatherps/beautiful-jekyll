---
layout: post
title: Nontemporal Memory Access, part 1
subtitle: Nontemporal stores
toc: true
---

Have you ever looked at code reading/writing to a large or infrequently used datastructure and thought "What a waste of the cache?".
If so, 

### Nontemporal memory access in intel-x86
Modern x86 cpus come with the ability to store, and in some cases load, past the cache. The memory access instructions
usually go by nontemporal stores/loads, but sometimes go by streaming stores/loads. Having the ability to manage cache usage in
such a fashion is extremely helpful when dealing with large datasets that don't reside in the cache, or accessing infrequently-accessed
memory without polluting the cache.

Nontemporal loads require that one is loading from write-combining memory, which requires a kernel module to easily access
in linux userspace. For this article, I'll just focus on nontemporal stores and later talk about the module and ways to use
nontemporal loads as well

### Mechanics of nontemporal stores

#### Write-combining buffers

Write combining buffers do what the name implies - they combine multiple stores to the same cache line to reduce
bandwidth in the cache subsystem. For normal stores, this means that multiple stores can be serviced by a single RFO,

#### Cost of nontemporal memory ordering
