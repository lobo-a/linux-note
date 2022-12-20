### socket

```
struct socket 
{    
    /*
    1. state：socket状态
    typedef enum 
    {
        SS_FREE = 0,            //该socket还未分配
        SS_UNCONNECTED,         //未连向任何socket
        SS_CONNECTING,          //正在连接过程中
        SS_CONNECTED,           //已连向一个socket
        SS_DISCONNECTING        //正在断开连接的过程中
    }socket_state; 
    */
    socket_state        state;

    kmemcheck_bitfield_begin(type);
    /*
    2. type：socket类型
    enum sock_type 
    {
        SOCK_STREAM    = 1,    //stream (connection) socket
        SOCK_DGRAM    = 2,    //datagram (conn.less) socket
        SOCK_RAW    = 3,    //raw socket
        SOCK_RDM    = 4,    //reliably-delivered message
        SOCK_SEQPACKET    = 5,//sequential packet socket
        SOCK_DCCP    = 6,    //Datagram Congestion Control Protocol socket
        SOCK_PACKET    = 10,    //linux specific way of getting packets at the dev level.
    };
    */
    short            type;
    kmemcheck_bitfield_end(type);

    /*
    3. flags：socket标志
        1) #define SOCK_ASYNC_NOSPACE 0
        2) #define SOCK_ASYNC_WAITDATA 1
        3) #define SOCK_NOSPACE 2
        4) #define SOCK_PASSCRED 3
        5) #define SOCK_PASSSEC 4
    */
    unsigned long        flags;

    //fasync_list is used when processes have chosen asynchronous handling of this 'file'
    struct fasync_struct    *fasync_list;
    //4. Not used by sockets in AF_INET
    wait_queue_head_t    wait;

    //5. file holds a reference to the primary file structure associated with this socket
    struct file        *file;

    /*
    6. sock
    This is very important, as it contains most of the useful state associated with a socket. 
    */
    struct sock        *sk;

    //7. ops：定义了当前socket的处理函数
    const struct proto_ops    *ops;
};
```

### sock

1、struct sock保存和当前socket大量的元描述信息

2、struct sock本身不能获取到当前socket的IP、Port相关信息

3、通过inet_sk()进行转换得到struct inet_sock才能得到IP、Port相关信息

```
struct sock 
{
    /*
     * Now struct inet_timewait_sock also uses sock_common, so please just
     * don't add nothing before this first member (__sk_common) --acme
     */
    //shared layout with inet_timewait_sock
    struct sock_common    __sk_common;
#define sk_node            __sk_common.skc_node
#define sk_nulls_node        __sk_common.skc_nulls_node
#define sk_refcnt        __sk_common.skc_refcnt

#define sk_copy_start        __sk_common.skc_hash
#define sk_hash            __sk_common.skc_hash
#define sk_family        __sk_common.skc_family
#define sk_state        __sk_common.skc_state
#define sk_reuse        __sk_common.skc_reuse
#define sk_bound_dev_if        __sk_common.skc_bound_dev_if
#define sk_bind_node        __sk_common.skc_bind_node
#define sk_prot            __sk_common.skc_prot
#define sk_net            __sk_common.skc_net

    kmemcheck_bitfield_begin(flags);
    //mask of %SEND_SHUTDOWN and/or %RCV_SHUTDOWN
    unsigned int        sk_shutdown  : 2,
                //%SO_NO_CHECK setting, wether or not checkup packets
                sk_no_check  : 2,
                //%SO_SNDBUF and %SO_RCVBUF settings
                sk_userlocks : 4,
                //which protocol this socket belongs in this network family
                sk_protocol  : 8,
                //socket type (%SOCK_STREAM, etc)
                sk_type      : 16;
    kmemcheck_bitfield_end(flags);
    //size of receive buffer in bytes
    int            sk_rcvbuf;
    //synchronizer
    socket_lock_t        sk_lock;
    /*
     * The backlog queue is special, it is always used with
     * the per-socket spinlock held and requires low latency
     * access. Therefore we special case it's implementation.
     */
    struct 
    {
        struct sk_buff *head;
        struct sk_buff *tail;
    } sk_backlog;
    //sock wait queue
    wait_queue_head_t    *sk_sleep;
    //destination cache
    struct dst_entry    *sk_dst_cache;
#ifdef CONFIG_XFRM
    //flow policy
    struct xfrm_policy    *sk_policy[2];
#endif
    //destination cache lock
    rwlock_t        sk_dst_lock;
    //receive queue bytes committed
    atomic_t        sk_rmem_alloc;
    //transmit queue bytes committed
    atomic_t        sk_wmem_alloc;
    //"o" is "option" or "other"
    atomic_t        sk_omem_alloc;
    //size of send buffer in bytes
    int            sk_sndbuf;
    //incoming packets
    struct sk_buff_head    sk_receive_queue;
    //Packet sending queue
    struct sk_buff_head    sk_write_queue;
#ifdef CONFIG_NET_DMA
    //DMA copied packets
    struct sk_buff_head    sk_async_wait_queue;
#endif
    //persistent queue size
    int            sk_wmem_queued;
    //space allocated forward
    int            sk_forward_alloc;
    //allocation mode
    gfp_t            sk_allocation;
    //route capabilities (e.g. %NETIF_F_TSO)
    int            sk_route_caps;
    //GSO type (e.g. %SKB_GSO_TCPV4)
    int            sk_gso_type;
    //Maximum GSO segment size to build
    unsigned int        sk_gso_max_size;
    //%SO_RCVLOWAT setting
    int            sk_rcvlowat;
    /*
    1. %SO_LINGER (l_onoff)
    2. %SO_BROADCAST
    3. %SO_KEEPALIVE
    4. %SO_OOBINLINE settings
    5. %SO_TIMESTAMPING settings
    */
    unsigned long         sk_flags;
    //%SO_LINGER l_linger setting
    unsigned long            sk_lingertime;
    //rarely used
    struct sk_buff_head    sk_error_queue;
    //sk_prot of original sock creator (see ipv6_setsockopt, IPV6_ADDRFORM for instance)
    struct proto        *sk_prot_creator;
    //used with the callbacks in the end of this struct
    rwlock_t        sk_callback_lock;
    //last error
    int            sk_err,
                //rrors that don't cause failure but are the cause of a persistent failure not just 'timed out'
                sk_err_soft;
                    //raw/udp drops counter
    atomic_t        sk_drops;
    //always used with the per-socket spinlock held
    //current listen backlog
    unsigned short        sk_ack_backlog;
    //listen backlog set in listen()
    unsigned short        sk_max_ack_backlog;
    //%SO_PRIORITY setting
    __u32            sk_priority;
    //%SO_PEERCRED setting
    struct ucred        sk_peercred;
    //%SO_RCVTIMEO setting
    long            sk_rcvtimeo;
    //%SO_SNDTIMEO setting
    long            sk_sndtimeo;
    //socket filtering instructions
    struct sk_filter          *sk_filter;
    //private area, net family specific, when not using slab
    void            *sk_protinfo;
    //sock cleanup timer
    struct timer_list    sk_timer;
    //time stamp of last packet received
    ktime_t            sk_stamp;
    //Identd and reporting IO signals
    struct socket        *sk_socket;
    //RPC layer private data
    void            *sk_user_data;
    //cached page for sendmsg
    struct page        *sk_sndmsg_page;
    //front of stuff to transmit
    struct sk_buff        *sk_send_head;
    //cached offset for sendmsg
    __u32            sk_sndmsg_off;
    //a write to stream socket waits to start
    int            sk_write_pending;
#ifdef CONFIG_SECURITY
    //used by security modules
    void            *sk_security;
#endif
    //generic packet mark
    __u32            sk_mark;
    /* XXX 4 bytes hole on 64 bit */
    //callback to indicate change in the state of the sock
    void            (*sk_state_change)(struct sock *sk);
    //callback to indicate there is data to be processed
    void            (*sk_data_ready)(struct sock *sk, int bytes);
    //callback to indicate there is bf sending space available
    void            (*sk_write_space)(struct sock *sk);
    //callback to indicate errors (e.g. %MSG_ERRQUEUE)
    void            (*sk_error_report)(struct sock *sk);
    //callback to process the backlog
      int            (*sk_backlog_rcv)(struct sock *sk, struct sk_buff *skb);  
      //called at sock freeing time, i.e. when all refcnt == 0
    void                    (*sk_destruct)(struct sock *sk);
}
```

### proto_ops

```
struct proto_ops 
{
    int        family;
    struct module    *owner;
    int        (*release)   (struct socket *sock);
    int        (*bind)         (struct socket *sock, struct sockaddr *myaddr, int sockaddr_len);
    int        (*connect)   (struct socket *sock, struct sockaddr *vaddr, int sockaddr_len, int flags);
    int        (*socketpair)(struct socket *sock1, struct socket *sock2);
    int        (*accept)    (struct socket *sock, struct socket *newsock, int flags);
    int        (*getname)   (struct socket *sock, struct sockaddr *addr, int *sockaddr_len, int peer);
    unsigned int    (*poll)         (struct file *file, struct socket *sock, struct poll_table_struct *wait);
    int        (*ioctl)     (struct socket *sock, unsigned int cmd, unsigned long arg);
    int         (*compat_ioctl) (struct socket *sock, unsigned int cmd, unsigned long arg);
    int        (*listen)    (struct socket *sock, int len);
    int        (*shutdown)  (struct socket *sock, int flags);
    int        (*setsockopt)(struct socket *sock, int level, int optname, char __user *optval, unsigned int optlen);
    int        (*getsockopt)(struct socket *sock, int level, int optname, char __user *optval, int __user *optlen);
    int        (*compat_setsockopt)(struct socket *sock, int level, int optname, char __user *optval, unsigned int optlen);
    int        (*compat_getsockopt)(struct socket *sock, int level, int optname, char __user *optval, int __user *optlen);
    int        (*sendmsg)   (struct kiocb *iocb, struct socket *sock, struct msghdr *m, size_t total_len);
    /* Notes for implementing recvmsg:
     * ===============================
     * msg->msg_namelen should get updated by the recvmsg handlers
     * iff msg_name != NULL. It is by default 0 to prevent
     * returning uninitialized memory to user space.  The recvfrom
     * handlers can assume that msg.msg_name is either NULL or has
     * a minimum size of sizeof(struct sockaddr_storage).
     */
    int        (*recvmsg)   (struct kiocb *iocb, struct socket *sock, struct msghdr *m, size_t total_len, int flags);
    int        (*mmap)         (struct file *file, struct socket *sock, struct vm_area_struct * vma);
    ssize_t        (*sendpage)  (struct socket *sock, struct page *page, int offset, size_t size, int flags);
    ssize_t     (*splice_read)(struct socket *sock,  loff_t *ppos, struct pipe_inode_info *pipe, size_t len, unsigned int flags);
};
```

### sockaddr

```
struct sockaddr 
{
    // address family, AF_xxx
    unsigned short    sa_family;
    
    // 14 bytes of protocol address
    char              sa_data[14];  
};

/* Structure describing an Internet (IP) socket address. */
#define __SOCK_SIZE__    16        /* sizeof(struct sockaddr)    */
struct sockaddr_in 
{
    /* Address family */
    sa_family_t        sin_family;
    
    /* Port number */
    __be16        sin_port;
    
    /* Internet address */
    struct in_addr    sin_addr;    

    /* Pad to size of `struct sockaddr'. */
    unsigned char        __pad[__SOCK_SIZE__ - sizeof(short int) - izeof(unsigned short int) - sizeof(struct in_addr)];
};
#define sin_zero    __pad        /* for BSD UNIX comp. -FvK    */

/* Internet address. */
struct in_addr 
{
    __be32    s_addr;
};
```

### inet_sock

```
static inline struct inet_sock *inet_sk(const struct sock *sk)
{
    return (struct inet_sock *)sk;
}

struct inet_sock 
{
    /* sk and pinet6 has to be the first two members of inet_sock */
    //ancestor class
    struct sock        sk;
#if defined(CONFIG_IPV6) || defined(CONFIG_IPV6_MODULE)
    //pointer to IPv6 control block
    struct ipv6_pinfo    *pinet6;
#endif
    /* Socket demultiplex comparisons on incoming packets. */
    //Foreign IPv4 addr
    __be32            daddr;
    //Bound local IPv4 addr
    __be32            rcv_saddr;
    //Destination port
    __be16            dport;
    //Local port
    __u16            num;
    //Sending source
    __be32            saddr;
    //Unicast TTL
    __s16            uc_ttl;
    __u16            cmsg_flags;
    struct ip_options_rcu    *inet_opt;
    //Source port
    __be16            sport;
    //ID counter for DF pkts
    __u16            id;
    //TOS
    __u8            tos;
    //Multicasting TTL
    __u8            mc_ttl;
    __u8            pmtudisc;
    __u8            recverr:1,
                //is this an inet_connection_sock?
                is_icsk:1,
                freebind:1,
                hdrincl:1,
                mc_loop:1,
                transparent:1,
                mc_all:1;
                //Multicast device index
    int            mc_index;
    __be32            mc_addr;
    struct ip_mc_socklist    *mc_list;
    //info to build ip hdr on each ip frag while socket is corked
    struct 
    {
        unsigned int        flags;
        unsigned int        fragsize;
        struct ip_options    *opt;
        struct dst_entry    *dst;
        int            length; /* Total length of all frames */
        __be32            addr;
        struct flowi        fl;
    } cork;
};
```
