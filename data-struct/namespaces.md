
### nsproxy

```
/*
A structure to contain pointers to all per-process namespaces
1. fs (mount)
2. uts
3. network
4. sysvipc
5. etc
'count' is the number of tasks holding a reference. The count for each namespace, then, will be the number of nsproxies pointing to it, not the number of tasks.
The nsproxy is shared by tasks which share all namespaces. As soon as a single namespace is cloned or unshared, the nsproxy is copied.
*/
struct nsproxy 
{
    atomic_t count;

    /*
    1. UTS(UNIX Timesharing System)命名空间包含了运行内核的名称、版本、底层体系结构类型等信息
    */
    struct uts_namespace *uts_ns;

    /*
    2. 保存在struct ipc_namespace中的所有与进程间通信(IPC)有关的信息
    */
    struct ipc_namespace *ipc_ns;

    /*
    3. 已经装载的文件系统的视图，在struct mnt_namespace中给出
    */
    struct mnt_namespace *mnt_ns;

    /*
    4. 有关进程ID的信息，由struct pid_namespace提供
    */
    struct pid_namespace *pid_ns;

    /*
    5. struct net包含所有网络相关的命名空间参数
    */
    struct net          *net_ns;
};
extern struct nsproxy init_nsproxy;
```

### ipc_namespace

1、管理IPC命名空间比较简单，因为它们之间没有层次关系，给定的进程属于task_struct->nsproxy->ipc_ns指向的命名空间，初始的默认命名空间通过ipc_namespace的静态实例init_ipc_ns实现

2、每个struct ipc_ids结构实例对应于一种IPC机制: 共享内存、信号量、消息队列。为了防止对每个类别都需要查找对应的正确数组索引，内核提供了辅助函数msg_ids、shm_ids、sem_ids

```
struct ipc_namespace 
{
    atomic_t    count;
    /*
    每个数组元素对应一种IPC机制
        1) ids[0]: 信号量
        2) ids[1]: 消息队列
        3) ids[2]: 共享内存
    */
    struct ipc_ids    ids[3];

    int        sem_ctls[4];
    int        used_sems;

    int        msg_ctlmax;
    int        msg_ctlmnb;
    int        msg_ctlmni;
    atomic_t    msg_bytes;
    atomic_t    msg_hdrs;
    int        auto_msgmni;

    size_t        shm_ctlmax;
    size_t        shm_ctlall;
    int        shm_ctlmni;
    int        shm_tot;

    struct notifier_block ipcns_nb;

    /* The kern_mount of the mqueuefs sb.  We take a ref on it */
    struct vfsmount    *mq_mnt;

    /* # queues in this ns, protected by mq_lock */
    unsigned int    mq_queues_count;

    /* next fields are set through sysctl */
    unsigned int    mq_queues_max;   /* initialized to DFLT_QUEUESMAX */
    unsigned int    mq_msg_max;      /* initialized to DFLT_MSGMAX */
    unsigned int    mq_msgsize_max;  /* initialized to DFLT_MSGSIZEMAX */
};

struct ipc_ids 
{
    //1. 当前使用中IPC对象的数目
    int in_use; 

    /*
    2. 用户连续产生用户空间IPC ID，需要注意的是，ID不等同于序号，内核通过ID来标识IPC对象，ID按资源类型管理，即一个ID用于消息队列，一个用于信号量、一个用于共享内存对象
    每次创建新的IPC对象时，序号加1(自动进行回绕，即到达最大值自动变为0)
    用户层可见的ID = s * SEQ_MULTIPLER + i，其中s是当前序号，i是内核内部的ID，SEQ_MULTIPLER设置为IPC对象的上限
    如果重用了内部ID，仍然会产生不同的用户空间ID，因为序号不会重用，在用户层传递了一个旧的ID时，这种做法最小化了使用错误资源的风险
    */
    unsigned short seq;
    unsigned short seq_max; 

    //3. 内核信号量，它用于实现信号量操作，避免用户空间中的竞态条件，该互斥量有效地保护了包含信号量值的数据结构
    struct rw_semaphore rw_mutex;

    //4. 每个IPC对象都由kern_ipc_perm的一个实例表示，ipcs_idr用于将ID关联到指向对应的kern_ipc_perm实例的指针
    struct idr ipcs_idr;
};
```

### pid_namespace

```
struct pid_namespace 
{
    struct kref kref;
    struct pidmap pidmap[PIDMAP_ENTRIES];
    int last_pid;

    /*
    1. 每个PID命名空间都具有一个进程，其发挥的作用相当于全局的init进程，init的一个目的是对孤儿调用wait4，命名空间局部的init变体也必须完成该工作。child_reaper保存了指向该进程的task_struct的指针
    */
    struct task_struct *child_reaper;
    struct kmem_cache *pid_cachep;

    /*
    2. level表示当前命名空间在命名空间层次结构中的深度。初始命名空间的level为0，下一层为1，逐层递增。level较高的命名空间中的ID，对level较低的命名空间来说是可见的(即子命名空间对父命名空间可见)。从给定的level位置，内核即可推断进程会关联到多少个ID(即子命名空间中的进程需要关联从当前命名空间一直到最顶层的所有命名空间)
    */
    unsigned int level;

    /*
    3. parent是指向父命名空间的指针
    */
    struct pid_namespace *parent;
#ifdef CONFIG_PROC_FS
    struct vfsmount *proc_mnt;
#endif
#ifdef CONFIG_BSD_PROCESS_ACCT
    struct bsd_acct_struct *bacct;
#endif
};
```

### mnt_namespace

```
struct mnt_namespace 
{
    //使用计数器，指定了使用该命名空间的进程数目
    atomic_t        count;

    //指向根目录的vfsmount实例
    struct vfsmount *    root;

    //双链表表头，保存了VFS命名空间中所有文件系统的vfsmount实例，链表元素是vfsmount的成员mnt_list
    struct list_head    list;
    wait_queue_head_t poll;
    int event;
};
```

### net

```
struct net {
	atomic_t		passive;	/* To decided when the network
						 * namespace should be freed.
						 */
	atomic_t		count;		/* To decided when the network
						 *  namespace should be shut down.
						 */
#ifdef NETNS_REFCNT_DEBUG
	atomic_t		use_count;	/* To track references we
						 * destroy on demand
						 */
#endif
	spinlock_t		rules_mod_lock;

	struct list_head	list;		/* list of network namespaces */
	struct list_head	cleanup_list;	/* namespaces on death row */
	struct list_head	exit_list;	/* Use only net_mutex */

	struct user_namespace   *user_ns;	/* Owning user namespace */

	unsigned int		proc_inum;

	struct proc_dir_entry 	*proc_net;
	struct proc_dir_entry 	*proc_net_stat;

#ifdef CONFIG_SYSCTL
	struct ctl_table_set	sysctls;
#endif

	struct sock 		*rtnl;			/* rtnetlink socket */
	struct sock		*genl_sock;

	struct list_head 	dev_base_head;
	struct hlist_head 	*dev_name_head;
	struct hlist_head	*dev_index_head;
	unsigned int		dev_base_seq;	/* protected by rtnl_mutex */
	int			ifindex;

	/* core fib_rules */
	struct list_head	rules_ops;


	struct net_device       *loopback_dev;          /* The loopback */
	struct netns_core	core;
	struct netns_mib	mib;
	struct netns_packet	packet;
	struct netns_unix	unx;
	struct netns_ipv4	ipv4;
#if IS_ENABLED(CONFIG_IPV6)
	struct netns_ipv6	ipv6;
#endif
#if defined(CONFIG_IP_SCTP) || defined(CONFIG_IP_SCTP_MODULE)
	struct netns_sctp	sctp;
#endif
#if defined(CONFIG_IP_DCCP) || defined(CONFIG_IP_DCCP_MODULE)
	struct netns_dccp	dccp;
#endif
#ifdef CONFIG_NETFILTER
	struct netns_nf		nf;
	struct netns_xt		xt;
#if defined(CONFIG_NF_CONNTRACK) || defined(CONFIG_NF_CONNTRACK_MODULE)
	struct netns_ct		ct;
#endif
#if IS_ENABLED(CONFIG_NF_DEFRAG_IPV6)
	struct netns_nf_frag	nf_frag;
#endif
	struct sock		*nfnl;
	struct sock		*nfnl_stash;
#endif
#ifdef CONFIG_WEXT_CORE
	struct sk_buff_head	wext_nlevents;
#endif
	struct net_generic __rcu	*gen;

	/* Note : following structs are cache line aligned */
#ifdef CONFIG_XFRM
	struct netns_xfrm	xfrm;
#endif
	struct netns_ipvs	*ipvs;
	struct sock		*diag_nlsk;
	atomic_t		rt_genid;
};
```

### uts_namespace

```
struct user_namespace {
	struct uid_gid_map	uid_map;
	struct uid_gid_map	gid_map;
	struct uid_gid_map	projid_map;
	atomic_t		count;
	struct user_namespace	*parent;
	int			level;
	kuid_t			owner;
	kgid_t			group;
	unsigned int		proc_inum;
	unsigned long		flags;
	bool			may_mount_sysfs;
	bool			may_mount_proc;
};

struct uts_namespace {
	struct kref kref;
	struct new_utsname name;
	struct user_namespace *user_ns;
	unsigned int proc_inum;
};
extern struct uts_namespace init_uts_ns;
```
