
```
struct pid
{
    //1. 指向该数据结构的引用次数
    atomic_t count;

    /*
    2. 该pid在pid_namespace中处于第几层
        1) 当level=0时
        表示是global namespace，即最高层 
    */
    unsigned int level;
    
    /* lists of tasks that use this pid */
    //3. tasks[i]指向的是一个哈希表。譬如说tasks[PIDTYPE_PID]指向的是PID的哈希表
    struct hlist_head tasks[PIDTYPE_MAX];

    //4. 
    struct rcu_head rcu;

    /*
    5. numbers[1]域指向的是upid结构体
    numbers数组的本意是想表示不同的pid_namespace，一个PID可以属于不同的namespace
        1) umbers[0]表示global namespace
        2) numbers[i]表示第i层namespace
        3) i越大所在层级越低
    目前该数组只有一个元素， 即global namespace。所以namepace的概念虽然引入了pid，但是并未真正使用，在未来的版本可能会用到
    */
    struct upid numbers[1];
};
```
