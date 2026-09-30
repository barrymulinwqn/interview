# Linux Shell 性能与运维排障指南

> 面向 Linux 运维工程师、SRE、DevOps、Shell 工程师和后端架构师。本文重点回答三个问题：当前系统发生了什么、命令输出应该如何解读、如何从现象逐步定位到根因。
>
> 命令默认以 Linux/GNU 工具链为准。不同发行版、内核版本、容器运行时和权限配置会影响输出；排障时先确认环境，再记录采样时间、主机、容器、服务版本和业务流量。

## 目录

1. [性能排障总方法](#一性能排障总方法)
2. [环境与基线](#二环境与基线)
3. [系统总体状态](#三系统总体状态)
4. [CPU 与 Load Average](#四cpu-与-load-average)
5. [内存与 Swap](#五内存与-swap)
6. [磁盘与文件系统](#六磁盘与文件系统)
7. [网络与连接](#七网络与连接)
8. [进程、线程与文件描述符](#八进程线程与文件描述符)
9. [应用级深度定位](#九应用级深度定位)
10. [容器与 cgroup 性能](#十容器与-cgroup-性能)
11. [常见性能症状到根因](#十一常见性能症状到根因)
12. [一键采样脚本示例](#十二一键采样脚本示例)
13. [面试高频问题](#十三面试高频问题)
14. [排障自检清单](#十四排障自检清单)

---

## 一、性能排障总方法

### 1. 先定义问题，而不是先执行命令

先把模糊描述变成可测量的问题：

- 是平均延迟升高，还是 P95/P99 长尾升高？
- 是所有请求变慢，还是某个接口、租户、机器或版本变慢？
- 是吞吐下降、错误率上升、队列堆积，还是资源利用率升高？
- 是从什么时候开始，是否对应发布、流量变化、配置变化、依赖故障或定时任务？
- 影响的是主机、容器、进程、线程、磁盘、网络，还是某个外部依赖？

### 2. 用四类证据交叉验证

| 证据类型 | 主要问题 | 常用工具 |
| --- | --- | --- |
| 业务指标 | 用户是否真的受影响 | APM、接口延迟、错误率、吞吐、队列 |
| 系统指标 | 哪类资源可能成为瓶颈 | `uptime`、`vmstat`、`iostat`、`sar`、`ss` |
| 进程指标 | 哪个进程或线程消耗资源 | `top`、`ps`、`pidstat`、`pmap`、`lsof` |
| 内核与调用证据 | 进程正在等待或执行什么 | `/proc`、`strace`、`perf`、日志、内核事件 |

单个指标只能提出假设，不能直接证明根因。例如 CPU 使用率 90% 可能是业务计算、内核网络处理、虚拟机 steal，或者只是某个短时采样点。

### 3. 推荐排障顺序

```text
用户症状
  -> 时间线和影响范围
  -> 主机总体状态
  -> CPU / 内存 / 磁盘 / 网络四大资源
  -> 具体进程和线程
  -> 系统调用、锁、I/O、GC 或下游依赖
  -> 修复、验证、回归和复盘
```

排查应尽量从低成本、低侵入命令开始，再进入 `strace`、`perf`、火焰图或 heap dump 等高成本工具。生产环境执行诊断命令前要确认权限、采样时长和性能影响。

### 4. 记录每次采样

```bash
date -Is
hostname
uptime
```

每条输出都要带采样时间。性能问题是时间序列问题，单次命令输出很容易错过峰值或把恢复后的状态误认为根因。

---

## 二、环境与基线

### Q1：如何确认当前 Linux 环境？

```bash
uname -a
uname -r
cat /etc/os-release
hostnamectl 2>/dev/null || true
arch
```

#### 输出解读

- `uname -r`：内核版本；不同内核的调度器、cgroup、网络和文件系统特性可能不同。
- `/etc/os-release`：发行版和版本，例如 Ubuntu、RHEL、Amazon Linux。
- `arch`：CPU 架构，例如 `x86_64`、`aarch64`。
- `hostnamectl`：主机名、虚拟化类型和内核信息，部分精简系统没有该命令。

如果发现命令来自容器，应进一步确认宿主机内核、容器限制和 namespace 视图。容器内的 `uname` 通常显示宿主机内核，而进程、挂载和网络可能是容器视图。

### Q2：如何确认 CPU、内存和磁盘基线？

```bash
lscpu
nproc
free -h
lsblk -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINTS
df -hT
```

重点记录：

- CPU 逻辑核数、物理核数、Socket、NUMA 节点和频率策略。
- 总内存、可用内存、Swap、文件系统类型和挂载参数。
- 磁盘是本地 SSD、HDD、网络盘还是云盘，容量和 IOPS 上限是什么。
- 应用进程和容器实际可用资源是否小于主机总资源。

“主机有 32 核”不代表进程可以使用 32 核；CPU affinity、cgroup quota、容器限制和 NUMA 绑定都可能缩小可用范围。

### Q3：如何查看资源限制？

```bash
ulimit -a
cat /proc/$$/limits
systemctl show my-service -p LimitNOFILE -p CPUQuotaPerSecUSec -p MemoryMax
```

常见限制包括：

- `Max open files`：进程可打开的文件描述符数量。
- `Max processes`：用户或进程可创建的线程/进程数量。
- `Max locked memory`：锁定内存限制。
- systemd `MemoryMax`、`CPUQuota`、`TasksMax`：服务级 cgroup 限制。

应用看起来“资源还够”但仍然报 `Too many open files`、无法创建线程或被 OOM 杀死时，优先检查限制，而不是只看宿主机总量。

### Q4：如何建立正常基线？

在业务正常时定期采集：

- CPU：平均、峰值、用户态、内核态、iowait、steal。
- Load：1/5/15 分钟值和 CPU 核数。
- 内存：available、swap in/out、major page fault、RSS。
- 磁盘：吞吐、IOPS、await、队列长度、util、空间和 inode。
- 网络：吞吐、丢包、错误、重传、连接数、TIME_WAIT。
- 应用：QPS、P50/P95/P99、错误率、线程池、连接池、GC 和队列。

没有基线时，不要把某个绝对数值直接定义为异常。CPU 90% 对批处理可能正常，对低延迟 API 可能危险；磁盘 `util=100%` 对顺序吞吐可能正常，对随机低延迟业务则通常意味着排队。

---

## 三、系统总体状态

### Q5：`uptime` 和 `/proc/loadavg` 如何解读？

```bash
uptime
cat /proc/loadavg
```

典型输出：

```text
10:30:00 up 12 days,  4:10,  2 users,  load average: 2.40, 1.80, 1.20
```

三个 load 值分别表示过去 1、5、15 分钟的平均负载。Linux load average 主要统计处于可运行状态和不可中断睡眠状态的任务，后者常见于等待磁盘或某些内核 I/O 操作，因此 load 高不等于 CPU 一定高。

解读时要和逻辑 CPU 数比较：

- 4 核机器 load 长期 2：通常仍有余量，但要结合 I/O 和延迟。
- 4 核机器 load 长期 8：说明平均有明显排队，可能是 CPU runnable 或 I/O 阻塞。
- load 从 1/5/15 分钟逐渐升高：压力可能正在增加。
- 1 分钟高、5/15 分钟低：可能是短时尖峰或定时任务。
- load 高但 CPU idle 很高：优先检查 I/O、锁、NFS、磁盘和不可中断任务。

不要用“load 大于 1 就故障”的规则。关键是 load 与 CPU 核数、业务延迟、`vmstat` 的 `r/b`、磁盘等待和进程状态结合。

### Q6：`top` 的关键字段如何解读？

```bash
top
```

常见 CPU 行：

```text
%Cpu(s): 35.0 us, 8.0 sy, 0.0 ni, 45.0 id, 10.0 wa, 0.0 hi, 2.0 si, 0.0 st
```

| 字段 | 含义 | 高值时关注 |
| --- | --- | --- |
| `us` | 用户态执行时间 | 应用计算、解释器、业务循环 |
| `sy` | 内核态执行时间 | 系统调用、网络、内核锁、驱动 |
| `ni` | 调整过 nice 的用户进程时间 | 批处理或低优先级任务 |
| `id` | 空闲时间 | 越低表示 CPU 越忙 |
| `wa` | 等待 I/O 的时间 | 磁盘、网络存储、块设备 |
| `hi` | 硬件中断 | 网卡、磁盘等硬件中断 |
| `si` | 软件中断 | 网络软中断、内核处理 |
| `st` | 虚拟化 steal time | 宿主机争抢 vCPU |

进程区常看：

- `%CPU`：进程或线程 CPU 使用率；多核系统中可能超过 100%。
- `%MEM`：占物理内存比例，不等于进程完整内存成本。
- `RES`：驻留物理内存，通常比 `VIRT` 更接近实际 RAM 使用。
- `VIRT`：虚拟地址空间，包含映射、共享库、保留空间，不应直接当作物理内存。
- `S`：进程状态，例如 `R` 运行、`S` 可中断睡眠、`D` 不可中断睡眠、`Z` 僵尸。
- `TIME+`：累计 CPU 时间，不是当前瞬时使用率。

### Q7：`vmstat` 如何判断 CPU 还是 I/O 瓶颈？

```bash
vmstat 1 10
```

常见字段：

| 字段 | 含义 | 解读 |
| --- | --- | --- |
| `r` | 可运行任务数 | 持续明显高于 CPU 核数，说明 CPU 排队 |
| `b` | 不可中断睡眠任务数 | 高时关注磁盘、NFS、块设备和内核 I/O |
| `swpd` | 使用的 Swap 大小 | 只看这个不能判断当前是否在抖动 |
| `free` | 空闲内存 | Linux 会积极使用缓存，不能单独作为内存健康指标 |
| `buff` | buffer 使用量 | 传统块设备元数据缓冲 |
| `cache` | page cache 等缓存 | 回收后可用于应用 |
| `si`/`so` | Swap 换入/换出速率 | 持续非零且伴随延迟时需关注内存压力 |
| `bi`/`bo` | 块设备读/写块速率 | 结合磁盘设备和业务判断 |
| `in` | 中断次数 | 高时关注网卡、磁盘和设备负载 |
| `cs` | 上下文切换次数 | 高时关注线程数量、锁竞争和调度压力 |
| `us`/`sy` | 用户/内核 CPU | 区分应用计算和内核工作 |
| `wa` | I/O wait | 等待块设备完成的 CPU 时间 |
| `st` | steal | 虚拟机被宿主机抢占的时间 |

判断例子：

- `r` 高、`us` 高、`id` 低：可能是 CPU 密集或线程过多。
- `b` 高、`wa` 高、`si/so` 低：可能是磁盘或网络存储等待。
- `si/so` 持续高、`free`/`available` 低：内存压力导致换页。
- `cs` 极高但 CPU 不高：可能是大量线程唤醒、锁竞争或高频短任务。

### Q8：`sar` 如何做历史和持续采样？

```bash
sar -q 1 5
sar -u 1 5
sar -r 1 5
sar -W 1 5
sar -b 1 5
sar -n DEV 1 5
```

常见用途：

- `sar -q`：运行队列、进程创建和 load。
- `sar -u`：CPU 用户态、系统态、iowait、steal。
- `sar -r`：内存和 page fault。
- `sar -W`：Swap 换入换出。
- `sar -b`：块设备 I/O 概览。
- `sar -n DEV`：网卡吞吐、包和错误。

`sysstat` 服务启用后可以查看历史数据，例如：

```bash
sar -u -f /var/log/sa/sa28
```

如果故障发生在过去而没有历史采样，只能依赖应用监控、日志、云监控或内核日志事后还原。生产环境应提前配置采样保留周期，而不是故障发生后才安装工具。

---

## 四、CPU 与 Load Average

### Q9：如何找出最耗 CPU 的进程和线程？

```bash
ps -eo pid,ppid,tid,stat,psr,pcpu,pmem,etime,comm,args --sort=-pcpu | head -n 20
pidstat -u -t -p ALL 1 5
```

定位步骤：

1. 先找出持续高 `%CPU` 的 PID。
2. 用 `pidstat -t` 或 `top -H -p <pid>` 查看线程。
3. 记录高 CPU 线程 TID，并转换为十六进制，供 Java、Go、Python 或 native profiler 使用：

```bash
tid=${1:-1234}
printf '%x\n' "$tid"
```

4. 结合应用线程名、日志、堆栈或 `perf` 判断线程在做什么。

CPU 高不代表一定是“死循环”。可能是压缩、加密、正则、JSON 序列化、GC、系统调用、网络软中断或某个批处理任务。

### Q10：`pidstat` 如何用于按进程定位？

```bash
pid=${1:-1234}
pidstat -p "$pid" -u -r -d -w 1 10
```

- `-u`：用户态/系统态 CPU、等待、上下文。
- `-r`：缺页、内存和 RSS 变化。
- `-d`：读写 I/O。
- `-w`：自愿和非自愿上下文切换。

解释示例：

- `cswch/s` 高：进程主动等待 I/O、锁或条件变量的频率高。
- `nvcswch/s` 高：被调度器强制切换，可能 CPU 争抢或时间片用尽。
- `kB_rd/s` 高：进程发起大量读取，但还要确认是否命中缓存和设备延迟。
- `iodelay` 高：进程等待 I/O 的时间增加。

### Q11：`perf top` 和 `perf record` 如何使用？

```bash
pid=${1:-1234}
sudo perf top -p "$pid"
sudo perf record -F 99 -p "$pid" -g -- sleep 30
sudo perf report
```

- `perf top`：实时查看 CPU 栈上热点符号。
- `perf record`：采样并保存数据。
- `-F 99`：采样频率，过高会增加开销。
- `-g`：记录调用栈，要求符号和栈展开条件合适。

输出中某个函数占比高，说明 CPU 样本经常落在该函数附近，不等于函数每次调用都慢，也不自动证明它是根因。要结合请求量、输入规模、锁等待、下游延迟和优化前后对比。

权限受限时可能遇到 `perf_event_paranoid`、容器能力或内核配置问题。线上使用前要评估采样开销，并在低频率、短时长下开始。

### Q12：CPU 高但业务吞吐没有增加，如何定位？

优先排查：

1. 单线程热点：一个线程 100%，其他线程闲置，可能存在锁、分片或事件循环瓶颈。
2. 锁竞争：CPU 用在自旋、上下文切换或频繁唤醒，而不是有效业务工作。
3. GC 或内存分配：CPU 用于回收和对象创建，吞吐反而下降。
4. 系统调用和网络软中断：`sy`、`si` 高，可能是网络包、连接或内核处理压力。
5. 错误重试风暴：请求失败后重复计算或重复调用下游。
6. CPU 限额：容器被 cgroup throttling，业务线程在 quota 中排队。

验证方法包括 `pidstat -t`、`perf`、应用 profiler、线程 dump、cgroup `cpu.stat` 和请求级指标。不要看到 CPU 高就只增加线程数，线程数可能进一步增加调度和锁竞争。

### Q13：Load 高但 CPU 使用率不高意味着什么？

Linux load 包含运行队列和不可中断睡眠任务。CPU idle 仍高时，常见原因包括：

- 磁盘或网络文件系统 I/O 阻塞，进程处于 `D` 状态。
- NFS、块设备或存储阵列响应慢。
- 内核等待资源、设备或锁。
- 采样时 CPU 已经恢复，但阻塞任务的平均值仍未下降。

检查：

```bash
ps -eo state,pid,ppid,wchan:32,comm | awk '$1 ~ /D/ { print }'
vmstat 1 5
iostat -xz 1 5
```

`wchan` 可以提供等待点线索，但不同内核、权限和符号配置下信息可能有限。

### Q14：CPU steal time 高说明什么？

虚拟机中的 `st` 表示 vCPU 想运行但被宿主机调度给其他虚拟机的时间。`st` 持续高说明宿主机超卖、实例类型受限或云平台底层争抢，应用即使自身没有做更多工作也会变慢。

验证：

```bash
mpstat -P ALL 1 5
vmstat 1 5
```

处理方式包括迁移实例、调整实例规格、检查云厂商 CPU credit、减少 vCPU overcommit，或将延迟敏感服务放到更稳定的计算资源。不要通过应用线程调优掩盖宿主机资源争抢。

---

## 五、内存与 Swap

### Q15：`free -h` 应该怎么看？

```bash
free -h
```

典型字段：

- `total`：可见物理内存总量。
- `used`：已使用内存的传统统计口径，不能单独判断是否不足。
- `free`：当前完全未使用的内存。
- `shared`：常见为 tmpfs 等共享内存。
- `buff/cache`：内核 buffer 和 page cache，可以在需要时回收一部分。
- `available`：内核估算在不发生 Swap 的情况下可供新应用使用的内存，更适合判断短期可用性。
- `Swap used`：已经使用的 Swap；只看非零并不代表当前正在发生严重换页。

```text
               total        used        free      shared  buff/cache   available
Mem:            16Gi        10Gi       500Mi       200Mi         5Gi         6Gi
Swap:            2Gi       100Mi       1.9Gi
```

这个例子中 `free` 很小但 `available` 仍有 6 GiB，不能简单结论为内存耗尽。应结合 `vmstat` 的 `si/so`、major fault、进程 RSS 和 OOM 日志判断是否有实际内存压力。

### Q16：如何找出内存占用最大的进程？

```bash
ps -eo pid,ppid,user,%mem,rss,vsz,stat,etime,comm,args --sort=-rss | head -n 20
top -o %MEM
```

- `RSS`：进程当前驻留在物理内存中的页，通常是第一观察值。
- `VSZ`：虚拟地址空间，包含共享映射和未驻留页，不能直接当作 RAM 使用。
- 多进程服务中共享库、共享内存会让简单 RSS 相加高估真实物理占用。
- 容器中要同时看 cgroup memory usage，主机 `ps` 不能替代容器指标。

如果 RSS 持续上涨，记录时间序列，区分流量增长、缓存增长、请求堆积、内存泄漏、碎片和正常 page cache。

### Q17：如何查看 `/proc/meminfo`？

```bash
grep -E '^(MemTotal|MemFree|MemAvailable|Buffers|Cached|SReclaimable|SwapTotal|SwapFree|Dirty|Writeback|AnonPages|Mapped):' /proc/meminfo
```

关注：

- `MemAvailable`：估算可供应用使用的内存。
- `AnonPages`：匿名页，通常来自进程堆和栈。
- `Cached`、`SReclaimable`：可回收缓存的一部分。
- `Dirty`、`Writeback`：等待写回或正在写回磁盘的页。
- `SwapFree`：Swap 剩余量，需与换入换出速率一起看。

内核版本会影响字段定义和估算方式，不能把每个字段简单相加。要判断应用泄漏，优先结合进程 RSS、heap/profile、分配速率和 GC，而不是只看 `Cached`。

### Q18：什么时候 Swap 使用是问题？

Swap 不是“只要使用就故障”。冷数据被换出、系统仍有足够 `MemAvailable` 且没有持续换页时，影响可能有限。问题通常出现在：

```bash
vmstat 1 10
sar -W 1 10
```

`si`/`so` 持续非零，同时出现响应延迟、major fault、磁盘 I/O 和 `available` 下降，说明内存压力正在影响工作集。

排查顺序：

1. 找出增长最快的进程和容器。
2. 检查缓存是否无界、请求是否堆积、线程/连接是否过多。
3. 检查 cgroup memory limit 和 OOM 事件。
4. 查看 `vm.swappiness` 作为调度倾向，不要把它当作内存扩容替代品。
5. 评估增加内存、限制缓存、降低并发或修复泄漏。

### Q19：如何排查 OOM Killer？

```bash
dmesg -T | grep -i -E 'out of memory|killed process|oom'
journalctl -k -g 'oom|out of memory|killed process' --no-pager
```

查看：

- 被杀进程 PID、RSS 和命令行。
- 是系统级 OOM 还是容器 cgroup OOM。
- `oom_score_adj` 是否让某进程更容易被选择。
- 发生前是否有 Swap、page cache、容器 memory limit 或瞬时峰值。

修复不能只把内存限制调大。要确认应用峰值、泄漏、缓存上限、请求体、批量大小和进程模型，并建立 OOM 告警和自动收集现场证据。

### Q20：共享内存和容器 `/dev/shm` 如何排查？

```bash
df -h /dev/shm
ipcs -m
ls -lh /dev/shm
```

某些数据库、浏览器、Python multiprocessing 或 IPC 场景依赖共享内存。`/dev/shm` 满可能导致应用报错，即使普通磁盘和内存还有空间。容器默认共享内存通常较小，应按应用模型配置 `--shm-size` 或 Kubernetes 的 `emptyDir` memory medium，并限制使用量。

---

## 六、磁盘与文件系统

### Q21：`df -h`、`df -i` 和 `du` 如何配合？

```bash
df -hT
df -i
du -xdev -h --max-depth=1 /var 2>/dev/null | sort -h
```

- `df -hT`：文件系统整体字节空间和类型。
- `df -i`：inode 使用率；小文件过多时 inode 可能先耗尽。
- `du`：目录树中可见文件的累计大小。

`df` 和 `du` 不一致时，常见原因是文件已删除但仍被进程打开：

```bash
lsof +L1
```

删除目录项不会立即释放仍被文件描述符引用的空间。正确处理通常是让进程关闭或重新打开文件，而不是反复执行 `rm`。

### Q22：`iostat -xz` 的字段如何解读？

```bash
iostat -xz 1 5
```

关键字段：

| 字段 | 含义 | 判断方向 |
| --- | --- | --- |
| `r/s`、`w/s` | 每秒读写请求数 | IOPS 负载 |
| `rkB/s`、`wkB/s` | 每秒读写吞吐 | 带宽负载 |
| `await` | 请求从提交到完成的平均时间 | 延迟，包含队列等待 |
| `r_await`、`w_await` | 读/写平均延迟 | 区分读写瓶颈 |
| `aqu-sz` | 平均请求队列长度 | 持续升高说明排队 |
| `%util` | 设备忙碌时间比例 | 接近 100% 说明设备持续忙，但不等于一定饱和 |

判断例子：

- `await` 高、`aqu-sz` 高、`%util` 接近 100%：设备或后端存储可能饱和。
- `%util` 高但 `await` 低：设备可能能稳定处理，业务未必有明显延迟。
- 吞吐低但 `await` 高：可能是随机 I/O、存储后端抖动或队列排队。
- 设备指标正常但应用 I/O 慢：检查应用锁、文件系统、NFS、同步写和单线程串行化。

不要只看 `%util`；云盘、SSD、RAID 和虚拟块设备的含义不同。

### Q23：如何区分磁盘空间问题和 I/O 性能问题？

- 空间问题：`df -h` 接近 100%，写入报 `No space left on device`。
- inode 问题：`df -i` 接近 100%，大量小文件无法创建。
- I/O 性能问题：`iostat` 的 `await`/`aqu-sz` 高，业务延迟上升，可能空间仍充足。
- 文件系统或挂载问题：出现只读、错误、NFS 超时或内核日志。

```bash
mount | column -t
findmnt -T /var/log
journalctl -k -p warning..alert --no-pager
```

处理空间问题前要定位增长来源、保留策略和业务所有者，避免删除正在使用的数据库文件、容器层或审计证据。

### Q24：如何排查高 I/O 进程？

```bash
pidstat -d -p ALL 1 5
iotop -oPa
lsof +D /var/log 2>/dev/null | head
```

`iotop` 需要权限并可能不存在。`pidstat -d` 适合低侵入地看进程读写速率；`lsof` 可以帮助定位文件，但递归扫描目录可能有成本。

高 I/O 的常见来源：日志过量、批量扫描、数据库 checkpoint、备份、压缩、page cache 回写、临时文件和监控采集。要结合设备延迟和应用行为判断是否真正影响用户。

### Q25：如何排查 NFS 或网络文件系统卡顿？

```bash
mount | grep -E 'nfs|cifs'
nfsstat -m 2>/dev/null || true
ps -eo state,pid,wchan:32,comm | awk '$1 == "D" { print }'
```

进程长期处于 `D` 状态、load 高但 CPU idle 多，可能与远程文件系统响应、网络、服务端负载或锁有关。不要随意 `umount -f` 正在被业务使用的挂载点；先确认依赖、超时、重挂载风险和数据一致性。

### Q26：文件系统缓存如何影响性能？

Linux 会使用空闲内存作为 page cache，加速重复读取。第一次读取可能产生设备 I/O，后续读取可能命中内存。`free` 变小不一定是内存泄漏，`buff/cache` 在需要时通常可回收。

```bash
sync
# 不要在生产环境随意执行以下命令
# echo 3 | sudo tee /proc/sys/vm/drop_caches
```

清 cache 只能用于受控实验，不能作为线上修复手段。它会造成缓存冷启动和额外 I/O，可能让业务更慢。

---

## 七、网络与连接

### Q27：`ss` 如何查看监听端口和连接状态？

```bash
ss -lntp
ss -s
ss -tan state established
ss -tan state time-wait | wc -l
```

- `-l`：监听 socket。
- `-n`：不解析 DNS，输出更快且避免 DNS 干扰。
- `-t`：TCP。
- `-p`：显示关联进程，通常需要权限。
- `ESTAB`：已建立连接。
- `TIME-WAIT`：主动关闭连接后等待，数量高可能表示短连接过多，但不自动代表故障。
- `SYN-SENT`：客户端发起连接后等待响应，关注网络和服务端入口。
- `SYN-RECV`：服务端收到 SYN 等待完成，过多可能涉及 backlog、攻击或服务端处理。

先看总量和变化趋势，再按端口、进程、远端地址分组，不要只看一个状态数量。

### Q28：如何统计连接状态和远端来源？

```bash
ss -tan | awk 'NR > 1 { count[$1]++ } END { for (state in count) print state, count[state] }'
ss -tan state established | awk 'NR > 1 { print $5 }' | sort | uniq -c | sort -nr | head
```

输出解析依赖 `ss` 版本和列布局，生产脚本尽量使用稳定的 JSON 或结构化接口。连接数增长可能来自流量增长、连接池泄漏、服务端响应变慢、客户端重试或 TIME_WAIT 累积。

### Q29：网络吞吐、丢包和错误如何查看？

```bash
ip -s link
sar -n DEV 1 5
ethtool -S eth0 2>/dev/null | grep -i -E 'drop|error|timeout|miss'
```

关注：

- RX/TX bytes 和 packets：吞吐与包速率。
- dropped：内核或网卡丢弃的包。
- errors、CRC、carrier：链路、驱动、网卡或物理层问题。
- softnet backlog、softirq：CPU 处理网络包的能力。

虚拟网卡和云环境不一定提供完整硬件计数器。要结合交换机、云网络监控、应用重传和 TCP 指标判断。

### Q30：如何查看 TCP 重传和协议统计？

```bash
nstat -az 2>/dev/null | grep -E 'Tcp|Ip'
netstat -s 2>/dev/null | grep -i -E 'retrans|listen|failed|reset'
```

TCP 重传升高可能由网络丢包、拥塞、接收端处理慢、队列满或 MTU 问题引起。连接重置可能是应用主动关闭、代理超时、防火墙或进程崩溃。不要看到重传就直接判断网卡坏了，应结合两端抓包和路径监控。

### Q31：`curl` 如何区分 DNS、TCP、TLS 和服务端响应耗时？

```bash
curl -sS -o /dev/null \
    -w 'dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} first_byte=%{time_starttransfer} total=%{time_total}\n' \
    --connect-timeout 3 --max-time 10 \
    https://example.com/health
```

- `time_namelookup`：DNS 解析完成。
- `time_connect`：TCP 连接完成。
- `time_appconnect`：TLS 握手完成。
- `time_starttransfer`：收到首字节，包含服务端处理和网络等待。
- `time_total`：完整请求完成。

如果 DNS 时间高，检查 resolver、缓存和搜索域；TCP 时间高，检查网络路径和连接建立；TTFB 高，检查服务端队列、数据库和下游；total 高但 TTFB 正常，可能是响应体大或带宽受限。

### Q32：如何使用 `tcpdump` 做低风险网络确认？

```bash
sudo timeout 30 tcpdump -i any -nn -s 128 \
    'host 10.0.0.10 and port 443' \
    -w /tmp/https-check.pcap
```

`-nn` 避免 DNS 和服务名解析，`-s 128` 限制单包抓取长度，`timeout` 限制采样时长。抓包文件可能包含敏感信息和个人数据，应限制权限、保存周期和访问范围。

只根据单个 SYN、ACK 或重传包下结论很危险。要看完整流、客户端和服务端两侧、时间戳、窗口、重传和应用日志，并确认抓包本身没有造成过高开销。

---

## 八、进程、线程与文件描述符

### Q33：如何查看进程树和启动参数？

```bash
pid=${1:-1234}
ps -eo pid,ppid,pgid,sid,user,stat,lstart,etime,args --forest
pstree -aps "$pid"
tr '\0' ' ' < "/proc/$pid/cmdline"; printf '\n'
tr '\0' '\n' < "/proc/$pid/environ" | sed -n '1,20p'
```

`/proc/<pid>/cmdline` 和 `environ` 以 NUL 分隔。环境变量可能包含秘密，不要把完整输出发送到公共日志。进程树可以发现 wrapper、shell、worker、僵尸子进程和信号未传递问题。

### Q34：如何区分僵尸进程和孤儿进程？

```bash
ps -eo stat,pid,ppid,comm | awk '$1 ~ /^Z/ { print }'
```

- 僵尸进程：子进程已退出，但父进程还没有调用 `wait` 回收其退出状态；它几乎不占用运行资源，但会占用 PID 表项。
- 孤儿进程：父进程退出后被重新托管，仍可能正常运行。

僵尸多时要查父进程的子进程回收逻辑和信号处理。不能靠 `kill -9` 杀死僵尸本身，通常要修复或重启父进程。

### Q35：如何查看线程级 CPU、状态和等待点？

```bash
pid=${1:-1234}
top -H -p "$pid"
ps -L -p "$pid" -o pid,tid,psr,stat,pcpu,etime,wchan:32,comm
for task in "/proc/$pid"/task/*; do
    printf '%s ' "${task##*/}"
    awk '{ print $3, $39 }' "$task/stat"
done
```

线程级定位可以区分单线程热点、线程池饥饿、锁等待和 I/O 等待。`wchan` 只是内核等待点线索；要确认用户态锁和调用栈，需要线程 dump、语言 runtime 工具或 `perf`。

### Q36：如何检查文件描述符是否耗尽？

```bash
pid=${1:-1234}
ls "/proc/$pid/fd" | wc -l
cat "/proc/$pid/limits" | grep -i 'open files'
cat /proc/sys/fs/file-nr
lsof -p "$pid" | awk 'NR > 1 { count[$5]++ } END { for (type in count) print type, count[type] }'
```

判断：

- 进程 fd 数接近 `Max open files`：可能出现连接、日志或文件打开失败。
- 系统 `/proc/sys/fs/file-nr` 接近系统上限：可能是多个进程共同耗尽。
- 某类 fd 持续增长：检查 socket、pipe、eventpoll、日志、临时文件和连接池。

提高 `ulimit` 只能缓解容量不足，不能修复 fd 泄漏。要结合打开对象类型、生命周期和应用代码定位。

### Q37：如何查看进程内存映射和线程栈？

```bash
pid=${1:-1234}
pmap -x "$pid"
cat "/proc/$pid/smaps_rollup"
cat "/proc/$pid/status" | grep -E 'Vm|Threads|FDSize'
```

- `Rss`：驻留内存。
- `Pss`：按共享页比例分摊后的内存，更适合估算真实占用。
- `Private_*`：进程私有页。
- `Shared_*`：共享映射页。
- `Threads`：线程数量，过多会增加栈内存和调度压力。

精确解释需要结合语言运行时，例如 Python allocator、JVM heap、native malloc、共享库和 mmap 文件。单看 `VIRT` 不足以判断泄漏。

### Q38：如何使用 `strace` 定位系统调用级阻塞？

```bash
pid=${1:-1234}
sudo timeout 20 strace -ff -ttT -p "$pid" -e trace=file,network,futex,read,write
```

- `-tt`：微秒级时间戳。
- `-T`：显示每个系统调用耗时。
- `-ff`：跟踪线程/子进程并分文件输出。
- `-e trace=`：限制系统调用范围，降低开销和噪声。

观察：

- `futex` 长时间等待：可能是用户态锁或条件变量。
- `connect`/`poll`/`epoll_wait`：网络连接或事件循环等待。
- `read`/`write`：文件或网络 I/O 阻塞。
- `openat` 大量失败：路径、权限、文件描述符或配置问题。

`strace` 会带来额外开销，线上只做短时、窄范围采样，不要把全部系统调用长期打开。

---

## 九、应用级深度定位

### Q39：如何从进程热点继续定位到代码？

建议按语言选择工具：

- C/C++/Rust/native：`perf record -g`、符号表、火焰图、core dump。
- Java：线程 dump、JFR、`jstack`、async-profiler、GC 日志。
- Go：pprof、goroutine dump、mutex/block profile。
- Python：`py-spy`、`cProfile`、`scalene`，并区分 Python 时间和 native 库时间。
- Node.js：CPU profile、event loop delay、heap snapshot。

共同流程是：先确认系统层 PID 和线程，采样调用栈，再把热点映射到业务请求、输入规模和发布版本。函数占比高不等于一定需要优化；要比较总耗时、调用次数、每次成本和业务收益。

### Q40：如何判断是锁竞争还是 I/O 等待？

锁竞争常见特征：

- CPU 可能不高，但上下文切换和线程等待多。
- `strace` 可能看到 `futex`。
- 线程 dump 显示大量线程等待同一锁。
- 应用线程池活跃但完成任务少。

I/O 等待常见特征：

- `vmstat` 的 `b`/`wa` 增高。
- `iostat` 的 `await`、队列或设备利用率升高。
- `strace` 的 `read`、`write`、`poll`、`connect` 耗时长。
- 进程线程处于 `D` 或等待网络响应。

两者可能同时存在，例如锁保护的代码中执行了数据库请求。应检查临界区是否包含外部 I/O，并将锁竞争和下游延迟分开测量。

### Q41：如何判断 GC、内存分配还是业务逻辑导致 CPU 高？

证据组合：

1. 进程 CPU 和线程 CPU：是否集中在 GC/allocator 线程。
2. 堆使用和 GC 暂停：是否与延迟峰值同步。
3. `perf`/语言 profiler：热点是否在分配、复制、扫描或回收。
4. 请求速率和对象创建量：是否由流量、序列化或缓存 miss 增长。
5. 修复后对比：减少对象、批量化或调整缓存后，CPU 和延迟是否同时改善。

不要只看到内存高就判断“GC 导致 CPU 高”，也不要只调大堆。先建立分配速率、存活对象和暂停时间证据。

### Q42：如何定位连接池耗尽？

现象通常是请求延迟升高、线程等待连接、连接数接近上限、下游数据库或 HTTP 服务并未达到处理上限。排查：

```bash
ss -tanp
pid=${1:-1234}
pidstat -t -p "$pid" 1 5
lsof -p "$pid" | grep -E 'TCP|unix' | head
```

应用层要查看：

- 活跃连接、空闲连接、等待连接线程数。
- 获取连接耗时、连接使用时长和泄漏检测。
- 下游响应时间、连接建立时间和失败率。
- 超时、重试和连接池大小是否匹配。

盲目增大连接池可能把压力转移到数据库、服务端端口或网络设备。容量应由并发、服务端处理能力、连接成本和延迟目标共同决定。

### Q43：如何定位日志导致的性能问题？

检查：

```bash
pid=${1:-1234}
pidstat -d -p "$pid" 1 5
iostat -xz 1 5
lsof -p "$pid" | grep -E '\.(log|out)' | head
journalctl -u my-service --since '10 min ago' --no-pager | wc -l
```

常见问题：同步写日志阻塞业务线程、错误重试产生日志风暴、日志格式化和堆栈生成消耗 CPU、磁盘或网络日志代理背压、日志轮转不当造成旧文件持续占用。

修复应包括日志级别、采样、异步队列上限、背压策略、字段脱敏、轮转和集中式采集，而不是简单删除日志。

### Q44：如何判断是应用问题还是下游依赖问题？

把一次请求拆成时间段：DNS、连接、TLS、排队、应用计算、数据库、远程 API、序列化和响应发送。使用 trace ID、服务端 access log、`curl -w`、连接池指标和数据库慢查询交叉验证。

- 应用 CPU/队列高，依赖耗时正常：优先查本服务。
- 本服务线程都在等待下游，依赖 P99 同步升高：依赖可能是根因。
- 本服务重试增加并放大下游：需要先止损，例如限流、熔断和降级。
- 客户端超时但服务端仍完成请求：要检查超时预算和取消传播，避免重复请求。

### Q45：性能修复后如何验证？

必须使用与原问题可比的环境和流量：

- 同一版本或明确记录版本差异。
- 同等数据规模、并发、请求比例和网络条件。
- 比较 P50/P95/P99、吞吐、错误率、CPU、内存、I/O 和下游压力。
- 观察足够长时间，覆盖缓存预热、GC、批处理和流量峰值。
- 验证没有把问题转移到数据库、磁盘、网络或其他服务。

修复应配套回滚开关、监控告警和复盘记录。一次“命令输出变好”不足以证明用户体验已经改善。

---

## 十、容器与 cgroup 性能

### Q46：如何确认进程是否受到 cgroup 限制？

```bash
cat /proc/1/cgroup
systemd-cgls 2>/dev/null || true
systemctl status my-service
```

cgroup v2 常见文件：

```bash
find /sys/fs/cgroup -maxdepth 2 -name 'cpu.max' -o -name 'memory.max' -o -name 'memory.current'
cat /sys/fs/cgroup/cpu.max 2>/dev/null
cat /sys/fs/cgroup/memory.max 2>/dev/null
cat /sys/fs/cgroup/memory.current 2>/dev/null
```

`memory.max` 为 `max` 表示没有该层硬限制；`memory.current` 表示当前使用。`cpu.max` 通常形如 `quota period`，例如 `200000 100000` 约等于最多使用 2 个 CPU 的配额。

路径会因 systemd、Docker、Kubernetes 和 cgroup v1/v2 层级不同而变化，不能只硬编码一个路径。

### Q47：如何判断 CPU throttling？

```bash
cat /sys/fs/cgroup/cpu.stat 2>/dev/null
```

cgroup v2 常见字段：

- `usage_usec`：累计 CPU 使用时间。
- `user_usec`/`system_usec`：用户态和系统态使用。
- `nr_periods`：调度周期数。
- `nr_throttled`：被限制的周期数。
- `throttled_usec`：被限制等待的时间。

如果 `nr_throttled` 和 `throttled_usec` 在业务高延迟时明显增加，说明容器 CPU quota 可能成为瓶颈，即使宿主机仍有 idle CPU。应调整 requests/limits、减少 CPU 峰值、优化线程模型或迁移到合适的资源池。

### Q48：如何查看容器内存压力和 OOM？

```bash
cat /sys/fs/cgroup/memory.events 2>/dev/null
cat /sys/fs/cgroup/memory.pressure 2>/dev/null
cat /sys/fs/cgroup/memory.current 2>/dev/null
cat /sys/fs/cgroup/memory.max 2>/dev/null
```

`memory.events` 中的 `high`、`max`、`oom`、`oom_kill` 可帮助判断是否达到 cgroup 层限制。容器可能在主机还有大量空闲内存时被自己的 memory limit 杀死。

同时查看 Kubernetes events、容器重启原因、工作集、page cache、临时文件和应用堆。不要只看 `docker stats` 的瞬时值。

### Q49：什么是 PSI，如何解读？

```bash
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io
```

Pressure Stall Information 描述任务因资源不足而无法推进的时间比例：

- `some`：至少部分任务受到资源压力。
- `full`：所有相关任务都在等待，通常表示更严重的停顿。
- `avg10`、`avg60`、`avg300`：不同时间窗口的平均压力百分比。
- `total`：累计微秒数。

PSI 很适合发现 CPU、内存和 I/O 的争用，即使传统使用率看起来不满。容器和 cgroup 也可能有对应的 pressure 文件。要将 PSI 与 P99、队列和资源指标对齐后再设告警阈值。

---

## 十一、常见性能症状到根因

### 1. API 延迟突然升高

```text
先看：错误率、QPS、P99、发布时间
  -> uptime / vmstat / top
  -> pidstat / ss / iostat
  -> 线程池、连接池、数据库慢查询、下游 trace
  -> 限流、熔断、回滚或扩容
```

重点区分：流量增加、应用 CPU、GC、磁盘等待、连接池耗尽、网络重传、依赖变慢和重试放大。

### 2. CPU 使用率接近 100%

检查：

```bash
mpstat -P ALL 1 5
pidstat -u -t 1 5
pid=${1:-1234}
perf top -p "$pid"
```

如果单线程满核，优先查热点、锁和事件循环；如果所有核都忙，查业务计算、批任务、GC、压缩、加密和错误风暴；如果 `sy/si` 高，查系统调用、网络包和内核路径；如果 `st` 高，查虚拟化争抢。

### 3. Load Average 持续高

```bash
uptime
vmstat 1 5
ps -eo state,pid,ppid,wchan:32,comm | awk '$1 ~ /D/ { print }'
iostat -xz 1 5
```

- `r` 高：CPU 排队。
- `b` 高：不可中断 I/O 等待。
- `wa` 高：块设备等待。
- CPU idle 高且 `D` 多：磁盘、NFS 或存储链路。
- load 与业务无关且短时出现：可能是备份、压缩、扫描或定时任务。

### 4. 内存持续上涨

```bash
free -h
ps -eo pid,comm,rss,%mem --sort=-rss | head
vmstat 1 5
cat /proc/meminfo
```

建立 RSS/PSS 时间序列，检查缓存上限、请求积压、线程数、连接数、临时对象和运行时 heap。若出现 `si/so`、major fault 或 OOM，优先止损并保留现场，不要先清 cache。

### 5. 磁盘 I/O 变慢

```bash
iostat -xz 1 5
pidstat -d 1 5
lsof +L1
df -hT
df -i
```

区分设备饱和、空间/inode 满、日志风暴、删除但仍打开、NFS、同步写和应用锁。`%util=100%` 是线索，不是完整结论；要同时看 `await`、队列、吞吐和业务延迟。

### 6. 网络请求超时

```bash
ss -s
ss -tan state syn-sent
curl -sS -o /dev/null -w 'dns=%{time_namelookup} connect=%{time_connect} ttfb=%{time_starttransfer} total=%{time_total}\n' --max-time 10 "$url"
nstat -az 2>/dev/null | grep -i retrans
```

依次定位 DNS、连接建立、TLS、服务端排队、响应传输、连接池和重试。网络超时不一定是网络设备问题，也可能是服务端没有及时 accept、线程池满、连接池耗尽或依赖链变慢。

### 7. 容器被杀或变慢但宿主机正常

```bash
cat /sys/fs/cgroup/cpu.stat 2>/dev/null
cat /sys/fs/cgroup/memory.events 2>/dev/null
cat /sys/fs/cgroup/cpu.pressure 2>/dev/null
cat /sys/fs/cgroup/memory.pressure 2>/dev/null
```

优先检查 CPU throttling、memory limit、cgroup OOM、memory pressure、CPU requests/limits、节点争用和容器重启事件。宿主机全局 CPU idle 不代表该容器没有 quota 限制。

---

## 十二、一键采样脚本示例

下面脚本只做低侵入、只读采样，适合故障初期收集现场。它不替代专业监控，也不应在没有权限和变更审批的生产环境中随意扩展为破坏性操作。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

output_dir=${1:-"/tmp/perf-snapshot-$(date +%Y%m%d-%H%M%S)"}
mkdir -p -- "$output_dir"

run_capture() {
    local name=$1
    shift
    {
        printf '===== command: %q'
        printf ' %q' "$@"
        printf '\n===== time: %s =====\n' "$(date -Is)"
        "$@"
    } >"$output_dir/$name.txt" 2>&1 || true
}

run_capture environment uname -a
run_capture uptime uptime
run_capture memory free -h
run_capture vmstat vmstat 1 5
run_capture cpu mpstat -P ALL 1 5
run_capture disk iostat -xz 1 5
run_capture processes ps -eo pid,ppid,stat,pcpu,pmem,rss,etime,comm,args --sort=-pcpu
run_capture sockets ss -s
run_capture mounts findmnt
run_capture limits cat /proc/limits

if [[ -r /proc/pressure/cpu ]]; then
    run_capture pressure_cpu cat /proc/pressure/cpu
    run_capture pressure_memory cat /proc/pressure/memory
    run_capture pressure_io cat /proc/pressure/io
fi

printf 'snapshot saved to %s\n' "$output_dir"
```

### 脚本使用注意

- `run_capture` 中的 `|| true` 是有意的：某些命令可能不存在或没有权限，但不应阻断其他采样；同时必须从输出中识别缺失命令，而不是把采样不完整当作正常。
- 采样目录可能包含主机名、进程参数、IP、环境信息和路径，必须限制权限并按敏感数据处理。
- `ps` 命令参数和 `mpstat`/`iostat` 是否安装取决于发行版；脚本可扩展为命令存在性检查和版本记录。
- 采样要有时间限制，避免诊断脚本自身成为资源负担。

---

## 十三、面试高频问题

### Q50：为什么不能只看 CPU 使用率判断性能？

因为 CPU 使用率只描述 CPU 时间如何分配，不能解释 I/O 等待、锁等待、网络延迟、内存压力、cgroup throttling、磁盘队列或下游服务。CPU 低可能是线程都在等待，CPU 高也可能是有效吞吐正常。

成熟排障至少组合：业务延迟/错误率、load、`vmstat`、进程/线程、磁盘 I/O、网络连接、应用线程池和依赖指标。

### Q51：`free` 很少是否代表内存不足？

不一定。Linux 会使用空闲内存作为 page cache，因此应优先看 `available`、Swap 活动、major fault、进程 RSS/PSS 和 OOM 事件。只有当可用内存下降、持续换页、应用延迟上升或发生 OOM 时，才能确认内存压力正在影响系统。

### Q52：Load Average 和 CPU 使用率有什么区别？

CPU 使用率表示 CPU 时间花在哪里；Load Average 表示一段时间内处于可运行或不可中断等待的任务数量平均值。Load 高可能是 CPU 排队，也可能是磁盘/NFS 等 I/O 阻塞。必须结合 CPU 核数、`r/b`、`wa`、设备延迟和进程状态。

### Q53：`await` 高但 `%util` 不高，可能是什么原因？

可能是单个请求延迟高但并发不够，设备本身没有持续忙满；也可能是虚拟存储后端、网络存储、队列、文件系统或同步写造成延迟。应继续看 `aqu-sz`、读写类型、设备层级、业务 I/O 模式和云盘指标，不能仅凭 `%util` 下结论。

### Q54：为什么 `kill -9` 不是首选恢复手段？

`SIGKILL` 无法被进程捕获或清理，可能造成临时文件、锁、事务、连接和子进程状态不完整。优先 `SIGTERM`，等待优雅退出，必要时再使用 `SIGKILL`。对 D 状态进程，`kill -9` 也通常要等内核 I/O 返回后才能真正退出。

### Q55：为什么生产脚本不应该使用 `rm -rf "$DIR"` 前就假设变量一定有值？

如果变量为空、拼接错误或指向错误路径，可能造成灾难性删除。应使用 `set -u`、参数校验、路径白名单、dry-run、确认根目录不是 `/`、不是空字符串，并记录审计。破坏性操作必须有可恢复方案。

### Q56：如何解释“连接数不高但请求仍超时”？

连接数只是数量，不表示连接是否能及时处理。可能是服务端线程池满、连接池等待、单连接上的请求排队、TLS/CPU、数据库锁、下游超时、接收窗口、代理超时或应用事件循环阻塞。应将连接建立、排队、处理和响应传输拆开测量。

### Q57：如何定位线上偶发性能问题？

建立持续采样和关联 ID，而不是依赖故障发生后手工执行一次命令。使用低开销时间序列指标记录 P99、资源、队列、连接池、GC、I/O、网络和 cgroup pressure；发生时再短时启用 `pidstat`、`perf`、`strace` 或抓包，并控制采样范围。

### Q58：性能优化后怎样证明没有把问题转移？

对比优化前后的业务指标和所有相关资源：吞吐、P50/P95/P99、错误率、CPU、内存、Swap、磁盘 await、网络重传、线程池、连接池、数据库 QPS 和下游延迟。使用相近流量和数据规模，覆盖预热、峰值和长时间运行，并保留回滚开关。

---

## 十四、排障自检清单

### 环境与范围

- [ ] 已确认主机、容器、内核、发行版、CPU 架构和 cgroup 版本。
- [ ] 已记录故障开始时间、结束时间、影响服务、版本和流量变化。
- [ ] 已区分单机、单容器、单进程、单接口还是全局问题。
- [ ] 已确认诊断命令权限、采样时长和数据敏感性。

### CPU 与进程

- [ ] 已比较 load 和逻辑 CPU 核数。
- [ ] 已检查 `vmstat` 的 `r`、`b`、`wa`、`cs` 和 `st`。
- [ ] 已用 `pidstat -t` 或 `top -H` 定位到线程。
- [ ] 已区分用户态、内核态、I/O wait、steal、GC 和锁等待。

### 内存与磁盘

- [ ] 已看 `MemAvailable`、Swap in/out、major fault 和 OOM 日志。
- [ ] 已确认进程 RSS/PSS、线程数、缓存上限和 cgroup memory limit。
- [ ] 已同时检查 `df -h`、`df -i`、`du` 和 deleted-open 文件。
- [ ] 已用 `iostat -xz` 看 await、队列、读写类型和 util。

### 网络与依赖

- [ ] 已查看监听端口、连接状态、连接数和文件描述符。
- [ ] 已拆解 DNS、TCP、TLS、TTFB 和响应传输时间。
- [ ] 已检查丢包、错误、重传、SYN backlog 和代理超时。
- [ ] 已通过 trace、日志、连接池和下游指标确认依赖链路。

### 修复与复盘

- [ ] 修复有明确假设、验证指标和回滚方案。
- [ ] 没有用清 cache、无限重试、无限扩容或 `kill -9` 掩盖根因。
- [ ] 已验证修复没有把压力转移到其他资源或服务。
- [ ] 已补充告警、容量上限、自动化检查和故障复盘。

> 高质量 Linux 性能排障不是记住最多命令，而是能把用户症状、系统指标、进程行为、内核证据和业务结果串成一条可验证的因果链。
