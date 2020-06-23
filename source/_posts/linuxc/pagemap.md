---
title: Linux之 /proc/<pid>/pagemap 理解
type: linuxc
description: /proc/pid/pagemap对于一个进程虚拟地址到物理地址映射的描述，这里包含了该进程所有用到的虚拟地址空间的映射描述信息
date: 2020-06-22 19:10:42
---

# Linux之 /proc/<pid>/pagemap 理解

## 含义理解

```
10	 * /proc/pid/pagemap.  This file lets a userspace process find out which
11	   physical frame each virtual page is mapped to.  It contains one 64-bit
12	   value for each virtual page, containing the following data (from
13	   fs/proc/task_mmu.c, above pagemap_read):
14	
15	    * Bits 0-54  page frame number (PFN) if present  ## 如果虚拟内存地址被映射，那么Bits 0~54就表示实际物理地址的页面号
16	    * Bits 0-4   swap type if swapped
17	    * Bits 5-54  swap offset if swapped
18	    * Bit  55    pte is soft-dirty (see Documentation/vm/soft-dirty.txt)
19	    * Bit  56    page exclusively mapped (since 4.2)
20	    * Bits 57-60 zero
21	    * Bit  61    page is file-page or shared-anon (since 3.5)
22	    * Bit  62    page swapped
23	    * Bit  63    page present   # 如果虚拟内存地址被映射，那么该为会被置1
```

`/proc/pid/pagemap`对于一个进程虚拟地址到物理地址映射的描述，这里包含了该进程所有用到的虚拟地址空间的映射描述信息。`/proc/pid/pagemap`该文件
可以被看成一个由`64Bits`长度的数组，数组的每个元素对应进程中分配的一个虚拟地址空间，根据上面所述，如果该`64Bits`的数值第64bit被置1，那么说明该
虚拟地址已经被映射到实际的物理地址，而 `Bits 0~54`的值就是该实际物理地址的页面号。得到物理地址页面号 再加上页面内地址偏移量就可以得到该虚拟地址
对应的实际物理地址。

1. 虚拟地址页面号 = 虚拟地址 / 页面大小；
2. 虚拟地址业内便宜量 = 虚拟地址 % 页面大小；
3. 虚拟地址页内偏移量 和 实际物理地址页内偏移量是相同的；

_页面大小获取方法，函数接口 `sysconf (_SC_PAGESIZE)`，或者命令行 `getconf PAGESIZE`_

下面是阅读vpp源码碰到的一段代码，举例解释

```
u64 *
clib_mem_vm_get_paddr (void *mem, int log2_page_size, int n_pages)
{
  int pagesize = sysconf (_SC_PAGESIZE);
  int fd;
  int i;
  u64 *r = 0;

  if ((fd = open ((char *) "/proc/self/pagemap", O_RDONLY)) == -1)
    return 0;

  /*log2_page_size 的值这里是12，log2(4096) = 12，4096是PAGESIZE*/
  for (i = 0; i < n_pages; i++)
    {
      u64 seek, pagemap = 0;
      uword vaddr = pointer_to_uword (mem); /*获取指针mem的值*/
      mylog("vaddr = %lu",vaddr);
      /* 左移log2_page_size，我的理解是，
            如果n_pages=1，那么就是获取mem虚拟地址，
            如果n_pages=2，那么就是获取mem虚拟地址所处页面的 下一个页面 ，
            。。。
            也就是说，n_pages就是获取的虚拟地址连续页面的个数
      */
      vaddr += (((u64) i) << log2_page_size); 
      /*
        ((u64) vaddr / pagesize) 得到虚拟地址vaddr 所在的页面号,
        再 *sizeof (u64)，找到该地址对应的 /proc/pid/pagemap 中描述该地址的元素位置（因为每个虚拟地址在/proc/pid/pagemap
        中都用一个64Bits长度来描述，所以这里 乘以 sizeof (u64)）
      */
      seek = ((u64) vaddr / pagesize) * sizeof (u64);
      mylog("seek=%lu,log2_page_size=%d,vaddr=%lu,pagesize=%d\n",seek,log2_page_size,vaddr,pagesize);

      /*
        使用lseek把fd指向上述虚拟地址对应的 pagemap元素
      */
      if (lseek (fd, seek, SEEK_SET) != seek)
	goto done;

      if (read (fd, &pagemap, sizeof (pagemap)) != (sizeof (pagemap)))
	goto done;

      /*获取pagemap元素第63bit，如果置1，说明该地址已经被映射到实际物理地址*/
      if ((pagemap & (1ULL << 63)) == 0)
	goto done;

      /*获取 0~64Bits，得到该虚拟地址对应的物理地址页面号*/
      pagemap &= pow2_mask (55);
      /*把物理地址页面号存到vec结构中,注意这里没有添加地址偏移量*/
      vec_add1 (r, pagemap * pagesize);
    }

done:
  close (fd);
  if (vec_len (r) != n_pages)
    {
      vec_free (r);
      return 0;
    }
  return r;
}
```
   
## 手册解释

```
1	pagemap, from the userspace perspective
2	---------------------------------------
3	
4	pagemap is a new (as of 2.6.25) set of interfaces in the kernel that allow
5	userspace programs to examine the page tables and related information by
6	reading files in /proc.
7	
8	There are four components to pagemap:
9	
10	 * /proc/pid/pagemap.  This file lets a userspace process find out which
11	   physical frame each virtual page is mapped to.  It contains one 64-bit
12	   value for each virtual page, containing the following data (from
13	   fs/proc/task_mmu.c, above pagemap_read):
14	
15	    * Bits 0-54  page frame number (PFN) if present
16	    * Bits 0-4   swap type if swapped
17	    * Bits 5-54  swap offset if swapped
18	    * Bit  55    pte is soft-dirty (see Documentation/vm/soft-dirty.txt)
19	    * Bit  56    page exclusively mapped (since 4.2)
20	    * Bits 57-60 zero
21	    * Bit  61    page is file-page or shared-anon (since 3.5)
22	    * Bit  62    page swapped
23	    * Bit  63    page present
24	
25	   Since Linux 4.0 only users with the CAP_SYS_ADMIN capability can get PFNs.
26	   In 4.0 and 4.1 opens by unprivileged fail with -EPERM.  Starting from
27	   4.2 the PFN field is zeroed if the user does not have CAP_SYS_ADMIN.
28	   Reason: information about PFNs helps in exploiting Rowhammer vulnerability.
29	
30	   If the page is not present but in swap, then the PFN contains an
31	   encoding of the swap file number and the page's offset into the
32	   swap. Unmapped pages return a null PFN. This allows determining
33	   precisely which pages are mapped (or in swap) and comparing mapped
34	   pages between processes.
35	
36	   Efficient users of this interface will use /proc/pid/maps to
37	   determine which areas of memory are actually mapped and llseek to
38	   skip over unmapped regions.
39	
40	 * /proc/kpagecount.  This file contains a 64-bit count of the number of
41	   times each page is mapped, indexed by PFN.
42	
43	 * /proc/kpageflags.  This file contains a 64-bit set of flags for each
44	   page, indexed by PFN.
45	
46	   The flags are (from fs/proc/page.c, above kpageflags_read):
47	
48	     0. LOCKED
49	     1. ERROR
50	     2. REFERENCED
51	     3. UPTODATE
52	     4. DIRTY
53	     5. LRU
54	     6. ACTIVE
55	     7. SLAB
56	     8. WRITEBACK
57	     9. RECLAIM
58	    10. BUDDY
59	    11. MMAP
60	    12. ANON
61	    13. SWAPCACHE
62	    14. SWAPBACKED
63	    15. COMPOUND_HEAD
64	    16. COMPOUND_TAIL
65	    17. HUGE
66	    18. UNEVICTABLE
67	    19. HWPOISON
68	    20. NOPAGE
69	    21. KSM
70	    22. THP
71	    23. BALLOON
72	    24. ZERO_PAGE
73	    25. IDLE
74	
75	 * /proc/kpagecgroup.  This file contains a 64-bit inode number of the
76	   memory cgroup each page is charged to, indexed by PFN. Only available when
77	   CONFIG_MEMCG is set.
78	
79	Short descriptions to the page flags:
80	
81	 0. LOCKED
82	    page is being locked for exclusive access, eg. by undergoing read/write IO
83	
84	 7. SLAB
85	    page is managed by the SLAB/SLOB/SLUB/SLQB kernel memory allocator
86	    When compound page is used, SLUB/SLQB will only set this flag on the head
87	    page; SLOB will not flag it at all.
88	
89	10. BUDDY
90	    a free memory block managed by the buddy system allocator
91	    The buddy system organizes free memory in blocks of various orders.
92	    An order N block has 2^N physically contiguous pages, with the BUDDY flag
93	    set for and _only_ for the first page.
94	
95	15. COMPOUND_HEAD
96	16. COMPOUND_TAIL
97	    A compound page with order N consists of 2^N physically contiguous pages.
98	    A compound page with order 2 takes the form of "HTTT", where H donates its
99	    head page and T donates its tail page(s).  The major consumers of compound
100	    pages are hugeTLB pages (Documentation/vm/hugetlbpage.txt), the SLUB etc.
101	    memory allocators and various device drivers. However in this interface,
102	    only huge/giga pages are made visible to end users.
103	17. HUGE
104	    this is an integral part of a HugeTLB page
105	
106	19. HWPOISON
107	    hardware detected memory corruption on this page: don't touch the data!
108	
109	20. NOPAGE
110	    no page frame exists at the requested address
111	
112	21. KSM
113	    identical memory pages dynamically shared between one or more processes
114	
115	22. THP
116	    contiguous pages which construct transparent hugepages
117	
118	23. BALLOON
119	    balloon compaction page
120	
121	24. ZERO_PAGE
122	    zero page for pfn_zero or huge_zero page
123	
124	25. IDLE
125	    page has not been accessed since it was marked idle (see
126	    Documentation/vm/idle_page_tracking.txt). Note that this flag may be
127	    stale in case the page was accessed via a PTE. To make sure the flag
128	    is up-to-date one has to read /sys/kernel/mm/page_idle/bitmap first.
129	
130	    [IO related page flags]
131	 1. ERROR     IO error occurred
132	 3. UPTODATE  page has up-to-date data
133	              ie. for file backed page: (in-memory data revision >= on-disk one)
134	 4. DIRTY     page has been written to, hence contains new data
135	              ie. for file backed page: (in-memory data revision >  on-disk one)
136	 8. WRITEBACK page is being synced to disk
137	
138	    [LRU related page flags]
139	 5. LRU         page is in one of the LRU lists
140	 6. ACTIVE      page is in the active LRU list
141	18. UNEVICTABLE page is in the unevictable (non-)LRU list
142	                It is somehow pinned and not a candidate for LRU page reclaims,
143			eg. ramfs pages, shmctl(SHM_LOCK) and mlock() memory segments
144	 2. REFERENCED  page has been referenced since last LRU list enqueue/requeue
145	 9. RECLAIM     page will be reclaimed soon after its pageout IO completed
146	11. MMAP        a memory mapped page
147	12. ANON        a memory mapped page that is not part of a file
148	13. SWAPCACHE   page is mapped to swap space, ie. has an associated swap entry
149	14. SWAPBACKED  page is backed by swap/RAM
150	
151	The page-types tool in the tools/vm directory can be used to query the
152	above flags.
153	
154	Using pagemap to do something useful:
155	
156	The general procedure for using pagemap to find out about a process' memory
157	usage goes like this:
158	
159	 1. Read /proc/pid/maps to determine which parts of the memory space are
160	    mapped to what.
161	 2. Select the maps you are interested in -- all of them, or a particular
162	    library, or the stack or the heap, etc.
163	 3. Open /proc/pid/pagemap and seek to the pages you would like to examine.
164	 4. Read a u64 for each page from pagemap.
165	 5. Open /proc/kpagecount and/or /proc/kpageflags.  For each PFN you just
166	    read, seek to that entry in the file, and read the data you want.
167	
168	For example, to find the "unique set size" (USS), which is the amount of
169	memory that a process is using that is not shared with any other process,
170	you can go through every map in the process, find the PFNs, look those up
171	in kpagecount, and tally up the number of pages that are only referenced
172	once.
173	
174	Other notes:
175	
176	Reading from any of the files will return -EINVAL if you are not starting
177	the read on an 8-byte boundary (e.g., if you sought an odd number of bytes
178	into the file), or if the size of the read is not a multiple of 8 bytes.
179	
180	Before Linux 3.11 pagemap bits 55-60 were used for "page-shift" (which is
181	always 12 at most architectures). Since Linux 3.11 their meaning changes
182	after first clear of soft-dirty bits. Since Linux 4.2 they are used for
183	flags unconditionally.
```