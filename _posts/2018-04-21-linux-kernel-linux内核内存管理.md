---
author: asmbits
comments: true
date: 2018-04-21 00:04:00 +0000
layout: post
title: linux内核内存管理
image: /assets/images/blog_back.jpg
categories:
- linux
tags:
- kernel
---

Memory Addresses
===

- logical address
    - segment:offset
        - the 'offset' is 'logical address'. 
- linear address
    - virtual address
        - if page unit is off, then the linear address is consistent as physical address. 
- physical address
    - represented as 32-bit or 36-bit unsigned integers.
- MMU(Memory Management Unit)

<!-- more -->

```

                 segmentation                    paging
logical address -------------> linear address ------------> physical address
                     unit                         unit

```


segmentation in Hardware
===

- segment selector

```
    15           3 2 1 0
    |    index   |TI|RPL|

# (index << 3) represent the offset in descriptor table
#
# if TI = 0 then
#   the segment selector in GDT( the address and size of GDT saved in gdtr )
# else TI = 1 then
#   the segment selector in LDT( the address and size of LDT saved in ldtr )
#
# important: LDT descriptor is defined in GDT, 
# so the oprand of lldt instruction is a selector in GDT 
#
# RPL : request privileges level  
```

**the segmentation selector in CS specifies the Current Privilegs Level(CPL) using its RPL**

- segment descriptors

```
 section description :

 |   7   |   6   |   5   |   4   |   3   |   2   |   1   |   0    |
 |7654321076543210765432107654321076543210765432107654321076543210|  <- 8-byte
 |--------========--------========--------========--------========|
high-------------------------------------------------------------low
 | base2 |   properties  |     section base1     |section limits1 |
 | 31..24|   look below  |         23..0         |      15..0     |
        /                \
       /                  \_________________________________________
      /                                                             \
     _________________________________________________________________
     | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
     | G |D/B| 0 |AVL|limits2(19..16)| p |  DPL  | S |     TYPE      |
     =================================================================
     |<-------------BYTE6----------->|<------------BYTE5------------>|
```

* code segment descriptor
* data segment descriptor
* Task state segment descriptor
    * only in GDT
    * type 9 or 11(busy) with S clear 
* local descriptor table descriptor
    * only in GDT
    * type 2 with S clear

```c
#define __USER_CS   (0x60 | RPL_0 | TI_0 )
#define __USER_DS   (0x68 | RPL_0 | TI_0 )
#define __KERNEL_CS (0x73 | RPL_3 | TI_0 )
#define __KERNEL_DS (0x7b | RPL_3 | TI_0 )

/*
the kernel just loads the value yielded by the __KERNEL_CS macro into the cs segmentation register.
note that: just can use 'retf' 'iret' 'jmp far' 'call far' to change 'cs'
*/
```


The linux GDT
===

**there is one GDT per CPU**


- GDT stored in 'cpu_gdt_table' array
  - Each GDT includes 18 segment descriptors and 14 null, unused, or reserved entries 
- the information using for 'lgdt' stored in 'cpu_gdt_descr' array
- the kernel defines a default LDT to be shared by most processes
  - stored in 'default_ldt' array
- TSS for each cpu store in 'init_tss' array
- Three Thread-Local Storage (TLS) segments 
  - this is a mechanism that allows multithreaded applications to make use of up to three segments containing data local to each thread. The set_thread_area( ) and get_thread_area( ) system calls, respectively, create and release a TLS segment for the executing process


Paging in Hardware
===

# regular paging

```
PDE
-------------------------------------
|11|10| 9| 8| 7| 6| 5| 4| 3| 2| 1| 0|
|________| |  |  |  |  |  |  |  |  |
     |     |  |  |  |  |  |  |  |  |--P:present
     |     |  |  |  |  |  |  |  |-----R/W:read/write (0 read, 1 read/write)
     |     |  |  |  |  |  |  |--------U/S:user/supervisor(0 supervisor, 1 user)
     |     |  |  |  |  |  |-----------PWT:write-through
     |     |  |  |  |  |--------------PCD:cache disabled
     |     |  |  |  |-----------------A:accessed
     |     |  |  |--------------------0:reserved (set to 0)
     |     |  |-----------------------PS:page size (0 indicates 4K bytes)
     |     |--------------------------G:global page (ignored)
     |--------------------------------Avail:available for system programmer's user


==============================================================================

PTE
-------------------------------------
|11|10| 9| 8| 7| 6| 5| 4| 3| 2| 1| 0|
|________| |  |  |  |  |  |  |  |  |
     |     |  |  |  |  |  |  |  |  |--P:present
     |     |  |  |  |  |  |  |  |-----R/W:read/write (0 read, 1 read/write)
     |     |  |  |  |  |  |  |--------U/S:user/supervisor(0 supervisor, 1 user)
     |     |  |  |  |  |  |-----------PWT:write-through
     |     |  |  |  |  |--------------PCD:cache disabled
     |     |  |  |  |-----------------A:accessed
     |     |  |  |--------------------D:dirty
     |     |  |-----------------------PAT:page table attribute index
     |     |--------------------------G:global page (ignored)
     |--------------------------------Avail:available for system programmer's user

```


- the 32 bits of a linear address devided into three fields
  - Directory, 31-22 bits
  - table,     21-12 bits 
  - offset,    11-0  bits

```
linear address   31           22 21           12 11             0
                |   directory   |     table     |     offset     |
                        |               |               |
                        |               |               |
                        |               |               |
                        |               |               |
                        | * 8           | * 8           |       
                        |               |               + ---> |======|
                        |               |               |      |      |
                        |               + ->|======| --------> |      |
                        |               |   |      |             page
                        + ->|======| -----> |      |             frame
                        |   |      |          page
                        |   |      |          table
  | cr3 | ----------------> |      |           |
     |                        page             |
     |                      directory          |
     |                         |               |
   phy addr                 phy addr        phy addr               
   of page                  of page         of page 
   directroy                table           frame
```

- present flag
  - set: contained in main memory
  - clear: generates Page Fault exception,and stores the linear address in 'cr2'
- accessed flag
  - set each time when accessing. used for swapping
- dirty flag
  - only to page table entries, set when modifing page frame.
- read/write flag
- user/supervisor flag
- PCD and PWT flags
  - about hardware cache
- page size flag
  - applies only to page directory entries
  - set, the entry refers to a 2MB or 4MB long page frame
- global flag
  - only to page table entries.
  - set, prevent pages from being flushed from TLB {and PGE(page global enable) of cr4 is set}

## extended paging

**allows page frames to be 4MB instead of 4KB**

*only when PSE flag of cr4 and Page size of PDE is set*

```             
linear address   31           22 21                             0
                |   directory   |             offset             |
                        |                        |
                        |                        |           
                        |                        |          4MB page
                        |                        |            frame
                        |          page          + ------->|=========|
                        |       directory        |         |         |
                        + ---->|=========| --------------->|         |
                        |      |         |
    | cr3 | ------------------>|         |
```

## the Physical Address Extension(PAE) Paging Mechanism

**address line expand from 32bits to 36bits**

>PAE is activated by setting the Physical Address Extension (PAE) flag in the cr4 control register.
>The Page Size (PS) flag in the page directory entry enables large page sizes (2 MB when PAE is enabled).

- Page directory(Table) entry size expands to 64 bits(24bits refers to page frame address, 12bits refers to attrubute.512 entries per page directory(table))
- the Page Directory Pointer Table (PDPT) consisting of four 64-bit entries(new introduced)
- cr3 control register contains a 27-bit PDPT base address field 
- when mapping 4KB page(PS flag in page directories cleared),32 bits linear address:
  - cr3: point to PDPT
  - bits 31-30: select PDPT entry(4 entries)
  - bits 29-21: select Page Directory entry(512 entries)
  - bits 20-12: select Page Table entry(512 entries)
  - bits 11-0 : offset in 4KB page frame
- when mapping 2MB page(PS flag in page directories set),32 bits linear address
  - cr3: point to PDPT
  - bits 31-30: select PDPT entry(4 entries)
  - bits 29-21: select Page Directory entry(512 entries)
  - bits 20-0:  offset in 2MB page frame
 
## Paging levels in some 64-bit architectures

|        | page size | address bits | paging level |   splite   |
|--------|-----------|--------------|--------------|------------|
| x86_64 | 4KB       |           48 |            4 | 9 9 9 9 12 |

## Hardware Cache

```
physical address devided into three parts
| tags | subset index | offset in cache line |
               |                 ^
               |                 |------------------------------------------
               ------------------------                                    |
                                      | compare tags                       |
                                      | with all cache line                |
        subset index of cache line    | in the same subset index           |
            | cache line 1 |          | if eque, then cache hit            |
            | cache line 2 |          | use offset, get data in cache line |
            | cache line 3 | <---------        |----------------------------
            |      ...     |
            | cache line n |  

```

- when cache hit:
  - read: just get the date from cache
  - write
    - write-through: update cache and memory
    - write-back: just update cache
      - when update memory:
        - instruction requires flush cache
        - hardware signal FLUSH occurs(ofen when cache miss)

## TLB(translation lookaside buffer)

>using for speeding up paging unit

- PCD
  - page cache disable: cache page frame or not
    - 0: enable
    - 1: disable  
- PWT
  - page write-through: write-through or write-back page frame
    - 0: write-back
    - 1: write-through  


paging in linux
===

**ia32**

|  page director(table) | 32bit2 with no PEA | 32bits with PEA | 64bits |
|-----------------------|--------------------|-----------------|--------|
| Page Global Directory | used               | used            | used   |
| Page Upper Directory  |                    |                 | used   |
| Page Middle Directory |                    | used            | used   |
| Page Table            | used               | used            | used   |


```

        | global dir | upper dir | middle dir | table | offset |
           |              |             |           |         |
           |              |             |           |         |
           |              |             |           |         |
           |              |             |           |         + --> |=======|
           |              |             |           |         |     |       |
           |              |             |           +->|===|------> |       |
           |              |             |           |  |   |         page
           |              |             +-> |=====|--> |   |         frame
           |              |             |   |     |     page
           |              +-> |=======| --> |     |     table
           |              |   |       |      page
           +--> |=======| --> |       |      middle
           |    |       |      page          directory
cr3 ----------> |       |      upper
                 page          directory
                 global
                 directory    

```

## line address fields

- PAGE_SHIFT
  - 12bits
  - size a tables entry can map
  - page size(PAGE_SIZE)
  - PAGE_MASK 0xfffff000
- PMD_SHIFT
  - length in bits of Offset and Table fields 
  - size a middel directory entry can map
    - PAE disable
      - 4MB, PMD_MASK 0xffc00000(if large page enable, large page size 4MB)
    - PAE enable
      - 2MB, PMD_MASK 0xffe00000(if large page enable, large page size 2MB)
- PUD_SHIFT
  - size a upper directory entry can map
  - PUD_SHIFT == PMD_SHIFT on 80x86
- PGDIR_SHIFT
  - size a page global directory entry can map
    - PAE disable
      - 4MB, PGDIR_MASK 0xffc00000
    - PAE enable
      - 1GB, PGDIR_MASK 0xc0000000 

-PTRS_PER_PTE, PTRS_PER_PMD, PTRS_PER_PUD, and PTRS_PER_PGD
  - entries in the Page Table, Page Middle Directory, Page Upper Directory, and Page Global Directory
    - PAE disable: 1024, 1, 1, 1024 
    - PAE enable:  512, 512, 1, 4

*The pte_present macro yields the value 1 if either the Present flag or the Page Size flag of a Page Table entry is equal to 1, the value 0 otherwise. Recall that the Page Size flag in Page Table entries has no meaning for the paging unit of the microprocessor; the kernel, however, marks Present equal to 0 and Page Size equal to 1 for the pages present in main memory but without read, write, or execute privileges. In this way, any access to such pages triggers a Page Fault exception because Present is cleared, and the kernel can detect that the fault is not due to a missing page by checking the value of Page Size.*

## Physical memory layout

```
        ___________________
        |_________________|                          _
        | highest usable  |         || mun_physpages  |
        | page frame      |         || max_pfn        |
        |_________________| <-------|| highend_pfn    |
        |                 |         |                 |
        |                 |         |                 |
        |                 |         |                 |
        |                 |         |                 |
        |                 |         |- totalhigh_pages|
        |                 |         |                 |
        |                 |         |                 |- totalram_pages
        |                 |         |                 |
        |                 |         |                 |
        |                 |         |                 |
        |_________________|         || highstart_pfn  |
        |_________________| <-------|| min_low_pfn    |
        |                 | <-------| max_low_pfn     |
        |-----------------|                           |
        | kernel img      |                           |
        |-----------------|                           |
        | lowest usable   |                           |
        | page frame      |                           |
        |_________________|----------------------------
        |                 |
        |_________________|  
```

- Process Page Tables
  - Linear addresses from 0x00000000 to 0xbfffffff can be addressed when the process runs in either User or Kernel Mode.
  - Linear addresses from 0xc0000000 to 0xffffffff can be addressed only when the process runs in Kernel Mode.

- master kernel page global directory
  - never directky used by any process or kernel thread
  - this directory refers to kernel space is the model of all regular process

*kernel use provisional page tables by below:*

```nasm
movl  $swapper_pg_dir-0xc0000000, %eax      ;maybe swapper_pg_dir use linear address in kernel space
movl  %eax, %cr3                            ;set page table pointer
movl  %cr0, %eax
orl   0x80000000, %eax                      ;open PG flag
movl  %eax, %cr0
```


*final kernel page table*

- reinitialized swapper_pg_dir Page Global 
  - ram size less then 896MB
    - 2级页表,将内核镜像要使用的物理内存地址映射到0xc0000000 3GB以上的线性地址
    - 开启4MB大页,且使页表据global属性,即不会刷出TLB
  - ram size between 896MB and 4096MB
    - The best Linux can do during the initialization phase is to map a RAM window of size 896 MB into the kernel linear address space
    - To initialize the Page Global Directory, the kernel uses the same code as in the previous case.
  - ram size more than 4096MB
    - PAE handles 36-bit physical addresses, linear addresses are still 32-bit addresses
    - 3级页表,将page global directory的前三个entries不分配pmd页,第4个entry分配pmd页.并初始化物理地址的钱896MB
    - 开启2MB大页,且使页表据global属性,即不会刷出TLB

## hardware caches

- Intel microprocessors offers only two TLB-invalidating techniques:
  - All Pentium models automatically flush the TLB entries relative to non-global pages when a value is loaded into the cr3 register.
  - In Pentium Pro and later models, the invlpg assembly language instruction invalidates a single TLB entry mapping a given linear address

- lazy flush TLB

```c
//用于 lazy flush TLB 的数据结构

#define NR_CPUS 32        //根据实际cpu数量来

typedef struct s_cpu_tlbstate
{
  emun state { 
    TLBSTATE_OK ,
    TLBSTATE_LAZY
  } ;
  void *active_mm ;       //指向 memory_descriptor: cpu_vm_mask
} cpu_tlbstate ;

/* 

                            CPU running kernel thread
                                        |
                         yes            |            no
                     ------------------------------------------
                     |                                        |
     cpu_tlbstate->state = TLBSTATE_LAZY           if other cpu flushed TLB
                     |                                       and
      when recieve   |                            they have the same active_mm
      a intercpu     |                                       and
      interrupt,     |                          the cpu_vm_mask include the cpu
      clear indecs of|                                        |
      cpu_vm_mask    |                              intercpu interrupt occurs
                     |                                        |
        return to regular process                             |
                     |                                    flush TLB
        page global directory change
                     |
       yes ----------------------- no
           |                     |
  flush TLB automatic      call __flush_tlb()
     by rewirte cr3            
*/
```

