---
title: cgroup v1 (1)
date: 2017-10-18 11:16:50
tags: [cgroup, docker]
---

## 基本概念[0]


**cgroup**: 一组进程的集合，用于划分、组织系统中运行的进程，进而能够以cgroup为单位做资源的划分，控制。

**subsystem**: 用于控制cgroup下进程的组件。比如控制进程cpu使用量的cpu subsystem，用于统计cpu使用量的cpuacct subsystem，用于统计和限制进程内存使用的memory subsystem。因为目前实现的大多数的subsystem都是与资源相关的，所以有些地方也将这些subsystem称之为resource controller或controller，目前已经实现的subsystem有: cpu, cpuacct, cpuset, memory, hugetlb, net_cls, net_prio, blkio, devices, freezer, pids, perf_event，关于各个 subsystem功能的介绍，下文会有详细描述。


**hierarchy**: 实现subsystem与cgroup之间的绑定，同时也规定了cgroup之间的层次关系。一个 hierarchy可以绑定零个或者多个subsystem，hierarchy以树的形式组织cgroup，根节点的cgroup称为这个hierarchy的root cgroup，hierarchy下的child cgroup会受parent cgroup subsystem参数配置的限制。

![cgroup subsystem and hierarchy](https://drive.google.com/uc?id=0B8b6aqj4rk9BWUFQQnVpNDBPTjQ)

上图展示了hierarchy，cgroup，subsystem之间的关系，图中有两个hierarchy: cpu,cpuacct hierarchy(绑定cpu, cpuacct subsystem); memory hierarchy(绑定memory subsystem)。以 cpu,cpuacct hierarchy为例，根节点cpu,cpuacct cgroup下创建了两个子的cgroup: batch job、long running service，用于区分管理batch和long running service类型的进程。batch类型的进程放置到batch job cgroup下，long running service类型的进程放置到long running service cgroup下。通过给batch job和long running service cgroup配置不同的权重，可以实现cpu资源在两类工作负载之间的合理划分。上图也展示了关于cgroup使用的一些基本的规则[1]:

+ 一个hierarchy可以绑定零个或多个subsystem。

+ 可以创建多个不同的hierarchy，不同hierarchy之间绑定的subsystem不能相交。

+ 在同一个hierarchy下一个进程只能属于某一个特定的cgroup。

+ 进程隶属于每个hierarchy下的某个cgroup。

+ 新创建的进程默认位于父进程所在的cgroup中。


通过cat /proc/cgroups可以查看系统中所支持的subsystem以及这些subsystem与hierarchy之间的绑定信息，结果格式为:

| 列名 | 含义 |
| --- | --- |
| subsys\_name | subsystem |
| hierarchy | subsusytem所绑定的hierarhcy id，如果hierarchy id相同，说明subsystem绑定到了同一个hierarchy下。如上可以看出cpu，cpuacct都绑定到了hierarchy id: 2，net\_cls, net\_prio都绑定到了hierarchy id: 5 |
| num_cgroups | hierarchy下cgroup个数 |
| enabled | 是否开启 |

示例:

```bash
root@ke-build:~# cat /proc/cgroups
#subsys_name    hierarchy       num_cgroups     enabled
cpuset  10      1       1
cpu     2       86      1
cpuacct 2       86      1
blkio   11      86      1
memory  6       152     1
devices 4       86      1
freezer 8       1       1
net_cls 5       1       1
perf_event      3       1       1
net_prio        5       1       1
hugetlb 7       1       1
pids    9       86      1
```

如果知道进程id，可以通过cat /proc/<pid>/cgroups来获取进程所在的cgroup，结果格式为:

[hierarchy id]:[relative path under cgroup root]

示例:

```bash
root@ke-build:~# cat /proc/20710/cgroup
11:blkio:/docker/4ac93d829eb70dccd4bb8f8f3c200837bfcccb40e227b4316d8459593e434928
10:cpuset:/docker/4ac93d829eb70dccd4bb8f8f3c200837bfcccb40e227b4316d8459593e434928
9:pids:/docker/4ac93d829eb70dccd4bb8f8f3c200837bfcccb40e227b4316d8459593e434928
8:freezer:/docker/4ac93d829eb70dccd4bb8f8f3c200837bfcccb40e227b4316d8459593e434928
7:hugetlb:/docker/4ac93d829eb70dccd4bb8f8f3c200837bfcccb40e227b4316d8459593e434928
6:memory:/docker/4ac93d829eb70dccd4bb8f8f3c200837bfcccb40e227b4316d8459593e434928
5:net_cls,net_prio:/docker/4ac93d829eb70dccd4bb8f8f3c200837bfcccb40e227b4316d8459593e434928
4:devices:/docker/4ac93d829eb70dccd4bb8f8f3c200837bfcccb40e227b4316d8459593e434928
3:perf_event:/docker/4ac93d829eb70dccd4bb8f8f3c200837bfcccb40e227b4316d8459593e434928
2:cpu,cpuacct:/docker/4ac93d829eb70dccd4bb8f8f3c200837bfcccb40e227b4316d8459593e434928
1:name=systemd:/docker/4ac93d829eb70dccd4bb8f8f3c200837bfcccb40e227b4316d8459593e434928
```

## API接口

cgroup并未实现新的系统调用, 而是实现了一个新的文件系统类型`cgroup`，通过操作cgroup文件系统来实现对cgroup功能的调用。

+ 通过mount cgroup类型的filesystem来创建hierarchy，绑定subsystem。

```bash
mount -t cgroup -o cpu,cpuacct cpu,cpuacct /sys/fs/cgroup/cpu,cpuacct
ls /sys/fs/cgroup/cpu,cpuacct
```
-t 表示文件系统类型为cgroup,cgroup文件系统类型会将各种文件系统操作转换成对cgroup的功能调用

-o 表示要绑定的subsystem

+ 通过mount查看所创建的hierarchy

```bash
root@ke-build:~# mount|grep cgroup
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,mode=755)
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/lib/systemd/systemd-cgroups-agent,name=systemd) # 一个未绑定任何subsystem的hierarchy
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio) # 绑定了net_cls,net_prio subsystem的hierarchy
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
```

+ 通过mkdir/rmdir创建新的cgroup

```bash
# 创建一个新的cgroup
mkdir /sys/fs/cgroup/cpu,cpuacct/lrs
# 删除一个 cgroup，要保证这个cgroup下不child cgroup和进程
rmdir /sys/fs/cgroup/cpu,cpuacct/lrs
```

+ 通过write cgroup下的文件来调整subsystem的参数，通过read cgroup下的文件来查看 subsystem的配置参数和一些统计信息

```bash
# 将某个进程加入到lrs cgroup下
echo $REDIS_PID > /sys/fs/cgroup/cpu,cpuacct/lrs/cgroup.proc
# 调整lrs cgroup cpu.shares的参数
echo "1024" > /sys/fs/cgroup/cpu,cpuacct/lrs/cpu.shares
# 查看lrs cgroup对cpu的使用信息
cat /sys/fs/cgroup/cpu,cpuacct/lrs/cpu.stat
nr_periods 1642
nr_throttled 111
throttled_time 6381539723
```

当通过mount一个新的hierarchy(创建root cgroup)或通过mkdir创建一个新的cgroup时, cgroup目录下会自动出现一些cgroup的控制文件，通过读写这类控制文件实现对cgroup的参数调整，这里面文件分两类: 一类是各个subsystem通用的配置文件，可以认为是每类subsystem都要实现的通用接口，文件以cgroup.xxx命名(如cgroup.procs)；另一类是与subsystem具体实现相关的，文件以SUBSYSTEMNAME.xxx命名(如cpu.shares)。

cgroup通用配置文件:

| 文件名 | 功能 |
| ------| ------ |
| cgroup.clone_children | 为了给cpuset设置默认值ref: https://lkml.org/lkml/2010/7/29/368 |
| cgroup.procs | 这个cgroup下的进程id列表(无序) |
| cgroup.sane_behavior | 历史上部分subsystem的行为逻辑不是特别的合理(比方说没有很好的实现 hierarchy所表示的层级控制关系)，后续做了改进，但是为了保持行为的一致性，通过 sane_behavior 这个开关来选择是使用旧的行为逻辑(insane)，还是合理(sane)的行为逻辑。 |
| notify_on_release | 当这个cgroup下的最后一个进程退出时，是否要调用release_agent。 |
| release_agent | release_agent的路径 |
| tasks | 这个cgroup下的线程id |

## subsystem介绍

### cpu

cpu subsystem用于控制cgroup对cpu资源的使用，具体的逻辑在调度器中实现，不同的调度器实现的策略不同，这里主要介绍一下cfq调度器[2]所提供的两种策略: cpu-share, cpu-quota。
 
**cpu-share[3]**: 保证各个cgroup能够一定的比例使用cpu资源，这个策略的主要作用是，当cpu busy 时，满足cgroup对cpu需求的下限，保证各个cgroup都能够按照配置的比例，分配到cpu资源，对应的控制文件是cpu.shares。

以下图为例，简单介绍cpu.share的控制与计算逻辑:

![cpu-share](https://drive.google.com/uc?id=0B8b6aqj4rk9BWVBEMlJzcy12ZDA)

上图所示是一个绑定了cpu subsystem的hierarchy，`/cpu`为root cgroup，下面有两个child cgroup: `/cpu/A`、`/cpu/B`，A下面又有两个child cgroup: `A0`，`A1`。`/cpu`配置的cpu.share为1024，其含义是`/cpu`为root的hierarchy可以按照1024/1024的比例使用系统的cpu。`/cpu`下面分别包含两个子树: `/cpu/A`(cpu.share 4096)，`/cpu/B`(cpu.share 2048)，这里的意思是说`/cpu/A`和`/cpu/B`按照4096:2048的比例分配cpu资源。`/cpu/A`下面有2个子cgroup: `/cpu/A/A0`(cpu.share 512), `cpu/A/A1`(cpu.share 3584），意思是`/cpu/A/A0`，`/cpu/A/A1`这按照512:3584的比例分配cpu资源。hierarchy下的cgroup是按照树的方式进行组织的，所以 cpu-share的分配也会有一定的继承关系，以下表为例简单描述层次关系下cpu-share的分配规则:

| cgroup | 是否有进程 | 全局cpu.share |
| ------ | -------- | ------------- |
| /cpu | 无 | - |
| /cpu/A | 有 | (4096 / (4096 + 512 + 3584)) * (4096 / (4096 + 2048))|
| /cpu/A/A0 | 有 | (512 / (1024 + 512 + 1024)) * (4096 / (4096 + 2048))|
| /cpu/A/A1 | 有 | (3584 / (1024 + 512 + 1024) * (4096 / (4096 + 2048)))|
| /cpu/B | 有 | 2048 / (4096 + 2048)|


Q: 如果/cpu这个cgroup下有进程运行的话，如何分配cpu资源?

A:

| cgroup | 是否有进程 | 全局cpu.share |
| ------ | ---------------- |
| /cpu |(1024 / (1024 + 4096 + 2048))|
| /cpu/A |(4096 / (4096 + 512 + 3584) * (4096 / (1024 + 4096 + 2024)))|
| /cpu/A/A0 |(512 / (1024 + 512 + 1024)) * (4096 / (1024 + 4096 + 2048))|
| /cpu/A/A1 |(3584 / (1024 + 512 + 1024) * (4096 / (1024 + 4096 + 2048)))|
| /cpu/B |2048 / (1024 + 4096 + 2048)|

**cpu-quota[4]**: 对cgroup使用CPU带宽做限制，限制cgroup使用CPU的上限，主要是通过限制cgroup使用cpu的时间来限制。有两个配置文件: cpu.cfs\_period\_us，配置统计周期；cpu.cfs\_quota\_us，表示在统计周期内，可以用多长时间cpu。例如: cpu.cfs\_period\_us:100000，cpu.cfs\_quota\_us:200000，表示最多能用2个核；cpu.cfs\_period\_us:100000，cpu.cfs\_quota\_us:50000，表示最多能用0.5个核。[ref https://www.kernel.org/doc/Documentation/scheduler/sched-bwc.txt]

cpu-quota的继承关系比较简单，在设置上满足，
child cgroup cpu.cfs\_quota\_us <= parent cpu.cfs\_quota\_us(允许出现sum(child cgroup cpu.cfs\_quota\_us) > parent cpu.cfs\_quota\_us)，同时，在统计cgroup使用了多少个cpu.cfs\_period\_us时，会算上child cgroup的使用时间。当parent cgroup超了cpu.cfs\_quota\_us时，child cgroup跟着一起被throttling。

![cpu-quota](https://drive.google.com/uc?id=0B8b6aqj4rk9BNHhUSlFWNGVQRTQ)

如上图所示，root cgroup在`/cpu`，cpu.cpu\_quota_\us:-1表示不限制，`/cpu/A`，`/cpu/B` 分别配置了cpu.cfs\_quota_\us:500000，表示`/cpu/A`，`/cpu/B`这两个子树最多各用5个核，`/cpu/A/A0`配置cpu.cfs\_quota\_us:-1表示不限制，因为`/cpu/A/A0`是`/cpu/A`的child cgroup，所以实际使用时，还是会受到parent cgroup `/cpu/A` cpu.cfs\_quota\_us:500000的限制。

Q: 上图所示`/cpu/A` + `/cpu/B`限制了整个系统最多使用10个CPU，如果系统只有8核怎么办?

A: 首先，cpu.cfs\_quota\_us限制的是上限，如果`/cpu/A`, `/cpu/B`下的所有进程开足马力也没用到8核，那么，这里的限制并没有什么效果，没什么关系。如果`/cpu/A`，下面的进程开足了马力，在不受限制的情况下能用到8核，那么`/cpu/A`顶多用到5核，同时具体`/cpu/A`，与`/cpu/B`各自实际用到多少核还依赖两个cgroup cpu.shares的比例设置。

因为cpu-qutoa这个策略是个对cpu资源使用的硬性限制，当cgroup对cpu的使用超过了 cpu.cfs\_quota\_us 规定的上限，调度器会采用throttling的方式对cgroup的cpu使用做限制，直到满足cpu.cfs\_quota\_us的限制，针对超限的情况，cpu subsystem提供了cpu.stat 这个文件，用来暴露cgroup cpu.cfs\_quota\_us的统计信息，便于监控和调整配置。

cpu.stat文件说明:

| 指标 | 说明 |
|-----|------|
|nr_periods|number of period intervals (as specified in cpu.cfs_period_us) that have elapsed|
|nr_throttled|number of times tasks in a cgroup have been throttled (that is, not allowed to run because they have exhausted all of the available time as specified by their quota).|
|throttled_time|the total time duration (in nanoseconds) for which tasks in a cgroup have been throttled.|

示例:

```bash
root@ke-build:~# cat $CGROUP_DIR/cpu.stat
nr_periods 1642
nr_throttled 111
throttled_time 6381539723
```

### cpuset

cpuset subsystem用于控制绑核(这里的核其实是cpu线程[5])，也就是在多核cpu上，限制cgroup中的进程只能使用某些cpu核(绑核的同时也可以绑定memory node)，通过这样做，可以提高cpu cache命中率。对于numa架构[6]也支持绑定memory node，通过绑定memory node，可以避免跨memory node的访存，提高程序性能。主要通过两类控制文件:

**绑核相关**: cpuset.cpus & cpuset.cpu\_exclusive，cpuset.cpus指定cgroup能够使用的 cpu核，cpuset.cpu\_exclusive用于cgroup是否互斥使用cpuset.cpus中配置的cpu核。

**绑memory node相关**: cpuset.mems & cpuset.mem\_exclusive，cpuset.mems指定cgroup 所绑定的memory node，cpuset.mem\_exclusive用于表示是否与其他cgroup互斥使用这些cpuset.mems中配置的memory node。

与其他subsystem不同的一点在于，在创建新的cgroup时，系统并不会为新创建cgroup填充cpuset.cpus & cpuset.mems默认值，需要在设置好这两个必填参数之后，才能使用这个cgroup。(如果开启了 cgroup.clone_child，系统会根据parent cgroup的配置给新的cgroup设置cpuset.cpus & cpuset.mems默认值[7])


![cpuset](https://drive.google.com/uc?id=0B8b6aqj4rk9BdTNxSTVzZ0ZVWFU)

上图展示了cgroup之间cpuset的继承关系。root cgroup `/cpuset`允许使用cpu核0-7，因为cgroup`/cpuset`设置了cpuset.cpu\_exclusive:1，所以它的两个child cgroup `/cpuset/A`与`/cpuset/B`之间允许使用的cpuset.cpus 不能有重叠，`/cpuset/A`配置使用0-3，`/cpuset/B`允许使用4-7。`/cpuset/A`下面的有两个child cgroup `/cpuset/A/A0`，`/cpuset/A/A1`，因为`/cpuset/A`中设置cpuset.cpu_exclusive:0，所以`/cpuset/A`的child cgroup可以重叠使用 cpuset.cpus。

在使用cpuset功能前，可以先通过`lscpu`来查看一下机器上cpu核数，cpu核与memory node的绑定关系，预先做好规划。

```bash
root@ke-build:~# lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                40
...
NUMA node0 CPU(s):     0-9,20-29
NUMA node1 CPU(s):     10-19,30-39
...
```

### cpuacct

这个subsystem其实不是用来控制的，只是用来提供监控审计信息，统计各个cgroup对cpu的使用情况。

### memory[8]

memory subsystem用来统计控制cgroup中进程对内存的使用，主要包含3类内存控制策略:

**ann + cache**: 限制cgroup中进程anonymous pages(heap, stack, anonymous mmap)(不能reclaim)，file caches(page cache)(可以reclaim，共享的page cache按照first touch first charge的原则计算归属）的使用，通过memory.limit\_in\_bytes来配置限制，通过memory.usage\_in\_bytes来查看当前使用信息。当memory.usage\_in\_bytes超过 memory.limit\_in\_bytes时，会通过reclaim进程的内存或者oom\_kill的方式确保cgroup内存使用不超过 memory.limit\_in\_bytes。另外可以通过memory.failcnt来查看memory.usage\_in\_bytes达到memory.limit\_in\_bytes的次数。

**ann + cache + swap**: 在上面的基础上增加对swap的限制，通过memory.memsw.limit\_in\_bytes来配置限制，通过memory.memsw.usage\_in\_bytes查看当前使用信息，通过memory.memsw.failcnt查看memory.memsw.usage\_in\_bytes超过 memory.memsw.limit\_in\_bytes的次数。对于未开启swap的系统，memory.memsw.xx对应的操作文件不存在，无需配置。

**kernel memory**: 限制cgroup中进程对tcp buffer，dentry[9]等内核内存的使用，memory.kmem.xxx文件的功能与上述两种控制策略类似。

除了上述控制策略外，memory subsystem还提供了一个memory.stat接口(文件)，用于获取对应cgroup对内存使用的统计信息。

memory subsystem对于cgroup继承上的控制体现在(开启了memory.use_hierarchy): reclaim的策略，当某个cgroup的内存使用触发了xxx.limit_in_bytes时，所触发的reclaim会reclaim child cgroup中的内存；内存计量会计量child cgroup。

### pids

pid subsystem用来控制统计cgroup下运行的进程个数，主要有两个控制文件:

**pids.max**: 控制cgroup(含child cgroup)可以运行的进程数。

**pids.current**: 统计当前cgroup(含child cgroup)正在运行的进程数。

注意，因为Linux中线程是采用进程实现的，线程，进程在 Linux中使用的是同样的数据结构，也是通过同样的系统调用创建的[10]，所以实际上pids.max，pids.current限制和统计的是系统中的线程数。

### blkio

blkio subsystem用来控制cgroup对io的使用，与cpu类似，blkio也实现了两类控制策略:

**weight**: 按比例分配cgroup对io的使用，与cpu-share类似，weight策略依赖cfq ioscheduler实现，如果使用的是其他的io scheduler，则不会有这种策略。

**throttling**: 控制cgroup使用某个设备的带宽，与cpu-quota类似，这个策略实现在block layer，与使用的io scheduler无关。

### device

device subsystem用于控制cgroup mknode/read/write设备文件的权限。主要包含3个接口: 通过devices.allow增加对device的操作权限，通过devices.deny来移除的操作device权限，通过 devices.list来查看这个cgroup下能够使用的各个devices以及使用权限。

### freezer

freezer subsystem用于pause/resume cgroup下的进程，通过向freezer.state写入`FROZEN`来suspend cgroup(以及child cgroup)下的进程，通过向 freezer.state 写入`THAWED`来 resume cgroup(以及child cgroup)下的进程。[CRIU](https://criu.org/Main_Page)利用freezer subsystem实现对Linux进程的checkpointing & resume，搭配pid namespace实现将机器A上的进程迁移到机器B上。

### net_cls

net_cls subsystem 用于给从cgroup出去的网络流量打一个classid，搭配其他的Linux网络组件实现对网络流量的控制(tc, iptables)。

### net_prio

设置cgroup对各个网卡使用的权重分配，限制从cgroup流出的网络流量。

### hugetlb

hubetlb subsystem用于控制cgroup对huge page的使用，具体使用姿势，应用场景待进一步调研学习。

### perf_event

perf_event subsystem，可以使perf tool以cgroup为monitor一组进程的运行指标，具体使用姿势，应用场景待进一步调研学习。

## 总结

本文主要介绍了cgroup的基本概念和功能，通过hierarchy的来层级的组织cgroup，搭配subsystem实现由粗到细的资源划分和限制，容器系统利用cgroup来实现对容器使用资源做限制。

附上一个kubelet节点上cpu hierarchy下cgroup的组织关系，通过学习kubernetes如何基于 hierarchy实现Guaranteed，Besteffort，Burstable三类QoS Class加深对cgroup功能和使用姿势的理解:-)。

```bash
tree /sys/fs/cgroup/cpu/kubepods
# 因为篇幅省略了一些其他树节点
/sys/fs/cgroup/cpu/kubepods/
├── besteffort
│   ├── cpu.shares
│   ├── pod813f4947-914d-11e7-b0a6-6c92bf3068da
│   │   ├── 0820d4c0f4f1e27869d72163c667ca22a06d67a5e10f1b79988c79861523641b
│   │   │   ├── cpu.shares
│   │   ├── cpu.shares
│   │   ├── e976587f2b5d9869a8c5f1687ce4e60af2f369680f9d776a546b4d6c78065c41
│   │   │   ├── cpu.shares
│   ├── pod81449e64-914d-11e7-b0a6-6c92bf3068da
│   │   ├── 865e0d7a2c8a2449f0411dde22c71a33a10bb2332538097b9cd17a439f21ec08
│   │   │   ├── cpu.shares
│   │   ├── cpu.shares
│   │   ├── f5f656d97cff811dfa60f785cbe00f207882a9ca94d3b804b136038d28571a4c
│   │   │   ├── cpu.shares
├── burstable
│   ├── cpu.shares
│   ├── pod1d2c7cf4-a2a5-11e7-b54f-00e0ed5d441c
│   │   ├── 64d03d96cb1a242e9f429085762c6d0ccaa217b754515038c1af767760fd9675
│   │   │   ├── cpu.shares
│   │   ├── cpu.shares
│   │   ├── d29564e94644ae7bfa9a2039c1d071071647427f7912cb3c2f9ac2186b25a60c
│   │   │   ├── cpu.shares
│   ├── pod38958a3e-93a7-11e7-b54f-00e0ed5d441c
│   │   ├── 86efda5489eceb56d1968638fb25b84d716758ab2a061c121b73f4c8847b816c
│   │   │   ├── cpu.shares
│   │   ├── c21cb1ebda499a07cc42719c4484797e70f982c115213906fbf19fc79d383375
│   │   │   ├── cpu.shares
│   │   ├── cpu.shares
├── cpu.shares
├── cpu.stat
├── notify_on_release
├── podf2cd1220-ad9c-11e7-b54f-00e0ed5d441c
│   ├── 4ab422bb3a30a902036ea29258af886d0ef44d8dfa2b442c8aead6fc6c40b208
│   │   ├── cpu.shares
│   ├── b6f78ea2d5ce380a3cad95f73e4ba35255aabb2225278721de0cc9f07b19f751
│   │   ├── cpu.shares
```




[0]. [Terminology](http://man7.org/linux/man-pages/man7/cgroups.7.html)

[1]. [RELATIONSHIPS BETWEEN SUBSYSTEMS, HIERARCHIES, CONTROL GROUPS AND TASKS](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/sec-relationships_between_subsystems_hierarchies_control_groups_and_tasks)

[2]. https://www.kernel.org/doc/Documentation/scheduler/sched-design-CFS.txt, 7.  GROUP SCHEDULER EXTENSIONS TO CFS

[3]. Linux Kernel Development 3rd, Part 4 Process Scheduling

[4]. https://www.kernel.org/doc/Documentation/scheduler/sched-bwc.txt

[5]. Computer Systems - A Programmer's Perspective 2nd, Part 4 Processor Architecture

[6]. https://software.intel.com/en-us/articles/optimizing-applications-for-numa

[7]. https://lkml.org/lkml/2010/7/29/368

[8]. Computer Systems - A Programmer's Perspective 2nd, Part 9 Virtual Memory

[9]. https://sysdig.com/blog/container-isolation-gone-wrong/

[10]. Linux Kernel Development 3rd, Part 3 Process Management

关于各个subsystem的内容，来自https://www.kernel.org/doc/Documentation/cgroup-v1/
