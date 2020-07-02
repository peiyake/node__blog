---
title: Pagemap Interface of Linux Explained
type: linuxc
description: trying to explain the pagemap interface which is used to explore the mapping information of physical memory.
date: 2020-07-02 19:10:42
---

[原文链接](https://blog.jeffli.me/blog/2014/11/08/pagemap-interface-of-linux-explained/)

As we know, Linux supports virtual memory. Actually, almost all modern general operating systems such as Solaris, Windows, Mac OS X support virtual memory. Every user space process in Linux has its own virtual address space. The virtual address will be translated to physical address by operating system finally. In fact, most CPU architectures such as x86 and arm provide hardware support for virtual to physical address translation with MMU. In that case, the translation is done by the cooperation of operating system(software) and CPU(hardware). In this post, I am trying to explain the pagemap interface which is used to explore the mapping information of physical memory.

I assume you are familiar with the basic memory management principles. At least you must know


* Terminologies like virtual page, page frame, page size
* How to locate an address within a page
* Basic administration knowledge of Linux such as proc file system
* Know how to figure out virtual addresses that are interesting to you of a process in the proc file system or with tools such as GDB and readelf

## Pagemap Interface Explained

Managing hardware is one of the operating system’s main responsibilitiles. OS kernel needs some data structures to manage the physical memory. Since those data structures lives in kernel space, they can’t be accessed from the user space directly. The pagemap interface which has been introduced since 2.6.25 allows page tables and related information to be examined from user space. The information is exposed as virtual files living in proc file system:

* /proc/$(pid}/pagemap
* /proc/kpagecount
* /proc/kpageflags

Let’s start to explore these files now.

## /proc/${pid}/pagemap

![pagemap](/images/pagemap2.png)

This file contains the map information between virtual pages and physical pages of a process. The mapping information between a virtual page and physical page is represented as a 64-bit long entry. The file is a virtual file which means that it does not exist in the disk. Even so, you can imagine that it consists of many 64-bit long records each of which contains the physical page frame number of a virtual memory page and some other attributes. For example, record 0, 1, 2 are for the first, second and third virtual page respectively. In other words, the records are indexed by the virtual page number. Fig-1: Format of Pagemap EntryFig-1: Format of Pagemap Entry Fig-1 is the format of pagemap entries. Please refer to the design doc of the pagemap interface to see the detail of the format. It should be noted that the entry’s content represent different information when page is present and swapped out.

## /proc/kpagecount

In linux, it is possible (and likely!) that a physical page is mapped to different virtual pages of different process. This kpagecount file contains the reference number of physical pages. It is also a virtual file in which each physical frame’s reference count is represent as a 64-bit integer and indexed by the physical frame number. For example, if you want to find second physical frame’s reference count, you can just need to simply read byte 8 to byte 15 of the kpagecount file. 

## /proc/kpageflags

This virtual file contains the flags of physical frames. The flags of each page frame are present in a 64-bit long entry which is also indexed by the physical frame number. That means it is accessed in the same way as the kpagecount file. Each bit in the entry present a flag. I won’t explore every flags here because they are explained clearly in the design doc. Even the entry is 64-bit long, only 23 bit are used so far.

## Parse the Pagemap Interface

Following is a Python script to parse the files of pagemap interface. It accepts 2 arguments, one is the process’ pid and the other is the virtual address. Remember to append the 0x prefix if you want to pass a hexadecimal address.

```
#!/usr/bin/python

import sys
import os
import binascii
import struct

def read_entry(path, offset, size=8):
  with open(path, 'r') as f:
    f.seek(offset, 0)
    return struct.unpack('Q', f.read(size))[0]

# Read /proc/$PID/pagemap
def get_pagemap_entry(pid, addr):
  maps_path = "/proc/{0}/pagemap".format(pid)
  if not os.path.isfile(maps_path):
    print "Process {0} doesn't exist.".format(pid)
    return

  page_size = os.sysconf("SC_PAGE_SIZE")
  pagemap_entry_size = 8
  offset  = (addr / page_size) * pagemap_entry_size

  return read_entry(maps_path, offset)

def get_pfn(entry):
  return entry & 0x7FFFFFFFFFFFFF

def is_present(entry):
  return ((entry & (1 << 63)) != 0)

def is_file_page(entry):
  return ((entry & (1 << 61)) != 0)
##########################################################

# Read /proc/kpagecount
def get_pagecount(pfn):
  file_path = "/proc/kpagecount"
  offset = pfn * 8
  return read_entry(file_path, offset)

##########################################################

# Read /proc/kpageflags
def get_page_flags(pfn):
  file_path = "/proc/kpageflags"
  offset = pfn * 8
  return read_entry(file_path, offset)


if __name__ == "__main__":
  pid = sys.argv[1]
  if sys.argv[2].startswith("0x"):
    addr = long(sys.argv[2], base=16)
  else:
    addr = long(sys.argv[2])

  entry = get_pagemap_entry(pid, addr)
  pfn = get_pfn(entry)
  print "PFN: {}".format(hex(pfn))
  print "Is Present? : {}".format(is_present(entry))
  print "Is file-page: {}".format(is_file_page(entry))
  print "Page count: {}".format(get_pagecount(pfn))
  print "Page flags: {}".format(hex(get_page_flags(pfn)))

```

Save the script as v2pfn.py. Usage:

```
sudo ./v2pfn $PID $ADDRESS
```

Of course, this script does not cover everything of the pagemap interface. What I want to show here is how to read the information of a physical frame from the interface. It is very to extend if you want to parse other flags. 

## Conclusion

Pagemap interface is a quite simple tool to learn more about how Linux manage the physical memory. Hope this post can be helpful to understanding it. However, pagemap interface does NOT expose the origin PTE entry. This is kind of pity because PTE is very helpful to understand how virtual addresses are translated to physical address by CPU. In next post, I will show you how to inspect the raw PTE entry with SystemTap. Stay tuned.