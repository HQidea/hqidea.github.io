---
title: 'Linux: Native Memory Fragmentation and Process Size Growth'
date: 2026-02-14 10:49:55
tags:
---

> This post was originally published by kgibm on July 24, 2014 at [IBM developerWorks](https://www.ibm.com/developerworks/community/blogs/kevgrig/entry/linux_native_memory_fragmentation_and_process_size_growth?lang=en).

The default Linux native memory allocator on most distributions is [Glibc malloc](http://www.gnu.org/software/libc/manual/html_node/Unconstrained-Allocation.html#Unconstrained-Allocation) (which is based on ptmalloc and dlmalloc). Glibc malloc either allocates like a classic heap allocator (from sbrk or mmap'ed arenas) or directly using mmap, depending on a sliding threshold ([M_MMAP_THRESHOLD](http://man7.org/linux/man-pages/man3/mallopt.3.html)). In the former case, the basic idea of a [heap allocator](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.47.275) is to request a large block of memory from the operating system and dole out chunks of it to the program. When the program frees these chunks, the memory is not returned to the operating system, but instead is saved for future allocations. This generally improves the performance by avoiding operating system overhead, including system call time. Techniques such as [binning](http://g.oswego.edu/dl/html/malloc.html) allows the allocator to quickly find a "right sized" chunk for a new memory request.

The major downside of all heap allocators is [fragmentation](https://bugzilla.redhat.com/show_bug.cgi?id=843478) (compaction is not possible because pointer addresses in the program could not be changed). While heap allocators can coallesce adjacent free chunks, program allocation patterns, malloc configuration, and malloc heap allocator design limitations mean that there are likely to be free chunks of memory that are unlikely to be used in the future. These free chunks are essentially "wasted" space, yet from the operating system point of view, they are still active virtual memory requests ("held" by glibc malloc instead of by the program directly). If no free chunk is available for a new allocation, then the heap must grow to satisfy it.

In the worst case, with certain allocation patterns and enough time, resident memory will grow unbounded. Unlike certain Java garbage collectors, glibc malloc does not have a feature of heap compaction. Glibc malloc does have a feature of trimming ([M_TRIM_THRESHOLD](http://man7.org/linux/man-pages/man3/mallopt.3.html)); however, this only occurs with contiguous free space at the top of a heap, which is unlikely when a heap is fragmented.

Glibc malloc does not make it easy to tell if fragmentation is the cause of process size growth, versus program demands or a leak. The [malloc_stats](http://man7.org/linux/man-pages/man3/malloc_stats.3.html) function can be called in the running process to print free statistics to stderr. It wouldn't be too hard to write a JVMTI shared library which called this function through a static method or MBean (and this could even be loaded dynamically through [Java Surgery](https://www.ibm.com/developerworks/community/blogs/kevgrig/entry/accouncing_a_new_tool_and_methodology_java_surgery?lang=en)). More commonly, you'll have a core dump (whether manually taken or from a crash), and the malloc structures don't track total free space in each arena, so the only way would be to write a gdb python script that walks the arenas and memory chunks and calculates free space (in the same way as malloc_stats). Both of these techniques, while not terribly difficult, are not currently available. In general, native heap fragmentation in Java program is much less likely than native memory program demands or a leak, so I always investigate those first (using techniques described [elsewhere](http://www-01.ibm.com/support/docview.wss?uid=swg27039764&aid=1)).

If you have determined that native heap fragmentation is causing unbounded process size growth, then you have a few options. First, you can change the application by reducing its native memory demands. Second, you can tune glibc malloc to immediately free certain sized allocations back to the operating system. As discussed above, if the requested size of a malloc is greater than M_MMAP_THRESHOLD, then the allocation skips the heaps and is directly allocated from the operating system using mmap. When the program frees this allocation, the chunk is un-mmap'ed and thus given back to the operating system. Beyond the additional cost of system calls and the operating system needing to allocate and free these chunks, mmap has additional costs because it must be zero-filled by the operating system, and it must be sized to the boundary of the page size (e.g. 4KB). This can cause worse performance and more memory waste (ceteris paribus).

If you decide to change the mmap threshold, the first step is to determine the allocation pattern. This can be done through tools such as ltrace (on malloc) or SystemTap, or if you know what is causing most of the allocations (e.g. Java DirectByteBuffers), then you can [trace](https://www.ibm.com/developerworks/community/blogs/kevgrig/entry/tracking_directbytebuffer_allocations_and_frees_in_ibm_java?lang=en) just those allocations. Next, create a histogram of these sizes and choose a threshold just under the smallest yet most frequent allocation. For example, let's say you've found that most allocations are larger than 8KB. In this case, you can set the threshold to 8192:

```
MALLOC_MMAP_THRESHOLD_=8192
```

Additionally, glibc malloc has a limit on the number of direct mmaps that it will make, which is 65536 by default. With a smaller threshold and many allocations, this may need to be increased. You can set this to something like 5 million:

```
MALLOC_MMAP_MAX_=5000000
```

These are set as [environment variables](http://www-01.ibm.com/support/docview.wss?uid=swg21254153) in each Java process. Note that there is a trailing underscore on these variable names.

You can verify these settings and the number and total size of mmaps using a core dump, gdb, and glibc symbols:

```
(gdb) p mp_
$1 = {trim_threshold = 131072, top_pad = 131072, mmap_threshold = 4096,
      arena_test = 0, arena_max = 1, n_mmaps = 1907812, n_mmaps_max = 5000000,
      max_n_mmaps = 2093622, no_dyn_threshold = 1, pagesize = 4096,
      mmapped_mem = 15744507904, max_mmapped_mem = 17279684608, max_total_mem = 0,
      sbrk_base = 0x1e1a000 ""}
```

In this example, the threshold was set to 4KB (mmap_threshold), there are about 1.9 million active mmaps (n_mmaps), the maximum number is 5 million (n_mmaps_max), and the total amount of memory currently mmap'ped is about 14GB (mmapped_mem).

There is also [some evidence](https://sourceware.org/bugzilla/show_bug.cgi?id=11261) that the [number of arenas](https://www.ibm.com/developerworks/community/blogs/kevgrig/entry/linux_glibc_2_10_rhel_6_malloc_may_show_excessive_virtual_memory_usage?lang=en) can contribute to fragmentation.
