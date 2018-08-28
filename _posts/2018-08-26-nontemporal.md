---
layout: post
title: Nontemporal Memory Access, part 1
subtitle: Nontemporal stores
toc: true
---

Have you ever looked at code reading/writing to a large or infrequently used datastructure and thought "What a waste of the cache?".
Look no further than nontemporal memory operations for all your cache-bypassing needs.

### Nontemporal memory access in intel-x86
Modern x86 chips contain load and store operations which completely bypass the cache, usually described as nontemporal
memory operations.

Nontemporal loads require that one is loading from write-combining memory, which requires a kernel module to easily access
in linux userspace. For this article, I'll just focus on nontemporal stores and later talk about the module and ways to use
nontemporal loads as well

### Mechanics of nontemporal stores

#### Write-combining buffers

Write combining buffers do what the name implies - they combine multiple stores to the same cache line to reduce
bandwidth in the cache subsystem. For normal stores, this means that multiple stores can be serviced by a single RFO,

#### Cost of nontemporal memory ordering
