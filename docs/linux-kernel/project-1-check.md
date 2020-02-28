### 問題一
轉出的 physical address 有的會超大（前面多一串80000000），於是我把 `pgd_val`, `pud_val`, `pmd_val`, `pte_val` 都印出來發現只有 `pte_val` 前面多這一串，於是就先去看了下 `pte_offset_kernel`

```
/*
 * the pte page can be thought of an array like this: pte_t[PTRS_PER_PTE]
 *
 * this function returns the index of the entry in the pte page which would
 * control the given virtual address
 */
static inline unsigned long pte_index(unsigned long address)
{
	return (address >> PAGE_SHIFT) & (PTRS_PER_PTE - 1);
}

static inline pte_t *pte_offset_kernel(pmd_t *pmd, unsigned long address)
{
	return (pte_t *)pmd_page_vaddr(*pmd) + pte_index(address);
}
```

發現很正常，和上幾層做的事都一樣，那看看 `pte_val`

```
#define pte_val(x)	native_pte_val(x)
```

```
static inline pteval_t native_pte_val(pte_t pte)
{
	return pte.pte;
}

static inline pteval_t pte_flags(pte_t pte)
{
	return native_pte_val(pte) & PTE_FLAGS_MASK;
}
```

而當我去找 `native_pte_val` 時發現下面有一個 `function pte_flags`，他從 `native_pte_val` 中取出 flag 的部分，也難怪直接取 `pte_val` 會包含 flag，我們需要把 flag 去掉，所以先去看看`PTE_FLAGS_MASK`

```
/* Extracts the PFN from a (pte|pmd|pud|pgd)val_t of a 4KB page */
#define PTE_PFN_MASK		((pteval_t)PHYSICAL_PAGE_MASK)

/* Extracts the flags from a (pte|pmd|pud|pgd)val_t of a 4KB page */
#define PTE_FLAGS_MASK		(~PTE_PFN_MASK)
```

看到這邊就恍然大悟，因為我們最後取物理位址是用
```
pte_val(*pte) & PAGE_MASK　+ vaddr & ~PAGE_MASK
```
但是看看上面幾層在取得物理位址時都是用
```
pxx_page_vaddr(*pxx) + index
```
其中 `pxx_page_vaddr` 都是在做
```
pxx_val(*pxx) & PTE_PFN_MASK
```
我們將 `pte_val` 的 flag 濾掉就好了～
```
pte_val(*pte) & PTE_PFN_MASK & PAGE_MASK　+ vaddr & ~PAGE_MASK
```

> PFN: page frame number


### 問題二 

如何直接訪問實體記憶體位址？

busybox 中的 devmem 可直接操作物理地址
<a href="https://github.com/brgl/busybox/blob/master/miscutils/devmem.c" target="_blank" rel="noopener">https://github.com/brgl/busybox/blob/master/miscutils/devmem.c</a>

其中他使用到 `mmap` 和 `/dev/mem`，透過 `mmap` 將 `/dev/mem` 的物理地址映射到 user space，我們就可以像操作虛擬地址般讀寫

```
Usage: devmem ADDRESS [WIDTH [VALUE]]

Read/write from physical address

        ADDRESS Address to act upon
        WIDTH   Width (8/16/...)
        VALUE   Data to be written
```

要實作出 devmem 有三個主要步驟

1 開啟 `/dev/mem`

```
fd = open("/dev/mem", argv[3] ? (O_RDWR | O_SYNC) : (O_RDONLY | O_SYNC));
```

2 將物理地址透過 `mmap` 映射到 user space

```
 /* void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset); */
map_base = mmap(0, MAP_SIZE, argv[3] ? (PROT_READ | PROT_WRITE) : PROT_READ, MAP_SHARED, fd, target & MAP_MASK);
```

3 算出地址後進行讀寫
```
virt_addr = map_base + (target & MAP_MASK);
// write
*(*type)virt_addr = write_val
// read
read_val = *(*type)virt_addr
```

#### 觀察 mmap

```
static int mmap_mem(struct file *file, struct vm_area_struct *vma)
{
	size_t size = vma->vm_end - vma->vm_start;
	phys_addr_t offset = (phys_addr_t)vma->vm_pgoff << PAGE_SHIFT;

	/* It's illegal to wrap around the end of the physical address space. */
	if (offset + (phys_addr_t)size - 1 < offset)
		return -EINVAL;

	if (!valid_mmap_phys_addr_range(vma->vm_pgoff, size))
		return -EINVAL;

	if (!private_mapping_ok(vma))
		return -ENOSYS;

	if (!range_is_allowed(vma->vm_pgoff, size))
		return -EPERM;

	if (!phys_mem_access_prot_allowed(file, vma->vm_pgoff, size,
						&vma->vm_page_prot))
		return -EINVAL;

	vma->vm_page_prot = phys_mem_access_prot(file, vma->vm_pgoff,
						 size,
						 vma->vm_page_prot);

	vma->vm_ops = &mmap_mem_ops;

	/* Remap-pfn-range will mark the range VM_IO */
	if (remap_pfn_range(vma,
			    vma->vm_start,
			    vma->vm_pgoff,
			    size,
			    vma->vm_page_prot)) {
		return -EAGAIN;
	}
	return 0;
}
```

1 **valid_mmap_phys_addr_range** 檢查物理地址是不是在範圍內
```
/*
 * Do not allow /dev/mem mappings beyond the supported physical range.
 */
int valid_mmap_phys_addr_range(unsigned long pfn, size_t size)
{
	return (pfn + (size >> PAGE_SHIFT)) <= (1 + (PHYS_MASK >> PAGE_SHIFT));
}
```

2 **private_mapping_ok** 對於有 MMU 的 CPU 直接回傳 1(MMU的權限管理可以支持私有映射)
```
#ifndef CONFIG_MMU
...
#else

static inline int private_mapping_ok(struct vm_area_struct *vma)
{
	return 1;
}
#endif
```

3 **range_is_allowed** 以 frame 為單位檢查物理地址，每一頁都呼叫 **devmem_is_allowed** 檢查
```
#ifdef CONFIG_STRICT_DEVMEM
static inline int range_is_allowed(unsigned long pfn, unsigned long size)
{
	u64 from = ((u64)pfn) << PAGE_SHIFT;
	u64 to = from + size;
	u64 cursor = from;

	while (cursor < to) {
		if (!devmem_is_allowed(pfn)) {
			printk(KERN_INFO
		"Program %s tried to access /dev/mem between %Lx->%Lx.\n",
				current->comm, from, to);
			return 0;
		}
		cursor += PAGE_SIZE;
		pfn++;
	}
	return 1;
}
#else
static inline int range_is_allowed(unsigned long pfn, unsigned long size)
{
	return 1;
}
#endif
```
3.1 **devmem_is_allowed**
```
/*
 * devmem_is_allowed() checks to see if /dev/mem access to a certain
 * address is valid. The argument is a physical page number.
 * We mimic x86 here by disallowing access to system RAM as well as
 * device-exclusive MMIO regions. This effectively disable read()/write()
 * on /dev/mem.
 */
int devmem_is_allowed(unsigned long pfn)
{
	if (iomem_is_exclusive(pfn << PAGE_SHIFT))
		return 0;
	if (!page_is_ram(pfn))
		return 1;
	return 0;
}

```
3.1.1 **iomem_is_exclusive** 遍歷 iomem_resource，檢查物理地址是否為 busy 或 exclusive 
```
#ifdef CONFIG_STRICT_DEVMEM
static int strict_iomem_checks = 1;
#else
static int strict_iomem_checks;
#endif

/*
 * check if an address is reserved in the iomem resource tree
 * returns 1 if reserved, 0 if not reserved.
 */
int iomem_is_exclusive(u64 addr)
{
	struct resource *p = &iomem_resource;
	int err = 0;
	loff_t l;
	int size = PAGE_SIZE;

	if (!strict_iomem_checks)
		return 0;

	addr = addr & PAGE_MASK;

	read_lock(&resource_lock);
	for (p = p->child; p ; p = r_next(NULL, p, &l)) {
		/*
		 * We can probably skip the resources without
		 * IORESOURCE_IO attribute?
		 */
		if (p->start >= addr + size)
			break;
		if (p->end < addr)
			continue;
		if (p->flags & IORESOURCE_BUSY &&
		     p->flags & IORESOURCE_EXCLUSIVE) {
			err = 1;
			break;
		}
	}
	read_unlock(&resource_lock);

	return err;
}
```
> 對於外設的IO資源，kernel中使用platform device機制來註冊平台設備（platform_device_register）時調用insert_resource將該設備相應的io資源插入到iomem_resource鍊錶中。
> 如果我要對某外設的IO資源進行保護，防止用戶空間訪問，可以將其resource的flags置位exclusive即可。
> <a href="https://blog.csdn.net/skyflying2012/article/details/47611399" target="_blank" rel="noopener">https://blog.csdn.net/skyflying2012/article/details/47611399</a>

3.1.2 不允許訪問 ram 地址
```
int page_is_ram(unsigned long pfn)
{
#ifndef CONFIG_PPC64	/* XXX for now */
	return pfn < max_pfn;
#else
	unsigned long paddr = (pfn << PAGE_SHIFT);
	struct memblock_region *reg;

	for_each_memblock(memory, reg)
		if (paddr >= reg->base && paddr < (reg->base + reg->size))
			return 1;
	return 0;
#endif
}
```

4 `phys_mem_access_prot_allowed` 直接回傳1

5 `phys_mem_access_prot` 確定我們映射頁的權限

6 最後呼叫 `remap_pfn_range` 設定 paging

結論： 如果有開 CONFIG_STRICT_DEVMEM 會先檢查

1. 物理地址不能超過上限
2. 不能是exclusive
3. 不能是ram

關閉的話就沒有限制