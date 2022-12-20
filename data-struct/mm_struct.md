### mm_struct 

1、指向进程所拥有的内存描述符，保存了进程的内存管理信息

```
struct mm_struct 
{
    struct vm_area_struct * mmap;        /* list of VMAs */
    struct rb_root mm_rb;
    struct vm_area_struct * mmap_cache;    /* last find_vma result */
    unsigned long (*get_unmapped_area) (struct file *filp, unsigned long addr, unsigned long len, unsigned long pgoff, unsigned long flags);
    void (*unmap_area) (struct mm_struct *mm, unsigned long addr);
    unsigned long mmap_base;        /* base of mmap area */
    unsigned long task_size;        /* size of task vm space */
    unsigned long cached_hole_size;     /* if non-zero, the largest hole below free_area_cache */
    unsigned long free_area_cache;        /* first hole of size cached_hole_size or larger */
    pgd_t * pgd;
    atomic_t mm_users;            /* How many users with user space? */
    atomic_t mm_count;            /* How many references to "struct mm_struct" (users count as 1) */
    int map_count;                /* number of VMAs */
    struct rw_semaphore mmap_sem;
    spinlock_t page_table_lock;        /* Protects page tables and some counters */

    /* List of maybe swapped mm's.    These are globally strung together off init_mm.mmlist, and are protected by mmlist_lock */
    struct list_head mmlist;        
    /* Special counters, in some configurations protected by the
     * page_table_lock, in other configurations by being atomic.
     */
    mm_counter_t _file_rss;
    mm_counter_t _anon_rss;

    unsigned long hiwater_rss;    /* High-watermark of RSS usage */
    unsigned long hiwater_vm;    /* High-water virtual memory usage */

    unsigned long total_vm, locked_vm, shared_vm, exec_vm;
    unsigned long stack_vm, reserved_vm, def_flags, nr_ptes;
    unsigned long start_code, end_code, start_data, end_data;
    unsigned long start_brk, brk, start_stack;
    unsigned long arg_start, arg_end, env_start, env_end;

    unsigned long saved_auxv[AT_VECTOR_SIZE]; /* for /proc/PID/auxv */

    struct linux_binfmt *binfmt;

    cpumask_t cpu_vm_mask;

    /* Architecture-specific MM context */
    mm_context_t context;

    /* Swap token stuff */
    /*
     * Last value of global fault stamp as seen by this process.
     * In other words, this value gives an indication of how long
     * it has been since this task got the token.
     * Look at mm/thrash.c
     */
    unsigned int faultstamp;
    unsigned int token_priority;
    unsigned int last_interval;

    unsigned long flags; /* Must use atomic bitops to access the bits */

    struct core_state *core_state; /* coredumping support */
#ifdef CONFIG_AIO
    spinlock_t        ioctx_lock;
    struct hlist_head    ioctx_list;
#endif
#ifdef CONFIG_MM_OWNER
    /*
     * "owner" points to a task that is regarded as the canonical
     * user/owner of this mm. All of the following must be true in
     * order for it to be changed:
     *
     * current == mm->owner
     * current->mm != mm
     * new_owner->mm == mm
     * new_owner->alloc_lock is held
     */
    struct task_struct *owner;
#endif

#ifdef CONFIG_PROC_FS
    /* store ref to file /proc/<pid>/exe symlink points to */
    struct file *exe_file;
    unsigned long num_exe_file_vmas;
#endif
#ifdef CONFIG_MMU_NOTIFIER
    struct mmu_notifier_mm *mmu_notifier_mm;
#endif
};
```

### vm_area_struct

1、进程虚拟内存的每个区域表示为struct vm_area_struct的一个实例

```
struct vm_area_struct 
{
    /* 
    associated mm_struct 
    vm_mm是一个反向指针，指向该区域所属的mm_struct实例
    */
    struct mm_struct             *vm_mm;   
    
    /* VMA start, inclusive vm_mm内的起始地址 */
    unsigned long                vm_start; 
    /* VMA end , exclusive 在vm_mm内结束地址之后的第一个字节的地址 */
    unsigned long                vm_end;    
    
    /* 
    list of VMA's 
    进程所有vm_area_struct实例的链表是通过vm_next实现的
    各进程的虚拟内存区域链表，按地址排序 
    */
    struct vm_area_struct        *vm_next;     

    /* 
    access permissions 
    该虚拟内存区域的访问权限 
    1) _PAGE_READ
    2) _PAGE_WRITE
    3) _PAGE_EXECUTE
    */
    pgprot_t                     vm_page_prot; 
    
    /* 
    flags 
    vm_flags是描述该区域的一组标志，用于定义区域性质，这些都是在<mm.h>中声明的预处理器常数 
    */
    unsigned long                vm_flags;      
    struct rb_node               vm_rb;         /* VMA's node in the tree */

    /*
    对于有地址空间和后备存储器的区域来说:
    shared连接到address_space->i_mmap优先树
    或连接到悬挂在优先树结点之外、类似的一组虚拟内存区的链表
    或连接到ddress_space->i_mmap_nonlinear链表中的虚拟内存区域
    */
    union 
    {         /* links to address_space->i_mmap or i_mmap_nonlinear */
        struct 
        {
            struct list_head        list;
            void                    *parent;
            struct vm_area_struct   *head;
        } vm_set;
        struct prio_tree_node prio_tree_node;
    } shared;

    /*
    在文件的某一页经过写时复制之后，文件的MAP_PRIVATE虚拟内存区域可能同时在i_mmap树和anon_vma链表中，MAP_SHARED虚拟内存区域只能在i_mmap树中
    匿名的MAP_PRIVATE、栈或brk虚拟内存区域(file指针为NULL)只能处于anon_vma链表中
    */
    struct list_head             anon_vma_node;     /* anon_vma entry 对该成员的访问通过anon_vma->lock串行化 */
    struct anon_vma              *anon_vma;         /* anonymous VMA object 对该成员的访问通过page_table_lock串行化 */
    struct vm_operations_struct  *vm_ops;           /* associated ops 用于处理该结构的各个函数指针 */
    unsigned long                vm_pgoff;          /* offset within file 后备存储器的有关信息 */
    struct file                  *vm_file;          /* mapped file, if any 映射到的文件(可能是NULL) */
    void                         *vm_private_data;  /* private data vm_pte(即共享内存) */
};
```

### x

```

```

### x

```

```
