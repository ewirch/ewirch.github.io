---
layout: article
title: "Linux Process Memory Layout"
categories: articles
date: 2013-11-20
modified: 2013-11-20
tags: [java, core, linux, memory]
image:
  feature: 
  teaser: /2013/11/memory-layout.jpg
  thumb: 
ads: false
comments: true
---

This article describes how the memory structure of each Linux process does look like.


{% include toc.html %}

Each Linux process starts with several memory blocks. A code block to hold the executable code of the program, stack blocks (one for each thread) and several data blocks (one for constants of the program, one for dynamic usage). Sometimes those blocks are called segments. In case of the data blocks we'll call them arenas in this article. This initial arena block is called the main arena. You'll see it named as "`[heap]`" in the contents of `/proc/xxx/maps` of any Linux process (replace xxx by PID).


## Basics

A Linux program can use the system functions `brk()`/`sbrk()` to change the size of it's main arena. It can also use `mmap()` to get new arenas from the system. But usually progams will use a memory management library instead of the system functions. The memory management library which is used by default is the libc library. One can override this, but usually all linux processes use this library. It exports functions like `malloc()`, `realloc()`, `calloc()`, `free()` to the program. The program uses those to allocate and free memory. And the memory management library itself uses `brk()`, `sbrk()`, `mmap()` to allocate the memory from the system.

libc creates data structures inside of the arenas to split the arena in smaller blocks. We'll call this structures "heap". So arenas are huge blocks issued by the system to libc. Heaps are the data structures inside this arenas (yes, there are several heaps) to manage smaller blocks.

## Java/JNI memory layout
But a process doesn't have to use the libc functions. Java for example has it's own memory management functions. Java starts just like every other linux process. But then it uses `mmap()` to allocate memory for it's own Java heap (and for the other Java memory areas, like `PermGenSpace` for classes or code cache for the JIT compiler) according to the size settings ("`-Xmx`" and others). All the memory which is used by Java objects and classes is *not* managed by libc. But Java has also parts which are implemented in native (JNI) code. If this native code requires memory it will use libc functions. Also (JNI) libraries loaded by Java are just like the Java native part, they also use libc functions.

## Difference vss/rss
When using several Linux tools, which do report memory usage (`ps`/`top`), you will stumble upon the two TLAs: `vss`, `rss`. (aliases are: `vsz`, `rsz`) They mean:

* Virtual Set Size
* Resident Set Size

*[vss]: Virtual Set Size
*[Vss]: Virtual Set Size
*[vsz]: Virtual Set Size
*[rss]: Resident Set Size
*[Rss]: Resident Set Size
*[rsz]: Resident Set Size

They exist to name two different memory allocation types. **Vss** names the reserved address space. **Rss** names the physical allocated memory (simplefied, see [Details For The Hard Core Developer 1](#dfthcd1)). One could (theoretically) allocate 16 terabytes of address space without using even one byte of physical memory. Only when the program asks the system to allocate physical memory, physical memory is allocated (simplified, see [Details For The Hard Core Developer 2](#dfthcd2)). How does the system tell between address space allocation and physical memory allocation? When the process allocates memory from the system it passes access permission to `mmap()`:

{% highlight c %}
// mmap() syntax: void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);

// this will allocate 200 bytes without any access permissions at a
// starting address choosen by the system
void *block1 = mmap(NULL, 200, PROT_NONE, MAP_PRIVATE, 0, 0);
{% endhighlight %}

This is just a address space reservation. The address space starting at block1 and spanning 200 bytes will not be issued at any other `mmap()` call. How does a process allocate physical memory? By changing the access permissions:

{% highlight c %}
// syntax of mprotect(): int mprotect(const void *addr, size_t len, int prot);
mprotect(block1, 100, PROT_READ | PROT_WRITE); 
{% endhighlight %}

After this invocation the first 100 bytes will be available for read and write. The remaining 100 bytes will still be just reserved address space.

But why we need to reserve address space anyway? If you are curious, see [Details For The Hard Core Developer 3](#dfthcd3).

## How libc Manages Memory
As already said, the system allocates the main arena block for each process at process start up. Libc creates a heap data structure inside of this main arena block. This is the main heap. Libc supports multi threaded programs. So invoking the memory functions in parallel is possible. Like any other program, libc has to use locking to guarantee data structure integrity while several threads call libc functions at the same time. To increase performance, libc tries to reduce lock contention (lock blocking) by having one lock per heap and by creating more heaps. When a memory request arrives at libc, it tries to lock the heap which was last used by the thread. If the thread did not use any heap yet, libc tries to lock the main heap. If locking fails, libc tries the next existing heap. If all heaps have been tested, libc creates a new heap. A heap is created by requesting a new arena from the system (`mmap()`) and writing heap structures into it:

{% highlight c %}
void *arena;
// reserves 64MB address space. libc heaps start always at 64MB size.
arena = mmap(NULL, 64*1024*1024, PROT_NONE, MAP_PRIVATE, 0, 0);
// allocate necessary memory at the beginning of heap
mprotect(arena, 1000, PROT_READ | PROT_WRITE);
// after this invocation libc can use the first 1000 bytes of the reserved address space 
{% endhighlight %}

When a heap was successfully locked, libc tries to find a free block which does satisfy the requested size. If a block was found, it is returned. If no block was found, libc tries to increase the size of the heap. It can do this in two ways.

1. Allocating bytes in the reserved address space (by calling `mprotect()` and increasing the area which has read/write permissions in the arena). This can only succeed if there is still room to expand.
1. Increasing address space and allocating bytes there (By calling `mremap()`, which will assign more address space. Heaps are resized in 64MB steps.). This can only succeed if the address space bordering the end of the current arena is still free.
1. If both fail (arena is full, arena cannot be resized) a new heap will be created. So heaps are created in two cases: to satisfy memory needs, to reduce lock contention.

How can we use this knowledge to optimize the memory footprint of our program? Let's assume we have a process with 100 threads. And let's assume there is heavy load, so all 100 threads call libc at the same time. In this case libc will create 100 heaps, each of 64MB in size. (Vss will instantly increase to 6.4GB, but rss will remain low, because the physical memory is not allocated yet.) This sounds not so bad yet (rss is still low). But let's continue this horror scenario. Let's assume this heavy load led to full exhaustion of the available memory. All 100 heaps are full. But the load stopped and the used memory is freed. Nearly everything is freed just some small blocks remain (which are used as a thread local buffer for example). Unluckily these buffers were allocated late, so that they lay at the "end" of the heap. The heap is nearly empty, just at the end there is this small block. libc is capable of shrinking the heaps to return memory to the system. If the free block at the end of the heap is big enough, libc shrinks the heap. But since there is a used block, libc cannot do this. The heap cannot have gaps. So even if the memory is not used by the program the rss and vss of the program will remain at 6.4GB.

So what do we learn from this? Don't use more threads than necessary. After all not all of those threads can run truly parallel. The system is limited to a much smaller number of CPU cores. Waiting for resources is often compensated by having more threads. If it's network resources, try to switch to asynchronous processing so the thread can do some other work, while the network interface is processing data. This way you will be able to reduce the number of required threads. The other thing we learned: If you use buffers, do free those buffers sometimes (This only applies for unmanaged languages like C. Java is able to move it's memory blocks to reduce fragmentation.).

## /proc/xxx/maps
Everyone can have a look at the blocks issued by the sytem to the program. The file maps in the proc folder of each process contains this information. The contents look like this (example of bash process):

{% highlight bash %}
00400000-004e1000 r-xp 00000000 fc:00 524291                             /bin/bash
006e0000-006e1000 r--p 000e0000 fc:00 524291                             /bin/bash
006e1000-006ea000 rw-p 000e1000 fc:00 524291                             /bin/bash
006ea000-006f0000 rw-p 00000000 00:00 0
00ec7000-01224000 rw-p 00000000 00:00 0                                  [heap]
7fc285b08000-7fc285b14000 r-xp 00000000 fc:00 6419                       /lib/x86_64-linux-gnu/libnss_files-2.15.so
7fc285b14000-7fc285d13000 ---p 0000c000 fc:00 6419                       /lib/x86_64-linux-gnu/libnss_files-2.15.so
7fc285d13000-7fc285d14000 r--p 0000b000 fc:00 6419                       /lib/x86_64-linux-gnu/libnss_files-2.15.so
7fc285d14000-7fc285d15000 rw-p 0000c000 fc:00 6419                       /lib/x86_64-linux-gnu/libnss_files-2.15.so
7fc285d15000-7fc285d1f000 r-xp 00000000 fc:00 5919                       /lib/x86_64-linux-gnu/libnss_nis-2.15.so
7fc285d1f000-7fc285f1f000 ---p 0000a000 fc:00 5919                       /lib/x86_64-linux-gnu/libnss_nis-2.15.so
7fc285f1f000-7fc285f20000 r--p 0000a000 fc:00 5919                       /lib/x86_64-linux-gnu/libnss_nis-2.15.so
7fc285f20000-7fc285f21000 rw-p 0000b000 fc:00 5919                       /lib/x86_64-linux-gnu/libnss_nis-2.15.so
7fc285f21000-7fc285f38000 r-xp 00000000 fc:00 6429                       /lib/x86_64-linux-gnu/libnsl-2.15.so
7fc285f38000-7fc286137000 ---p 00017000 fc:00 6429                       /lib/x86_64-linux-gnu/libnsl-2.15.so
7fc286137000-7fc286138000 r--p 00016000 fc:00 6429                       /lib/x86_64-linux-gnu/libnsl-2.15.so
7fc286138000-7fc286139000 rw-p 00017000 fc:00 6429                       /lib/x86_64-linux-gnu/libnsl-2.15.so
7fc286139000-7fc28613b000 rw-p 00000000 00:00 0
7fc28613b000-7fc286143000 r-xp 00000000 fc:00 6433                       /lib/x86_64-linux-gnu/libnss_compat-2.15.so
7fc286143000-7fc286342000 ---p 00008000 fc:00 6433                       /lib/x86_64-linux-gnu/libnss_compat-2.15.so
7fc286342000-7fc286343000 r--p 00007000 fc:00 6433                       /lib/x86_64-linux-gnu/libnss_compat-2.15.so
7fc286343000-7fc286344000 rw-p 00008000 fc:00 6433                       /lib/x86_64-linux-gnu/libnss_compat-2.15.so
7fc286344000-7fc2867c2000 r--p 00000000 fc:00 531094                     /usr/lib/locale/locale-archive
7fc2867c2000-7fc286977000 r-xp 00000000 fc:00 6434                       /lib/x86_64-linux-gnu/libc-2.15.so
7fc286977000-7fc286b76000 ---p 001b5000 fc:00 6434                       /lib/x86_64-linux-gnu/libc-2.15.so
7fc286b76000-7fc286b7a000 r--p 001b4000 fc:00 6434                       /lib/x86_64-linux-gnu/libc-2.15.so
7fc286b7a000-7fc286b7c000 rw-p 001b8000 fc:00 6434                       /lib/x86_64-linux-gnu/libc-2.15.so
7fc286b7c000-7fc286b81000 rw-p 00000000 00:00 0
7fc286b81000-7fc286b83000 r-xp 00000000 fc:00 6432                       /lib/x86_64-linux-gnu/libdl-2.15.so
7fc286b83000-7fc286d83000 ---p 00002000 fc:00 6432                       /lib/x86_64-linux-gnu/libdl-2.15.so
7fc286d83000-7fc286d84000 r--p 00002000 fc:00 6432                       /lib/x86_64-linux-gnu/libdl-2.15.so
7fc286d84000-7fc286d85000 rw-p 00003000 fc:00 6432                       /lib/x86_64-linux-gnu/libdl-2.15.so
7fc286d85000-7fc286da9000 r-xp 00000000 fc:00 340                        /lib/x86_64-linux-gnu/libtinfo.so.5.9
7fc286da9000-7fc286fa8000 ---p 00024000 fc:00 340                        /lib/x86_64-linux-gnu/libtinfo.so.5.9
7fc286fa8000-7fc286fac000 r--p 00023000 fc:00 340                        /lib/x86_64-linux-gnu/libtinfo.so.5.9
7fc286fac000-7fc286fad000 rw-p 00027000 fc:00 340                        /lib/x86_64-linux-gnu/libtinfo.so.5.9
7fc286fad000-7fc286fcf000 r-xp 00000000 fc:00 6422                       /lib/x86_64-linux-gnu/ld-2.15.so
7fc2871b3000-7fc2871c1000 r--p 00000000 fc:00 265459                     /usr/share/locale-langpack/de/LC_MESSAGES/bash.mo
7fc2871c1000-7fc2871c4000 rw-p 00000000 00:00 0
7fc2871c6000-7fc2871cd000 r--s 00000000 fc:00 535727                     /usr/lib/x86_64-linux-gnu/gconv/gconv-modules.cache
7fc2871cd000-7fc2871cf000 rw-p 00000000 00:00 0
7fc2871cf000-7fc2871d0000 r--p 00022000 fc:00 6422                       /lib/x86_64-linux-gnu/ld-2.15.so
7fc2871d0000-7fc2871d2000 rw-p 00023000 fc:00 6422                       /lib/x86_64-linux-gnu/ld-2.15.so
7fff98beb000-7fff98c0c000 rw-p 00000000 00:00 0                          [stack]
7fff98d48000-7fff98d49000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
{% endhighlight %}

The columns are:

* Address range of the block
* access permissions
* offset into the file, if block is a memory mapped file
* device number (of mapped file)
* inode (of mapped file)
* name of block or name of file (if mapped)

The tool `pmap` can display the same information in a more readable format. About identifying single blocks, see: [Details For The Hard Core Developer 4](#dfthcd4).

## Tools
I've written a tool to read the maps of a java process and guess the role of the different blocks. The tool is committed here: [javaJniMemUsage.pl][script-github]. It runs intrusionless on a server with very limited prerequisites (only Perl and Linux).

{% highlight bash %}
javaJniMemUsage.pl [OPTIONS] PID|proc-maps-file
  OPTIONS
   -c - print output as CSV
   -h - print CSV header line before data line
  PID - process id of process to retrieve memory maps information.
  proc-maps-file - contains memory maps information. Can be directly /proc/PID/maps.
{% endhighlight %}

One can invoke the tool with a pid, in which case the tool will read `/proc/PID/maps`, or a file which contains a maps dump. Without any other parameters the tool will dump a human readable output.


Here is a sample output of a busy java application server:

{% highlight bash %}
7f2096bf9000 -      5066752 (   4M 852K), rw-p, 0,
7f20973b6000 -      5066752 (   4M 852K), rw-p, 0,
7f2097b73000 -      5066752 (   4M 852K), rw-p, 0,
7f2098330000 -      5066752 (   4M 852K), rw-p, 0,
7f2098aed000 -      5066752 (   4M 852K), rw-p, 0,
7f20bc2fc000 -      5066752 (   4M 852K), rw-p, 0,
7f20d8327000 -      5066752 (   4M 852K), rw-p, 0,
7f20f8327000 -      5066752 (   4M 852K), rw-p, 0,
7f21161fb000 -      2703360 (   2M 592K), rw-p, 0,
7f2116778000 -      2703360 (   2M 592K), rw-p, 0,
7f2116cf5000 -      2703360 (   2M 592K), rw-p, 0,
7f2117272000 -      2703360 (   2M 592K), rw-p, 0,
7f21177ef000 -      2703360 (   2M 592K), rw-p, 0,
7f2117d6c000 -      2703360 (   2M 592K), rw-p, 0,
7f211c551000 -      2703360 (   2M 592K), rw-p, 0,
7f211cce3000 -      2703360 (   2M 592K), rw-p, 0,
7f21205d6000 -         8192 (        8K), rw-p, 0,
7f21211ef000 -        12288 (       12K), ---p, 0,
7f21211f2000 -      2088960 (  1M 1016K), rw-p, 0, [stack:28219]
7f2121ed4000 -        16384 (       16K), rw-p, 0,
7f2122182000 -         4096 (        4K), rw-p, 0,
7f2122604000 -         8192 (        8K), rw-p, 0,
7f2122f6f000 -         8192 (        8K), rw-p, 0,
7f21236c6000 -        12288 (       12K), rw-p, 0,
7f2123b89000 -       151552 (      148K), rw-p, 0,
7f2128000000 -         4096 (        4K), r--p, 0,
7f212801f000 -      1540096 (   1M 480K), rw-p, 0,
7f2128d82000 -         4096 (        4K), ---p, 0,
7f2128d83000 -     10731520 (  10M 240K), rw-p, 0, [stack:28198]
7f212a058000 -         4096 (        4K), ---p, 0,
7f212a059000 -      1208320 (   1M 156K), rw-p, 0, [stack:28190]
7f212a180000 -       118784 (      116K), ---p, 0,
7f212a19d000 -       770048 (      752K), rw-p, 0,
7f212a259000 -      8003584 (   7M 648K), rw-p, 0,
7f212a9fb000 -        36864 (       36K), ---p, 0,
7f212aa04000 -       348160 (      340K), rw-p, 0,
7f212aa59000 -       159744 (      156K), rw-p, 0,
7f212aa80000 -       118784 (      116K), ---p, 0,
7f212aa9d000 -       770048 (      752K), rw-p, 0,
7f212ab59000 -      8003584 (   7M 648K), rw-p, 0,
7f212b2fb000 -        36864 (       36K), ---p, 0,
7f212b304000 -       348160 (      340K), rw-p, 0,
7f212b359000 -      3526656 (   3M 372K), rw-p, 0,
7f212b6b6000 -       667648 (      652K), ---p, 0,
7f212b759000 -         4096 (        4K), rw-p, 0,
7f212b75a000 -     17432576 (  16M 640K), rwxp, 0,
7f212c7fa000 -     32899072 (  31M 384K), rw-p, 0,
7f212f196000 -         8192 (        8K), rw-p, 0,
7f212ff6b000 -        86016 (       84K), rw-p, 0,
7f2130b39000 -       167936 (      164K), rw-p, 0,
7f2130ee7000 -        20480 (       20K), rw-p, 0,
7f213150c000 -        16384 (       16K), rw-p, 0,
7f2131747000 -       438272 (      428K), rw-p, 0,
7f21317b2000 -       512000 (      500K), rw-p, 0,
7f2131837000 -        12288 (       12K), ---p, 0,
7f213183a000 -      1060864 (    1M 12K), rw-p, 0, [stack:28189]
7f2131942000 -         4096 (        4K), rw-p, 0,
7f2131943000 -         4096 (        4K), r--p, 0,
7f2131944000 -         8192 (        8K), rw-p, 0,
7f2131948000 -         4096 (        4K), rw-p, 0,
7fff955d3000 -       135168 (      132K), rw-p, 0, [stack]

Java-Blocks =
       count=8
        addr=[660000000,664df0000,668650000,680000000,774220000,7755e0000,780000000,7eb830000]
         rss= 6G 108M 64K
         vsz= 6G 512M
sizeInMappedFiles =  142M 536K
sizeInSystemMappings =  8K
main-arena =  1G 528M 244K
stacks =
     1M
       count=8
         rss= 8M
         vsz= 8M 32K
  1016K
       count=72
         rss= 71M 448K
         vsz= 72M 288K
     8M
       count=179
         rss= 1G 408M
         vsz= 1G 408M 716K
libc arenas =
   128M
       count=18
        addr=[7f1f00000000,7f1f90000000,7f1fa8000000,7f1fb0000000,7f1fc8000000,7f1fd8000000,7f1fe0000000,7f1ff8000000,7f2000000000,7f2008000000,7f2010000000,7f2030000000,7f2048000000,7f2050000000,7f2058000000,7f2064000000,7f20dc000000,7f20fc000000]
         rss= 2G 192M 556K
         vsz= 2G 256M
    64M
       count=65
        addr=[7f1ef8000000,7f1f08000000,7f1f10000000,7f1f14000000,7f1f18000000,7f1f1c000000,7f1f20000000,7f1f28000000,7f1f2c000000,7f1f30000000,7f1f34000000,7f1f38000000,7f1f3c000000,7f1f40000000,7f1f44000000,7f1f48000000,7f1f4c000000,7f1f54000000,7f1f58000000,7f1f5c000000,7f1f60000000,7f1f64000000,7f1f6c000000,7f1f74000000,7f1f78000000,7f1f7c000000,7f1f80000000,7f1f84000000,7f1f88000000,7f1f8c000000,7f1f98000000,7f1f9c000000,7f1fa0000000,7f1fa4000000,7f1fb8000000,7f1fc0000000,7f1fd0000000,7f1fd4000000,7f1fe8000000,7f1ff0000000,7f1ff4000000,7f2018000000,7f201c000000,7f2020000000,7f2024000000,7f2028000000,7f202c000000,7f2038000000,7f203c000000,7f2040000000,7f2060000000,7f206c000000,7f2070000000,7f2084000000,7f20b8000000,7f20d4000000,7f20e4000000,7f20e8000000,7f20ec000000,7f20f4000000,7f2104000000,7f2108000000,7f2110000000,7f2118000000,7f2124000000]
         rss= 3G 861M 148K
         vsz= 4G 64M
unknown rss = 145M 616K, vsz =  146M 580K
sum rss = 15G 275M 28K, vsz = 16G 90M 356K
{% endhighlight %}

First it dumps a list of blocks it was not able to identify. (The format changes slightly. Instead of the end address, the size of the block is printed.) The size of these blocks is summed up at the bottom: "unknown rss". This should be a small number. After that, it dumps categories of blocks it did identify. Each category is grouped into paragraphs of same size. Each size paragraph contains the number of blocks found and the size sum occupied by those blocks. Size sum is always split into vsz (vss) and rss. Vsz includes rss, this is why it will be always bigger. Some size paragraphs print also addresses of the identified blocks. They can be usually ignored. "sum rss" and "sum vsz" finally is the sum of all the blocks. They should be the same like reported by `ps`/`top`.

In the sample output before, we see that the 15GB rss are mainly used in this categories:
Java takes 6GB, exactly as specified by java parameters (`-Xmx6g`).
Thread stacks consume 1.5GB!
And finally the native heaps consume 7.5GB split into 83 heaps and one main arena/heap.

## Details For The Hard Core Developer
1. Rss includes the size of shared libraries mapped into memory. While this really uses physical memory, this memory is shared between processes using the same shared library. So if one process loades a 5MB shared library, rss for this process will be 5MB + the process ofwn stuff. If 10 processes load the same 5MB shared library, the rss for each process will be 5MB + the process stuff. But only 5MB of physical memory was really spent. So rss is not exactly physical memory size.
{: #dfthcd1}
2. Even if the system grants "physical" memory to the process it does not waste the memory yet. Two processes could allocate each the size of the whole system memory (assumed there is enough swap space) without anything bad happening. The system splits the memory into pages. And only if a process starts to access the memory, the page where the access is inside is considered "dirty" and backed by physical memory.
{: #dfthcd2}
3. This is necessary so libc can get a huge block from the system, without blocking memory which is not yet used. But why should we use libc and not just allocate memory by using mmap() instead? There are several reasons: 1. The memory returned by mmap() is aligned at page boundaries. The page size is usually 4kB. So if we allocate 10 bytes the rest of the page address space (4086 bytes) will be wasted. Since the system manages memory in pages this means real memory will be wasted. So we need libc to manage small memory blocks with smaller alignments. 2. There is a limited number of blocks which the system can return using mmap(). Since some processes use millions of memory blocks this would exhause this resource very quickly.
{: #dfthcd3}
4. We can guess three types of memory blocks in Java programs. 1. the Java blocks, 2. libc heaps, 3. stacks. Java blocks are usually at the beginning of the list and are huge. Their size equals the specified block sizes (-Xmx, -XX:PermGenSpace). The size may be split into several blocks. Especially into two blocks where one has read/write access and the other no access permissions (this is the growing heap). libc heaps have a size of exactly 64MB (64*1024*1024MB) or a magnitude of that (128MB, 192MB, ...). Each heap block is also split in two blocks with different access permissions. If the heap is full, there is only one block. Stacks have also a noticeable size. They are usually a magnitude of exactly 1MB. 1MB and 8MB stacks have been those I've seen the most in Java programs. Stacks have something more special: stack blocks are always preceeded by a guarding block. The guarding block is the size of one page (4k) and has no access premissions. This is used to detect stack overflows. The guarding block preceeds the stack block because stacks grow from bigger addresses to smaller addresses. If the stack overflows, the code will access the address space of this block without access permissions.
{: #dfthcd4}
5. Further deep detail reading: <http://www.blackhat.com/presentations/bh-usa-07/Ferguson/Whitepaper/bh-usa-07-ferguson-WP.pdf>

[script-github]: https://github.com/ewirch/javaJniMemUsage/blob/master/javaJniMemUsage.pl
