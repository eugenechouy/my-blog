### Linux PID

在linux中ID主要有以下四種:

1. PID
2. TGID 
如果一個行程是以 `CLONE_THREAD` 建立，則它處於一個 Thread Group，ID就是TGID，相同的Thread Group TGID都相同，其中Thread Group leader或是沒有使用執行續的PID和TGID相同。
3. PGID
    行程可以組成行程組，PGID等於組長ID
4. SID
    行程組也可以組成 session
    
為了管理PID linux使用了許多數據結構，直接從原始碼觀察比較難理解，以下我們慢慢將需求加入來分析

#### 忽略行程之間的關係及namespace
```
struct task_struct {
    ...
    struct pid_link pids;
    ...
};

struct pid_link {
    struct hlist_node node;
    struct pid *pid;
};

struct pid {
    struct hlist_head tasks;
    int nr;  // PID
    struct hlist_node pid_chain;
};
```
每個 task_struct 有個指向 struct pid 的指標，strcut pid 包含 PID

* pid_hash
* pid_map

#### 考慮行程間的關係
task_struct 中的 pid_link 多加幾項來只到其組長的 struct pid 
struct pid 加上幾項來指回組長的 task_struct

```
enum pid_type {
    PIDTYPE_PID,
    PIDTYPE_PGID,
    PIDTYPE_SID,
    PIDTYPE_MAX // number of ID type (not include TGID)
};

struct task_struct {
    ...
    pid_t pid;
    pid_t tgid;
    struct task_struct *group_leader; // Thread group leader
    struct pid_link pids[PIDTYPE_MAX];
    ...
};

struct pid_link {
    struct hlist_node node;
    struct pid *pid;
};

struct pid {
    struct hlist_head tasks[PIDTYPE_MAX];
    int nr;
    int hlist_node pid_chain;
}
```
假設今天有三個行程A B C在同一個行程組，其中A是行程組組長
* B, C 的 pids[PIDTYPE_PFID] 指向 A 的 struct pid
* A 的 struct pid 的 task[PIDTYPE_PGID] 串連起所有以該 PID 為組長的的行程　

#### 增加 PID Namespace

在每個可見行程的 namespace 都會給該行程分配一個 PID，所以一個行程可能有多個 PID
```
struct pid{
    unsigned int level;
    struct hlist_head tasks[PIDTYPE_MAX];
    struct upid numbers[1];
}

struct upid{
    int nr;
    struct pid_namespace *ns;   // pid 所處 namespace
    struct hlist_node pid_chain;
}
```
numbers[0] 表示 global namespace，numbers[i] 表示第 i 層 namespace，i 越大所在層級越低

#### 原碼

```
include/linux/sched.h

struct task_struct {
	···
	pid_t                 pid;
	pid_t                 tgid;
	struct task_struct   *group_leader;
	struct pid_link       pids[PIDTYPE_MAX];
	struct nsproxy       *nsproxy;
	···
};
```
其中 nsproxy 存該行程相關 namespace 資訊
```
include/linux/nsproxy.h

struct nsproxy {
    atomic_t count;
    struct uts_namespace *uts_ns;
    struct ipc_namespace *ipc_ns;
    struct mnt_namespace *mnt_ns;
    struct pid_namespace *pid_ns_for_children;
    struct net 	     *net_ns;
    struct cgroup_namespace *cgroup_ns;
};
```

```
include/linux/pid.h

struct upid {
        /* Try to keep pid_chain in the same cacheline as nr for find_vpid */
        int nr;
        struct pid_namespace *ns;
        struct hlist_node pid_chain;
};

struct pid
{
	atomic_t count;
	unsigned int level;
	/* lists of tasks that use this pid */
	struct hlist_head tasks[PIDTYPE_MAX];
	struct rcu_head rcu;
	struct upid numbers[1];
};
```

### 第一題

Write a new system call `get_process_zero_session_group(unsigned int *, int)`so that a process can use it to get the global PIDs of all processes that are in the same login session of process 0

這次的題目比上次單純許多，只要跑過每個 process 然後抓出 sid = 0 即可

```
asmlinkage int get_process_zero_session_group(unsigned int *result, int size){
        struct task_struct *p;
        unsigned int ret[size];
        int cnt = 0;

        for_each_process(p){
            if( task_session(p)->numbers[0].nr == 0 && cnt < size) 
                ret[cnt++] = task_pid(p)->numbers[0].nr;
        }

        if(copy_to_user(result, ret, sizeof(ret)))
            printk("error while copy_to_user\n");

        return cnt;
}
```

### 第二題

第二題是要拿出跟呼叫 system call 的 process 同一個 session 的 process，首先抓出 process 的 sid 再跑過所有 process 即可，注意上述的 sid 都要是在 **root namespace** 的 sid

```
asmlinkage int get_process_session_group(unsigned int *result, int size){
    struct task_struct *p;
    int current_sid;
    unsigned int ret[size];
    int cnt = 0;

    current_sid = task_session(current)->numbers[0].nr;
    for_each_process(p){
        if( task_session(p)->numbers[0].nr == current_sid && cnt < size )
            ret[cnt++] = task_pid(p)->numbers[task_pid(p)->level].nr;
    }
    
    if(copy_to_user(result, ret, sizeof(ret)))
            printk("error while copy_to_user\n");
    return cnt;
}   
```

> <p>reference:<br>
<a href="https://www.cnblogs.com/hazir/p/linux_kernel_pid.html" target="_blank" rel="noopener">https://www.cnblogs.com/hazir/p/linux_kernel_pid.html</a><br>
<a href="https://carecraft.github.io/basictheory/2017/03/linux-pid-manage/" target="_blank" rel="noopener">https://carecraft.github.io/basictheory/2017/03/linux-pid-manage/</a></p>