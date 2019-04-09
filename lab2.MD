# ab2 内存管理
                         0  
|  Real Mode IDT      |  4kb Bios interrupt vector table  
|                     |    
|  boot SECTOR        | 0x7c00 where Boot loader loaded  
|                     |     
|  ELF header (4KB)   | 0x10000  load kernel code to 0x10000 then call i386——init  
|                     |
|                     | 640kb  
|  Bios video ram     |  
|  Kernel code   　   | 1MB  
|  Page Directory     |   


！[image](https://upload-images.jianshu.io/upload_images/286774-73004835aee1e4bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

>> 在启用分页前，页目录所在的物理页面的首地址需要存放到 CR3 寄存器中

>> cpu 在进行页式转换时会自动从CR3中取得页目录物理地址   =======CR3： page table dirctory address

>> linear address 的 最高10位 +CR3 ===============page table address

>> page table address + linear address 的中间10位 ========= Page frame 的adress

>> page frame address + linear address 的底 12位 ===== true physic address

**在JOS中，为了简单起见，只用了一个页目录表，整个系统的线性地址空间4GB是被内核和所有其他的用户程序所共用的。由于CPU 和内存速度的不匹配，这样地址翻译时势必会降低系统的效率。为了提高地址翻译的速度，x86处理器引入了地址翻译缓存TLB（旁路转换缓冲）来缓存最近翻译过的地址。当然缓存之后会引入缓存和内存中页表内容不一致的问题，可以通过重载CR3使整个TLB内容失效或者通过 invlpg 指令。

### marco

```

// 线性地址分为如下三部分
//
// +--------10------+-------10-------+---------12----------+
// | Page Directory |   Page Table   | Offset within Page  |
// |      Index     |      Index     |                     |
// +----------------+----------------+---------------------+
//  \--- PDX(la) --/ \--- PTX(la) --/ \---- PGOFF(la) ----/
//  \---------- PGNUM(la) ----------/
//

// 页目录和页表的一些常量定义
#define NPDENTRIES  1024   //每个页目录的页目录项数目为1024
#define NPTENTRIES  1024   //每个页表的页表项数目也为1024

#define PGSIZE      4096   // 页大小为4096B，即4KB
#define PGSHIFT     12      // log2(PGSIZE)

#define PTSIZE      (PGSIZE*NPTENTRIES) // 一个页目录项映射内存大小，4MB
#define PTSHIFT     22      // log2(PTSIZE)

#define PTXSHIFT    12       
#define PDXSHIFT    22  

// 页号
#define PGNUM(la)   (((uintptr_t) (la)) >> PTXSHIFT)

// 页目录项索引(高10位)
#define PDX(la)     ((((uintptr_t) (la)) >> PDXSHIFT) & 0x3FF)

// 页表项索引（中间10位）
#define PTX(la)     ((((uintptr_t) (la)) >> PTXSHIFT) & 0x3FF)

// 页内偏移
#define PGOFF(la)   (((uintptr_t) (la)) & 0xFFF)

// 由索引构造线性地址
#define PGADDR(d, t, o) ((void*) ((d) << PDXSHIFT | (t) << PTXSHIFT | (o)))

// 由PageInfo结构得到页面物理地址
static inline physaddr_t
page2pa(struct PageInfo *pp)
{
    return (pp - pages) << PGSHIFT;
}

// 由物理地址得到PageInfo结构体
static inline struct PageInfo*
pa2page(physaddr_t pa) 
{
    if (PGNUM(pa) >= npages)
        panic("pa2page called with invalid pa");
    return &pages[PGNUM(pa)];
}

// 与 page2pa 类似，只不过返回的是 PageInfo 结构 pp 所对应的物理页面的内核首地址(虚拟地址)
static inline void*
page2kva(struct PageInfo *pp)
{
    return KADDR(page2pa(pp));
}
```
### 我们要实现的函数
```
/ 初始化一个页面结构和page_free_list。
void    page_init(void);

// 分配物理页
struct PageInfo *page_alloc(int alloc_flags);

// 释放页面，将页面加入page_free_list
void    page_free(struct PageInfo *pp);

// 将物理页pp映射到虚拟地址va，权限设置为 perm | PTE_P
int page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm);

// 移除虚拟地址va的映射
void    page_remove(pde_t *pgdir, void *va);

// 返回虚拟地址va映射的物理页的PageInfo地址
struct PageInfo *page_lookup(pde_t *pgdir, void *va, pte_t **pte_store);

// 给定页目录地址pgdir，检查虚拟地址va是否可以用页表翻译，若能，返回页表项地址，
// 否则根据需要创建页表项并返回页表项的内核地址，注意不是物理地址。
pte_t *pgdir_walk(pde_t *pgdir, const void *va, int create);
由于 JOS 只用了一个页目录，不像现代操作系统那样每个都有自己的页目录，所以整个系统的线性地址只有 4GB。JOS中内存组织如下图所示：
```
![image](https://upload-images.jianshu.io/upload_images/286774-70ab857ebcd75702.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)
