
Add a new system call `void linux_survey_TT(char *)` to your Linux kernel so that you can call it in your program

1. The system call has a parameter which specifies the address of a memory area that can store all information the system call collects in the kernel.
2. The system call records the virtual address intervals consisting of the user address space of the process executing the system call.
3. The system call records the corresponding physical address intervals used by the above virtual address intervals at the moment that you execute system call void linux_survey_TT().

---

### Step 1
我們要取得呼叫此 system call 的 process 的 virtual address intervals，目標就是去抓 `current` 裡面的 `mm_struct` 物件

#### task_struct
Linux kernel 中進程用 task_struct 結構體表示 
進程主要由以下幾部分組成：

* 代碼段：編譯後形成的一些指令
* 數據段：程序運行時需要的數據
    * 只讀數據段：常量
    * 已初始化數據段：全局變量，靜態變量
    * 未初始化數據段（bss)：未初始化的全局變量和靜態變量
* 堆棧段：程序運行時動態分配的一些內存
* PCB：進程信息，狀態標識等

![](https://i.imgur.com/c30RMFe.jpg)

```
struct task_struct {
   volatile long state; //进程状态
   struct mm_struct *mm, *active_mm; //内存地址空间
   pid_t pid;
     pid_t tgid;

   struct task_struct __rcu *real_parent; //真正的父进程，fork时记录的
   struct task_struct __rcu *parent; // ptrace后，设置为trace当前进程的进程
   struct list_head children;  //子进程
     struct list_head sibling;	//父进程的子进程，即兄弟进程
   struct task_struct *group_leader; //线程组的领头线程

   char comm[TASK_COMM_LEN];  //进程名，长度上限为16字符
   struct fs_struct *fs;  //文件系统信息
   struct files_struct *files; // 打开的文件

   struct signal_struct *signal;
   struct sighand_struct *sighand;
   struct sigpending pending;
   
   void *stack;    // 指向内核栈的指针
   ...
}    
```

而其中mm_struct長這樣

```
struct mm_struct {
    struct vm_area_struct * mmap;        /* list of VMAs */
    struct rb_root mm_rb;
    unsigned long mmap_base;        /* base of mmap area */
    unsigned long task_size;        /* size of task vm space */
    pgd_t * pgd;
    atomic_t mm_count;            /* How many references to "struct mm_struct" (users count as 1) */
    int map_count;                /* number of VMAs */

    unsigned long start_code, end_code, start_data, end_data;
    unsigned long start_brk, brk, start_stack;
    unsigned long arg_start, arg_end, env_start, env_end;

    struct file *exe_file;

    /* ... some code omitted ... */
};
```

其中有關virtual address intervals的資訊:

```
unsigned long start_code, end_code, start_data, end_data;
unsigned long start_brk, brk, start_stack;
unsigned long arg_start, arg_end, env_start, env_end;
```
這些變數存的是 process memory layout 中個區塊的起始, 結束位址，例如 text segments 就是`start_code` ~ `end_code`

 
```
struct vm_area_struct * mmap;
```
`mmap` 紀錄進程使用到的 VMA 們其中 `vm_area_struct` 中比較重要的資料有
```
unsigned long vm_start：記錄此 VMA 區塊的開始位址
unsigned long vm_end：記錄此 VMA 區塊的結束位址
struct vm_area_struct *vm_next：指向下一個 VMA 區塊結構的指標
```

我猜所有 VMA 的位址就是 process 的 virtual address intervals 了吧

```
void my_copy(char *result, unsigned long address, size_t length){
	if(copy_to_user(result, &address, length))
		printk("error while copy_to_user\n");
}

while(mmap->vm_next != NULL){
        my_copy(result, mmap->vm_start, length);
        result += length;
        my_copy(result, mmap->vm_end, length);
        result += length;
        mmap = mmap->vm_next;
}
my_copy(result, state_end, length);
result += length;
result += length;
```
這邊在做的事就是把每一個 vma 的 `vm_start` 和 `vm_end` 搬到 result，搬過去之後再將 result 往後 `sizeof(unsigned long)` 個 bytes 由於我是每個兩個 address 為一組，所以最後 `state_end` 我也保留兩個變數的位址

---

### Step 2
再來我們要取得 corresponding physical address intervals

理解 linux paging 就很好找了

```
#include <linux/kernel.h>
#include <linux/sched.h>
#include <asm/pgtable.h>

static unsigned long vaddr2paddr(unsigned long vaddr){
        pgd_t *pgd;
        pud_t *pud;
        pmd_t *pmd;
        pte_t *pte;
        unsigned long paddr=0, page_addr=0, page_offset=0;

        pgd = pgd_offset(current->mm, vaddr);
        if (pgd_none(*pgd)) {
            printk("not mapped in pgd\n");
            return -1;
        }
        pud = pud_offset(pgd, vaddr);
        if (pud_none(*pud)) {
            printk("not mapped in pud\n");
            return -1;
        }
        pmd = pmd_offset(pud, vaddr);
        if (pmd_none(*pmd)) {
            printk("not mapped in pmd\n");
            return -1;
        }
        pte = pte_offset_kernel(pmd, vaddr);
        if (pte_none(*pte)) {
            printk("not mapped in pte\n");
            return -1;
        }

        page_addr = pte_val(*pte) & PAGE_MASK;
        page_offset = vaddr & ~PAGE_MASK;
        paddr = page_addr | page_offset;
        return paddr;
}
```
透過 `pgd_offset`, `pud_offset`, `pmd_offset`, `pte_offset_kernel` 取得 page table，再配合`PAGE_MASK` 取得 physical address，其中並不是每個 virtual address 都有分配到 physical address，所以可能在取某級分頁時發生取不到的狀況，此時我們就可以判斷此 virtual address 沒有被分配到

```
mmap = mm->mmap;
        while(mmap->vm_next != NULL){	
                for(i=mmap->vm_start ; i<=mmap->vm_end ; i+=(~PAGE_MASK)+1){
                        unsigned long page_start = i, page_end = i+(~PAGE_MASK);
                        unsigned long frame_start = vaddr2paddr(page_start), frame_end = vaddr2paddr(page_end);
                        if(frame_start){
                                my_copy(result, page_start, length);
                                result += length;
                                my_copy(result, page_end, length);
                                result += length;
			        my_copy(result, frame_start, length);
                                result += length;
                                my_copy(result, frame_end, length);
                                result += length;
			}
                }
                my_copy(result, vma_end, length);
                result += length;
                result += length;
                result += length;
                result += length;
                mmap = mmap->vm_next;
        } 
	my_copy(result, state_end, length);
	result += length;
	result += length;
    result += length;
    result += length;

```
由於 virtual address 對到 physical address 是以page為單位，也就是說一塊 page 的起始位置有對應的physical address 則代表整塊 page 都有，所以我就遞迴過每個 vma 中的每個 page 這邊我是以每四個資料為一組(page頭尾+frame頭尾)，雖然這麼多 result+=length 有點醜不過我懶得用其他方法ㄌ 

---

### Step 3
列出呼叫 system call 時有多少 virtual address 有對應的 physical address（幾趴）

我這邊測試端按照剛剛的儲存方式抓出

 1 . virtual address intervals
    這時可以順便計算virtual address總數
```
int virtual_cnt = 0, physical_cnt = 0;
const int length = sizeof(unsigned long);
for(int i=0 ; ; i++) {
    unsigned long vm_start, vm_end;
    memcpy(&vm_start, result, length); next
    memcpy(&vm_end, result, length); next
    if(vm_start == STATE_END)
        break;
    virtual_cnt += (vm_end - vm_start);
    fprintf(pfile, "vma%d: 0x%lx ~ 0x%lx\n", i+1, vm_start, vm_end);
}
```

 2 . physical address intervals 順便算出physical address總數
    
```
fprintf(pfile, "physical address intervals: \n");
for(int i=0 ; ; ){
    unsigned long page_start, page_end, frame_start, frame_end;
    memcpy(&page_start, result, length); next
    memcpy(&page_end, result, length); next
    memcpy(&frame_start, result, length); next
    memcpy(&frame_end, result, length); next
    if(page_start == STATE_END)
        break;
    if(page_start == VMA_END){
        // fprintf(pfile, "\n");
        continue;
    }
    physical_cnt += MASK;
    i++;
    fprintf(pfile, "0x%lx ~ 0x%lx -> 0x%lx ~ 0x%lx\n", page_start, page_end, frame_start, frame_end);
}	
```
其中我這邊把 result+=length 替換成 next

 3 . 印出
```
printf("%s virtual addresses that have physical memory: %.2f% \n", filename, (double)physical_cnt/virtual_cnt*100);
```

---

### Step 4
印出哪些virtual address intervals對應到相同的physical address intervals

這裡就有點麻煩了，由於 fork() 後變成兩個process不能直接把資料存到變數中(child存的parent看不到)，所以必須要再讀檔案來分析

先宣告一個 struct 來存我們要的資料
```
struct page{
	unsigned long page_start;
	unsigned long frame_start;
};
```

讀檔後存在陣列中
```
void analyze_file(char *filename, struct page *pages){
	FILE *pfile;
	pfile = fopen(filename, "r");
	if(!pfile){
		printf("open file error!\n");
		return;
	}
    // 先抓掉前面的vma list
	char useless[100];
	while(fscanf(pfile, "%[^\n]", useless) != EOF && strcmp(useless, "physical address intervals: "))
		fscanf(pfile, "\n", useless);
	fscanf(pfile, "%[^\n]", useless);
	fscanf(pfile, "\n", useless);
    // 我們要的資訊
	unsigned long page_start, page_end, frame_start, frame_end;
	int i=0;
	while(fscanf(pfile, "0x%lx ~ 0x%lx -> 0x%lx ~ 0x%lx\n", &page_start, &page_end, &frame_start, &frame_end) != EOF){
		pages[i].page_start = page_start;
		pages[i].frame_start = frame_start;
		i++;
	}

	fclose(pfile);
}
```

最後再分別拿兩個child的result做比對（我就直接暴力了）
```
void calc_phy_relation(struct page *child, struct page *parent, int num){
	printf("\n\nthe virtual address intervals that map to same physical address at result_%d\n", num);
	for(int i=0 ; i<300 ; i++){
		if(!child[i].page_start && !child[i].frame_start) break;
		for(int j=0 ; j<300 ; j++){
			if(!parent[j].page_start && !parent[j].frame_start) break;
			if(child[i].frame_start == parent[j].frame_start){
				printf("0x%lx ~ 0x%lx\n", child[i].page_start, child[i].page_start+MASK-1);
				break;
			}
		}
	}
}
```

完成拉～～其實麻煩的都在資料處理

<p>reference:<br>
<a href="http://gityuan.com/2017/07/30/linux-process/" target="_blank" rel="noopener">http://gityuan.com/2017/07/30/linux-process/</a><br>
<a href="https://www.cnblogs.com/kwingmei/p/3731746.html" target="_blank" rel="noopener">https://www.cnblogs.com/kwingmei/p/3731746.html</a><br>
<a href="https://www.cnblogs.com/Rofael/archive/2013/04/13/3019153.html" target="_blank" rel="noopener">https://www.cnblogs.com/Rofael/archive/2013/04/13/3019153.html</a><br>
<a href="https://www.ffutop.com/posts/2019-07-17-understand-kernel-13/" target="_blank" rel="noopener">https://www.ffutop.com/posts/2019-07-17-understand-kernel-13/</a><br>
<a href="https://stackoverflow.com/questions/5748492/is-there-any-api-for-determining-the-physical-address-from-virtual-address-in-li" target="_blank" rel="noopener">https://stackoverflow.com/questions/5748492/is-there-any-api-for-determining-the-physical-address-from-virtual-address-in-li</a><br>
<a href="http://www.jollen.org/blog/2007/01/process_vma.html" target="_blank" rel="noopener">http://www.jollen.org/blog/2007/01/process_vma.html</a></p>