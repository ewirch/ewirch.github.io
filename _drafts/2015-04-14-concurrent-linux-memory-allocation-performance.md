---
layout: article
title: "Concurrent Linux Memory Allocation Performance/Scalability"
categories: articles
date: 2015-04-17
modified: 2015-04-17
tags: [tags]
image:
  feature: 
  teaser: /2014/03/stutter.jpg
  thumb: 
ads: false
comments: true
---

Teaser:
    - Allocating 600MB takes 1s. But allocating 600MB for each thread in a 8 thread group takes 20 times as long?! Is this a bug?

FIXME: kernel number

Start situation:
    - process loads big files (600MB) multiple times to memory
    - naive assumption: as long as the thread count doesn't exceed the core count, it should be not slower
    - reality: much slower than loading only one file. even slower than one thread loading x files?
    - What are the reasons and how can this be fixed?

Overview Allocation Process
    - multiple (software) layers Prog -> libc -> kernel
    - I'll talk about glibc 2.21, other libc implementations can do this differently. glibc as uses currently ptmalloc2 "This is a version (aka ptmalloc2) of malloc/free/realloc written by Doug Lea and adapted to multiple threads/arenas by Wolfram Gloger."
    - process of dynamically allocating memory
        - prog asks for a memory block (malloc())
        - a arena is picked
            - see post on description of arenas
            - malloc tries to lock the arena which was used the last time by this thread. if it fails, it continues to try-lock other arenas until the lock was successful. if none found, arena is created.
            - this approach high parallelization, but some vmem-overhead, and increased fragmentation.
        - libc decides if it should be mmapped or placed into heap
            - It uses some settings to decide:
                - mmap active? (MALLOC_MMAP_THRESHOLD_>0)
                - requested block size >= mmap_thresthold
                - is there enough free space in arena ?
                - is count of mmaped blocks smaller as MALLOC_MMAP_MAX_
        - calling mmap or retrieving a block from arena
            - if decision is to retrieve from arena (because the block doesn't meet the requirements to be mmaped)
                - check free space in arena
                - increase arena size if it's too small -> system call (mprotect())
                - if arena reached the maximum size (64MB on 64bit systems), create a new arena -> system call (mmap())
            - else call mmap -> system call (mmap())

Memory Pages Availability
    - memory 'allocation' via mmap() or brk() doesn't occupy any real memory. the kernel just sets some flags internally that the section has been allocated.
    - the real work is done when the memory section is actually accessed
    - this is done per-page. (typical page size for 64bit systems: 4096, call `getconf PAGESIZE`)
    - page is backed by physical memory and zeroed (filled with zeros).

!    - simple test: allocate pages without accessing them until out ouf memory.
!    - simple test: allocate pages with writing 1 byte (reading?) until out of memory

Bottlenecks
