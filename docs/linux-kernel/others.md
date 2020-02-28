### Virtual Memory Areas(VMA)
`struct mm_struct` 中有一個稱為 `mmap` 的 field，`mmap` 的型態為 `struct vm_area_struct`
Linux 以 `struct vm_area_struct` 資料結構來紀錄每一「區塊」的 VMA 資訊
> VMA 是 user process 裡一段 virtual address space 區塊，virtual address space 是連續的記憶體空間，當然 VMA 也會是連續的空間。VMA 對 Linux 的主要好處是，可以記憶體的使用更有效率，並且更容易管理 user process address space。
> 
> 從另一個觀念來看，VMA 可以讓 Linux kernel 以 process 的角度來管理 virtual address space。Process 的 VMA 對映，可以由 /proc/<pid>/maps 檔案查詢
> 
> src: <a href="http://www.jollen.org/blog/2007/01/linux_virtual_memory_areas_vma.html" target="_blank" rel="noopener">http://www.jollen.org/blog/2007/01/linux_virtual_memory_areas_vma.html</a>

```
struct vm_area_struct {
        struct mm_struct * vm_mm;
        unsigned long vm_start;
        unsigned long vm_end;

        struct vm_area_struct *vm_next;

        pgprot_t vm_page_prot;
        unsigned long vm_flags;

        rb_node_t vm_rb;

        struct vm_area_struct *vm_next_share;
        struct vm_area_struct **vm_pprev_share;

        struct vm_operations_struct * vm_ops;

        unsigned long vm_pgoff;

        struct file * vm_file;
        unsigned long vm_raend;
        void * vm_private_data;
};
```

### Linux Paging
Linux中採用了一種通用的四級分頁機制

* Page Global Directory 資料結構為 **pgd_t**
* Page Upper Directory **pmd_t**
* Page Middle Directory **pud_t**
* Page Table **pte_t**

在這種分頁機制下，一個完整的線性地址被分為五部分：**頁全局目錄+頁上級目錄+頁中間目錄+頁表+偏移量**

> 不管系統採用多少級分頁模型，線性地址本質上都是索引+偏移量的形式，甚至你可以將整個線性地址看作N+1個索引的組合，N是系統採用的分頁級數。在四級分頁模型下，線性地址被分為5部分
> ![](https://i.imgur.com/9BHfAGK.jpg)
> 
> src: <a href="http://edsionte.com/techblog/archives/3435" target="_blank" rel="noopener">http://edsionte.com/techblog/archives/3435</a>

**虛擬地址轉物理 :** 

**Step 1** 

從 CR3 register 中讀取 pgd 所在物理地址的基址，從 linear address 的第一部分獲取 pgd 的索引，兩者相加即是 virtual address 在 pgd 中對應欄的線性地址

```
arch/x86/include/asm/pgtable_64_types.h

/*
 * PGDIR_SHIFT determines what a top-level page table entry can map
 */
#define PGDIR_SHIFT     39
#define PTRS_PER_PGD    512
```

```
arch/x86/include/asm/pgtable.h

#define pgd_index(address) (((address) >> PGDIR_SHIFT) & (PTRS_PER_PGD - 1))
#define pgd_offset(mm, address) ((mm)->pgd + pgd_index((address)))
```

> 其中PGDIR_SHIFT 39 是 pmd(9)+pud(9)+pte(9)+offset(12)
> 
> PGD，PUD，PMD，PTE分別都是一個4k的page，PGD，PUD，PMD，PTE是四張table，table的大小都是4k，其中table的entry分別是: pgd_t, pud_t, pmd_t, pte_t,都是unsigned long類型（8個字節），4k（2的12次方）/8字節（2的3次方）= 512個entry（2的9次方）

**Step 2** 

`pgd_val(pgd)` 取得 pgd 中那欄所指到的值，它與 `PTE_PFN_MASK` 計算後再丟到 `__va()` 即得到 pud 所在物理地址的基址，從 linear address 的第一部分獲取 pud 的索引，兩者相加即是 virtual address 在 pud 中對應欄的線性地址

```
arch/x86/include/asm/pgtable_64_types.h

/*
 * 3rd level page
 */
#define PUD_SHIFT       30
#define PTRS_PER_PUD    512
```

```
arch/x86/include/asm/pgtable_types.h

/* Extracts the PFN from a (pte|pmd|pud|pgd)val_t of a 4KB page */
#define PTE_PFN_MASK            ((pteval_t)PHYSICAL_PAGE_MASK)
```

```
arch/x86/include/asm/pgtable.h

static inline unsigned long pgd_page_vaddr(pgd_t pgd)
{
        return (unsigned long)__va((unsigned long)pgd_val(pgd) & PTE_PFN_MASK);
}


/* to find an entry in a page-table-directory. */
static inline unsigned long pud_index(unsigned long address)
{
        return (address >> PUD_SHIFT) & (PTRS_PER_PUD - 1);
}

static inline pud_t *pud_offset(pgd_t *pgd, unsigned long address)
{
        return (pud_t *)pgd_page_vaddr(*pgd) + pud_index(address);
}  
```

```
arch/x86/include/asm/page.h

#define __va(x)                 ((void *)((unsigned long)(x)+PAGE_OFFSET))
```

後面步驟都差不多，最終拿到 pte 中對應欄的線性地址再用 
```
page_addr = pte_val(*pte) & PTE_PFN_MASK & PAGE_MASK;
page_offset = vaddr & ~PAGE_MASK;
paddr = page_addr | page_offset;
```

### copy-on-write

當我們 `fork()` 時，會產生一個和父進程完全相同的子進程(除了pid)
如果按傳統的做法，會直接將父進程的數據拷貝到子進程中，拷貝完之後，父進程和子進程之間的**數據段**和**堆棧**是相互獨立的
但是，通常子進程都會執行 `exec()` 來做自己想要實現的功能。

> exec()
> exec函數的作用就是：裝載一個新的程序（可執行映像）覆蓋當前進程內存空間中的映像，從而執行不同的任務。
> exec系列函數在執行時會直接替換掉當前進程的地址空間。

所以，如果按照傳統做法的話，創建子進程時復製過去的數據是沒用的(因為子進程執行exec()，原有的數據會被清空)，既然很多時候複製給子進程的數據是無效的，於是就有了Copy On Write這項技術了

* fork創建出的子進程，與父進程共享內存空間。也就是說，如果子進程不對內存空間進行寫入操作的話，內存空間中的數據並不會復制給子進程
* 並且如果在fork函數返回之後，子進程第一時間 exec一個新的可執行映像，那麼也不會浪費時間和內存空間

結論:

1. 在fork之後exec之前兩個進程用的是相同的物理空間（內存區），子進程的代碼段、數據段、堆棧都是指向父進程的物理空間，也就是說，**兩者的虛擬空間不同，但其對應的物理空間是同一個**
2. 當父子進程中**有更改相應段的行為發生時，再為子進程相應的段分配物理空間**
3. 如果不是因為exec，內核會給**子進程的數據段、堆棧段分配相應的物理空間**（至此兩者有各自的進程空間，互不影響），而代碼段繼續共享父進程的物理空間（兩者的代碼完全相同）。
4. 而如果是因為exec，由於兩者執行的代碼不同，子進程的代碼段也會分配單獨的物理空間。

實測後:(唯一差別是virtual address都一樣，沒有像上面說的都會不同)

```
if not write:
    phyiscal address same
else
    if no exec:
        data, stack, heap segment different
        code segment same
    if exec:
        all different
```

> <a href="http://eastrivervillage.com/The-Linux-COW/" target="_blank" rel="noopener">http://eastrivervillage.com/The-Linux-COW/</a>

### asmlinkage | fastcall

linux 支持多種CPU架構，比如x86、ppc和arm等，在不同的處理器結構上參數的傳遞方式都不同，例如

* x86的函數參數和函數內部局部變量會被分配到函數的stack中
* arm則是對函數調用過程中的傳參定義了一套規則，即ATPCS，規則中明確指出ARM中R0-R4都是作為通用寄存器使用，在函數調用時處理器從R0-R4中獲取參數，在函數返回時再將需要返回的參數一次存到R0-R4中，也就是說可以將函數參數直接存放在寄存器中

所以為了嚴格區別函數參數的存放位置，引入了兩個標記

* asmlinkage 表示將函數參數存放在stack中
* FASTCALL 是通知編譯器將函數參數用寄存器保存起來

```
#define asmlinkage CPP_ASMLINKAGE __attribute__((regparm(0)))
#define FASTCALL(x)	x __attribute__((regparm(3)))
#define fastcall	__attribute__((regparm(3)))
```

> __attribute__((regparm(0)))：告訴gcc編譯器該函數不需要通過任何寄存器來傳遞參數，參數只是通過堆棧來傳遞。
>__attribute__((regparm(3)))：告訴gcc編譯器這個函數可以通過寄存器傳遞多達3個的參數，這3個寄存器依次為EAX、EDX和ECX。更多的參數才通過堆棧傳遞。這樣可以減少一些入棧出棧操作，因此調用比較快。

### modules

* buildin module
```
obj-y += filename.o
```
編譯的時候就會在對應目錄產生built-in.o的檔案
這些檔案便會被包入在image裡面

* externel module
```
obj-m += filename.o
```
編譯的時候就會在對應目錄產生 `kernel/time/test_udelay.ko` 的檔案
這些檔案便會被放置在 `/lib/modules/$(uname -r)`

> <a href="http://yi-jyun.blogspot.com/2017/07/linux-kernel-modules.html" target="_blank" rel="noopener">http://yi-jyun.blogspot.com/2017/07/linux-kernel-modules.html</a>