# Shell 与运维现场 Coding 高频题库

> 面向 Shell/Bash 中高级工程师、Linux 运维工程师、DevOps/SRE 和平台架构师面试。内容覆盖 Bash 基础、Linux 运维知识、日志与进程排障、Shell 现场 Coding、发布与自动化脚本设计。
>
> 文中的脚本默认以 Bash 为主，生产示例优先使用 Bash 4+ 和 GNU/Linux 工具链。面试时应主动说明 macOS 默认 Bash 版本较旧、BSD 工具参数与 GNU 工具存在差异；需要跨平台时，应使用 POSIX `sh` 子集或明确适配层。

## 目录

1. [现场 Coding 的答题方法](#一现场-coding-的答题方法)
2. [Shell 基础知识](#二shell-基础知识)
3. [Linux 运维基础](#三linux-运维基础)
4. [Shell 现场 Coding 题](#四shell-现场-coding-题)
5. [运维现场 Coding 题](#五运维现场-coding-题)
6. [排障追问与测试清单](#六排障追问与测试清单)
7. [面试前自检](#七面试前自检)

---

## 一、现场 Coding 的答题方法

### 1. 先确认运行环境

写脚本前先问清楚：

- 目标 Shell 是 Bash、POSIX `sh`、Zsh 还是 PowerShell？
- 目标系统是 Linux、macOS 还是容器中的精简发行版？
- 是否可以使用 `jq`、`curl`、`flock`、`systemctl`、`awk`、`sed` 等外部命令？
- 输入来自参数、标准输入、文件、环境变量还是 API？
- 失败时是否允许部分成功？是否需要重试、回滚、告警或幂等？
- 脚本是否可能被并发执行？是否需要锁、超时和信号处理？

### 2. 推荐答题顺序

1. 复述输入、输出和失败条件。
2. 先写最小可运行版本，再补充参数校验和错误处理。
3. 为变量加双引号，避免路径空格、通配符和空值造成意外展开。
4. 说明退出码、日志、临时文件、锁和清理策略。
5. 用正常、空输入、异常输入、并发和中断场景验证。
6. 说明 Linux/GNU 与 macOS/BSD 的兼容差异。
7. 讨论生产环境的可观测性、权限、回滚和自动化测试。

### 3. Shell 高分实现的共同特征

- 使用 `#!/usr/bin/env bash` 或明确的 POSIX shebang，不混用 Bash 专属语法和 `sh`。
- 对外部输入使用双引号、白名单校验和安全的参数传递。
- 关键脚本使用 `set -Eeuo pipefail`，同时理解它的例外语义，而不是机械粘贴。
- 使用 `trap` 清理临时文件、释放锁和响应 `INT`/`TERM`。
- 用退出码区分成功、业务失败、参数错误和依赖不可用。
- 重要脚本可被重复执行，或者明确说明不可重复执行的原因。

---

## 二、Shell 基础知识

### Q1：Shebang 是什么？`#!/bin/bash` 和 `#!/usr/bin/env bash` 如何选择？

Shebang 是脚本第一行，用来指定解释器。`#!/bin/bash` 依赖 Bash 位于固定路径，行为更确定；`#!/usr/bin/env bash` 会从 `PATH` 中查找 Bash，适合路径不同的环境，但可能使用到非预期版本。

```bash
#!/usr/bin/env bash

printf 'running with: %s\n' "$BASH_VERSION"
```

生产脚本应明确最低 Bash 版本和依赖，不应只因为 `env` 更“便携”就忽略版本差异。若脚本只使用 POSIX 语法，应使用 `#!/bin/sh` 并用 ShellCheck 的对应模式检查。

### Q2：`set -euo pipefail` 分别做什么？有什么陷阱？

```bash
set -Eeuo pipefail
```

- `-e`：命令失败时退出，但在 `if` 条件、`while` 条件、`&&`/`||` 部分和某些管道上下文中有例外。
- `-u`：读取未设置变量时失败，通常使用 `${value:-default}` 或先初始化变量。
- `pipefail`：管道返回非零状态，只要其中一个命令失败，而不是只看最后一个命令。
- `-E`：让 `ERR` trap 在函数、命令替换和子 Shell 中继承，具体行为仍要结合上下文测试。

严格模式不能代替错误设计。对于允许失败的命令要显式处理：

```bash
if ! grep -q -- "$pattern" "$file"; then
    printf 'pattern not found\n' >&2
fi
```

不要把所有命令都写成 `command || true`，否则会掩盖真正故障；应说明哪些失败是预期的，并记录原因。

### Q3：Shell 中变量、环境变量和 `export` 有什么区别？

Shell 变量只存在于当前 Shell 进程；`export` 把变量放入环境，使后续启动的子进程可以读取。子进程修改环境不会反向修改父 Shell。

```bash
name='service'
export APP_ENV='production'

printf '%s\n' "$name"
env | grep '^APP_ENV='
```

变量赋值两边不能有空格。敏感信息不应通过命令行参数传递，因为可能出现在进程列表中；也不要默认把秘密写入环境，环境变量可能被诊断工具、子进程或错误日志暴露。

### Q4：为什么 Shell 变量通常需要双引号？

不加引号时，变量展开后可能继续发生分词和 pathname expansion，导致一个参数变成多个参数，或者 `*` 被展开成当前目录文件名。

```bash
file='report final.txt'
rm -- "$file"

for file in "$directory"/*; do
    printf '%s\n' "$file"
done
```

`"$@"` 会保留每一个位置参数，`$*` 在双引号中通常会把参数合并成一个字符串。处理用户输入、路径、日志字段和 API 返回值时，默认使用双引号；只有明确需要分词或通配时才不加引号。

### Q5：`$?`、`$$`、`$!`、`$#`、`$@` 分别是什么？

| 变量 | 含义 |
| --- | --- |
| `$?` | 最近一个命令的退出状态 |
| `$$` | 当前 Shell 进程的 PID |
| `$!` | 最近一个后台任务的 PID |
| `$#` | 位置参数数量 |
| `$@` | 位置参数集合，双引号中保留参数边界 |
| `$0` | 脚本或函数调用名，具体显示取决于上下文 |

```bash
printf 'script=%s args=%s pid=%s\n' "$0" "$#" "$$"
for argument in "$@"; do
    printf 'argument=%s\n' "$argument"
done
```

不要在执行其他命令后再读取 `$?`，否则会被覆盖。需要多个状态时，立即保存：`status=$?`。

### Q6：命令替换、进程替换和算术展开有什么区别？

- 命令替换：`$(command)`，把命令标准输出作为字符串插入。
- 进程替换：`<(command)` 或 `>(command)`，把命令输出或输入暴露为类似文件的路径。
- 算术展开：`$((expression))`，执行整数运算。

```bash
current_date=$(date +%F)
while IFS= read -r line; do
    printf '%s\n' "$line"
done < <(generate_report)
count=$((count + 1))
```

命令替换会去掉末尾换行，不能用它精确保存任意二进制数据。进程替换依赖 Shell 能力，不是 POSIX `sh` 通用语法。

### Q7：`[ ]`、`[[ ]]` 和 `(( ))` 如何选择？

- `[ ... ]` 通常是 `test` 命令，兼容性好，但参数和转义更容易出错。
- `[[ ... ]]` 是 Bash/Ksh 条件语法，支持更安全的字符串比较、模式匹配和正则匹配。
- `(( ... ))` 用于算术条件和计算。

```bash
if [[ -n ${name:-} && $name == service-* ]]; then
    printf 'valid service name\n'
fi

if (( retry_count >= 3 )); then
    printf 'retry limit reached\n' >&2
fi
```

使用 `[[ ]]` 时通常不需要给变量额外加引号来防止分词，但为了可读性仍应保持清晰。不要在声明为 `#!/bin/sh` 的脚本中使用 `[[ ]]` 或 Bash 数组。

### Q8：字符串、整数和正则判断如何写？

```bash
[[ -z ${value:-} ]]       # 空字符串
[[ -n ${value:-} ]]       # 非空字符串
[[ "$left" == "$right" ]] # 字符串相等
[[ $number -ge 10 ]]      # test 风格整数比较
(( number >= 10 ))        # Bash 算术比较
[[ $value =~ ^[0-9]+$ ]]  # Bash 正则匹配
```

正则匹配中的右侧表达式通常不需要加引号，否则可能失去正则语义。对用户输入做校验时，应使用白名单，例如只允许 `[a-zA-Z0-9._-]`，不要只依赖黑名单替换危险字符。

### Q9：数组和关联数组如何使用？

```bash
services=(api worker scheduler)
for service in "${services[@]}"; do
    printf '%s\n' "$service"
done

if (( ${#services[@]} == 0 )); then
    printf 'no services\n'
fi
```

Bash 4+ 支持关联数组：

```bash
declare -A ports=()
ports[api]=8080
ports[worker]=8081
printf '%s\n' "${ports[api]}"
```

macOS 系统自带 Bash 3.2，不支持关联数组；面试时应主动说明版本差异。`"${array[@]}"` 保留元素边界，`"${array[*]}"` 在双引号中通常合并成一个字符串。

### Q10：函数如何返回结果？`return` 和命令输出有什么区别？

Shell 函数的 `return` 只能返回 0 到 255 的退出状态，业务数据通常通过标准输出返回，再由命令替换捕获。

```bash
get_status() {
    local service_name=$1
    if systemctl is-active --quiet "$service_name"; then
        printf 'active\n'
        return 0
    fi
    printf 'inactive\n'
    return 1
}

if status=$(get_status nginx); then
    printf 'status=%s\n' "$status"
fi
```

函数内变量应使用 `local`，避免污染调用方。不要把日志和返回数据都写到标准输出，否则调用方捕获结果时会混入日志；日志写标准错误更清晰。

### Q11：重定向和文件描述符如何理解？

```bash
command >output.log 2>error.log
command >>output.log 2>&1
command &>combined.log
command </tmp/input.txt
```

`0` 是标准输入，`1` 是标准输出，`2` 是标准错误。重定向从左到右解释，因此 `command >file 2>&1` 与 `command 2>&1 >file` 的效果不同。

可以创建额外文件描述符：

```bash
exec 3>audit.log
printf 'deployment started\n' >&3
exec 3>&-
```

生产脚本应区分用户输出、诊断日志和审计信息，并控制敏感信息的权限。

### Q12：管道、子 Shell 和 `pipefail` 有什么关系？

管道把前一个命令的标准输出连接到后一个命令的标准输入。传统 Bash 中，管道中的每个命令通常运行在独立进程或子 Shell 中，因此在管道循环里修改的变量可能不会影响父 Shell。

```bash
count=0
while IFS= read -r line; do
    ((count++))
done < input.txt
printf 'count=%s\n' "$count"
```

如果写成 `cat input.txt | while ...`，在一些 Shell 中 `count` 的修改发生在子 Shell。需要检查每段管道的结果时使用 `${PIPESTATUS[@]}`，并在关键脚本中启用 `pipefail`。

### Q13：如何安全读取文件和处理带空格的文件名？

```bash
while IFS= read -r line; do
    printf '<%s>\n' "$line"
done < input.txt
```

读取文件列表时使用 NUL 分隔，避免换行、空格和通配符破坏路径：

```bash
while IFS= read -r -d '' file; do
    printf '%s\n' "$file"
done < <(find "$root" -type f -print0)
```

不要使用 `for file in $(find ...)`，它会发生命令替换、分词和路径破坏。不要对不可信输入使用 `eval`。

### Q14：`trap` 常见用途是什么？

`trap` 可以响应信号、脚本退出和错误，常用于删除临时目录、释放锁、记录错误和优雅停止子进程。

```bash
temp_dir=$(mktemp -d)
cleanup() {
    rm -rf -- "$temp_dir"
}
trap cleanup EXIT
trap 'printf "interrupted\n" >&2; exit 130' INT TERM
```

清理函数要可重复执行，不能因为临时文件已经不存在而导致退出流程失败。复杂脚本要避免在 `EXIT` trap 中覆盖原始退出码：

```bash
status=0
some_command || status=$?
cleanup
exit "$status"
```

### Q15：如何正确处理后台进程和信号？

```bash
long_running_command &
child_pid=$!

if ! wait "$child_pid"; then
    printf 'child failed\n' >&2
    exit 1
fi
```

`kill` 发送信号，`kill -0` 只检查进程是否存在和当前用户是否有权限。收到 `TERM` 时应停止接受新任务、通知子进程、等待有限时间，再必要时发送 `KILL`。不能把 `kill -9` 作为默认优雅关闭方式。

### Q16：`find`、`xargs` 和 `-print0` 如何安全组合？

```bash
find "$root" -type f -name '*.log' -print0 \
    | xargs -0 -r gzip --
```

`-print0` 与 `xargs -0` 可以安全处理空格和换行。`xargs` 的 `-r` 在 GNU 版本常见，BSD 版本可能不同；跨平台脚本可以使用 `while IFS= read -r -d ''`。

删除前先使用 `-print` 或 dry-run 检查路径。危险的 `find / -delete`、未引用变量和模糊的 glob 都可能造成大范围误删。

### Q17：`grep`、`sed` 和 `awk` 如何分工？

| 工具 | 适合场景 |
| --- | --- |
| `grep` | 过滤包含或不包含模式的行 |
| `sed` | 流式替换、删除和简单转换 |
| `awk` | 按字段处理、聚合、格式化和条件计算 |
| `cut` | 简单字段截取 |
| `sort`/`uniq` | 排序、去重和频次统计 |

```bash
grep -E 'ERROR|WARN' application.log
sed -n '1,20p' application.log
awk '$9 >= 500 { count++ } END { print count + 0 }' access.log
```

字段分隔符、日志格式和编码要先确认。复杂 JSON 不要用正则硬拆，优先使用 `jq`。

### Q18：如何理解退出码和 `PIPESTATUS`？

Unix 命令约定退出码 `0` 表示成功，非零表示失败；具体非零值的含义由命令定义。Shell 中 `if command; then` 是按退出码判断，而不是看标准输出。

```bash
set -o pipefail
producer | transformer | consumer
statuses=("${PIPESTATUS[@]}")
printf 'producer=%s transformer=%s consumer=%s\n' "${statuses[@]}"
```

读取 `PIPESTATUS` 必须紧跟在管道后，否则会被其他命令覆盖。脚本给调用方返回状态时，应保留有意义的退出码，避免所有错误都返回 1 而丢失分类。

### Q19：Shell 脚本如何避免命令注入？

- 所有变量默认双引号。
- 使用数组传递命令和参数，不拼接待执行字符串。
- 不使用 `eval` 执行用户输入。
- 对服务名、环境名、文件名和版本号做白名单校验。
- 使用 `--` 结束选项，避免以 `-` 开头的文件名被当作参数。
- 外部命令调用时明确工作目录、权限和环境变量。

```bash
args=(--connect-timeout 5 --fail --silent)
if [[ ${verbose:-false} == true ]]; then
    args+=(--show-error)
fi
curl "${args[@]}" -- "$url"
```

安全不仅是转义问题，还包括最小权限、秘密管理、日志脱敏、临时文件权限和供应链校验。

### Q20：ShellCheck、格式化和脚本测试有什么作用？

ShellCheck 可以发现未引用变量、错误的测试语法、数组误用、管道循环、未检查退出码和可移植性问题。`shfmt` 可以统一格式，但格式化不能证明逻辑正确。

成熟脚本至少应有：

- ShellCheck 静态检查。
- `bash -n script.sh` 语法检查。
- 正常、异常、空输入和中断测试。
- 临时目录或容器中的破坏性操作测试。
- 对外部命令和时间的可替换设计。

---

## 三、Linux 运维基础

### Q21：Linux 进程、线程和文件描述符如何排查？

常用命令：

```bash
pid=${1:-1234}
ps -ef
ps -eLo pid,tid,psr,pcpu,pmem,comm --sort=-pcpu | head
ls -l "/proc/$pid/fd"
lsof -p "$pid"
```

进程有 PID、父 PID、用户、状态、打开的文件描述符和资源限制。线程共享进程地址空间和大部分资源，但有独立的栈和调度上下文。文件描述符耗尽会导致网络连接、日志文件或子进程创建失败，需要同时检查进程级和系统级限制。

### Q22：CPU 高、Load Average 高和进程卡顿如何区分？

排查顺序可以是：

```bash
top
uptime
vmstat 1
pidstat -p <pid> 1
mpstat -P ALL 1
```

- CPU 高：看用户态、内核态、iowait 和具体线程。
- Load Average 高：表示可运行或不可中断睡眠任务较多，不等同于 CPU 使用率高。
- iowait 高：关注磁盘、网络存储和阻塞 I/O。
- 进程卡顿：检查锁、I/O、连接池、文件描述符、GC 或下游依赖。

不要只看到 load 高就直接扩容 CPU；要结合 CPU 核数、run queue、iowait、上下文切换和业务延迟判断。

### Q23：内存不足或 OOM 如何排查？

```bash
free -h
vmstat 1
ps -eo pid,ppid,comm,%mem,rss,vsz --sort=-rss | head
journalctl -k | grep -i -E 'oom|killed process'
```

区分进程 RSS 增长、页缓存、Swap、容器 cgroup 限制和内核 OOM Killer。容器中还要检查 `memory.current`、`memory.max` 和应用自身堆配置。不要只执行 `drop_caches`，它不能修复应用内存泄漏，且可能损害性能。

### Q24：磁盘满和 inode 满有什么区别？

```bash
df -h
df -i
du -xdev -h --max-depth=1 /var
```

磁盘空间满通常是文件内容占用过多，inode 满则可能是大量小文件，即使还有可用字节也无法创建新文件。排查日志、临时文件、容器层、删除但仍被进程打开的文件：

```bash
lsof +L1
```

处理时要先确认文件所有者和保留策略，再清理、压缩、轮转或扩容，不能直接删除正在写入的关键文件。

### Q25：网络连接和端口如何排查？

```bash
ss -lntp
ss -s
curl -v --connect-timeout 3 http://127.0.0.1:8080/health
nc -vz -w 3 host.example.com 443
```

区分 DNS、TCP 连接、TLS 握手、HTTP 状态、应用处理和响应读取超时。检查监听地址是 `127.0.0.1` 还是 `0.0.0.0`，防火墙、安全组、代理、连接池、TIME_WAIT 和文件描述符是否满足容量要求。

### Q26：systemd 服务排障步骤是什么？

```bash
systemctl status my-service
journalctl -u my-service -n 200 --no-pager
systemctl show my-service -p User -p Environment -p Restart
systemctl cat my-service
```

重点检查 ExecStart 路径、工作目录、用户权限、环境变量、依赖、Restart 策略、资源限制和退出码。服务能在交互式 Shell 中运行，不代表 systemd 环境下也能运行，因为 PATH、cwd、权限和环境不同。

### Q27：文件权限、ACL 和 umask 如何理解？

Linux 权限按 owner、group、other 分为读、写、执行。目录的执行权限表示能否进入和访问其中的路径项，不只是“运行目录”。

```bash
id
stat file
namei -l /path/to/file
getfacl file
umask
```

`umask` 从默认权限中屏蔽权限位。运维脚本应使用最小权限、专用用户和明确的临时目录权限；不要用 `chmod -R 777` 作为排障默认方案。

### Q28：日志轮转需要考虑什么？

日志轮转应考虑大小或时间触发、压缩、保留周期、权限、应用是否支持 reopen、磁盘空间、并发写入和审计要求。直接 `mv` 正在写入的日志后，进程可能仍持有旧文件描述符，导致新文件没有数据。

常见方案是 `logrotate` 配合 `copytruncate` 或发送 `USR1`/重载信号让应用重新打开日志。高吞吐服务优先使用应用或日志代理支持的 reopen 机制，避免 `copytruncate` 在复制期间丢失日志。

### Q29：Cron 和 systemd timer 如何选择？

Cron 简单、普遍，适合轻量定时任务；systemd timer 可以表达依赖、随机延迟、持久化补跑、资源限制和服务生命周期，更适合服务器生产任务。

定时任务必须处理：

- 使用绝对路径和明确的 `PATH`。
- 记录 stdout/stderr 和退出码。
- 加锁防止上一次未结束时重入。
- 设置超时和失败告警。
- 处理时区、夏令时和主机时间同步。

### Q30：容器中的 Shell 运维与宿主机有什么不同？

容器通常有独立的 PID、网络、挂载和 cgroup 视图，PID 1 还承担信号转发和孤儿进程回收职责。容器内看到的 CPU、内存和进程不一定代表宿主机全局情况。

运维脚本应避免假设 systemd 一定存在，优先通过容器编排平台提供健康检查、日志、资源限制和滚动发布。不要在容器中用 `kill -9`、删除日志或手工改配置来代替正确的发布和恢复流程。

---

## 四、Shell 现场 Coding 题

### Q31：写一个安全的目录备份脚本

**题目：** 接收源目录和备份目录，生成带时间戳的压缩包。目录不存在、参数不足或归档失败时返回非零退出码。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

if (( $# != 2 )); then
    printf 'usage: %s SOURCE_DIR BACKUP_DIR\n' "$0" >&2
    exit 2
fi

source_dir=$1
backup_dir=$2

if [[ ! -d $source_dir ]]; then
    printf 'source directory does not exist: %s\n' "$source_dir" >&2
    exit 1
fi

mkdir -p -- "$backup_dir"
timestamp=$(date +%Y%m%d-%H%M%S)
source_name=$(basename -- "$source_dir")
parent_dir=$(dirname -- "$source_dir")
archive="$backup_dir/${source_name}-${timestamp}.tar.gz"

tar -czf "$archive" -C "$parent_dir" "$source_name"
printf 'backup created: %s\n' "$archive"
```

**关键点：** 使用 `-C` 和 basename 避免把绝对路径层级写入归档；所有路径都引用；源目录和备份目录可能位于同一文件系统时要考虑备份文件是否被再次包含；生产脚本还应加入排除规则、保留策略、校验和、加密和空间检查。

### Q32：检查磁盘使用率并告警

**题目：** 检查挂载点使用率，超过阈值时输出告警并返回失败。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

threshold=${1:-80}
mount_filter=${2:-/}

if [[ $threshold =~ ^[0-9]+$ ]] && (( threshold >= 0 && threshold <= 100 )); then
    :
else
    printf 'threshold must be an integer between 0 and 100\n' >&2
    exit 2
fi

line=$(df -P -- "$mount_filter" | awk 'NR == 2 { print $5, $6 }')
read -r usage mount_point <<< "$line"
usage=${usage%%%}

if (( usage >= threshold )); then
    printf 'ALERT mount=%s usage=%s%% threshold=%s%%\n' \
        "$mount_point" "$usage" "$threshold" >&2
    exit 1
fi

printf 'OK mount=%s usage=%s%% threshold=%s%%\n' \
    "$mount_point" "$usage" "$threshold"
```

**关键点：** `df -P` 让输出更适合脚本解析；`--` 和双引号保护路径；Linux/GNU 与 macOS `df` 参数存在差异；生产环境应处理挂载点不存在、网络文件系统、inode 使用率、容器 cgroup 和告警去重。

### Q33：统计访问日志中 HTTP 状态码

**题目：** 解析常见 Apache/Nginx combined log，输出状态码和数量，按数量降序排列。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

log_file=${1:?"usage: $0 ACCESS_LOG"}

if [[ ! -r $log_file ]]; then
    printf 'log file is not readable: %s\n' "$log_file" >&2
    exit 1
fi

awk '{ count[$9]++ } END { for (status in count) print status, count[status] }' "$log_file" \
    | sort -k2,2nr -k1,1
```

**关键点：** 必须先确认日志格式，`$9` 只适用于特定 combined log；格式不稳定时应使用明确的解析器或 `jq`；生产脚本要处理日志轮转、压缩日志、坏行、时区和超大文件，并避免把敏感 URL 直接输出到公共日志。

### Q34：找出目录中最大的 N 个文件

**题目：** 递归查找指定目录下的普通文件，按大小降序输出前 N 个，路径包含空格时不能出错。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

root=${1:-.}
limit=${2:-10}

if [[ ! $limit =~ ^[1-9][0-9]*$ ]]; then
    printf 'limit must be a positive integer\n' >&2
    exit 2
fi

find "$root" -xdev -type f -printf '%s\t%p\0' 2>/dev/null \
    | sort -znr \
    | head -z -n "$limit" \
    | while IFS=$'\t' read -r -d '' size path; do
        printf '%s bytes\t%s\n' "$size" "$path"
    done
```

**关键点：** `-print0` 或 NUL 分隔可以安全处理特殊路径；上例使用 GNU `find`/`sort`/`head` 的 NUL 选项，macOS/BSD 需要改写或使用 Python；`-xdev` 避免跨文件系统；排查磁盘问题前要考虑权限、挂载点和删除但仍打开的文件。

### Q35：安全清理指定天数前的文件

**题目：** 删除目录下超过保留天数的普通文件，要求限制在同一文件系统，先支持 dry-run。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

root=${1:?"usage: $0 ROOT RETENTION_DAYS [--apply]"}
retention_days=${2:?"usage: $0 ROOT RETENTION_DAYS [--apply]"}
mode=${3:---dry-run}

if [[ ! $retention_days =~ ^[0-9]+$ ]]; then
    printf 'retention days must be a non-negative integer\n' >&2
    exit 2
fi
if [[ $mode != --dry-run && $mode != --apply ]]; then
    printf 'mode must be --dry-run or --apply\n' >&2
    exit 2
fi

while IFS= read -r -d '' file; do
    if [[ $mode == --dry-run ]]; then
        printf 'would remove: %s\n' "$file"
    else
        rm -- "$file"
        printf 'removed: %s\n' "$file"
    fi
done < <(find "$root" -xdev -type f -mtime "+$retention_days" -print0)
```

**关键点：** 默认 dry-run 是破坏性运维脚本的基本保护；不要把用户输入直接拼到 `find` 表达式中；生产实现还要排除当前日志、锁文件和业务目录，记录删除审计，处理并发写入，并在删除前验证根目录不是空字符串或 `/`。

### Q36：实现带重试和超时的 HTTP 检查

**题目：** 检查 URL，失败后有限次重试，每次使用连接和总请求超时，最终失败返回非零。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

url=${1:?"usage: $0 URL"}
max_attempts=${2:-3}

if [[ ! $max_attempts =~ ^[1-9][0-9]*$ ]]; then
    printf 'attempts must be positive\n' >&2
    exit 2
fi

for ((attempt = 1; attempt <= max_attempts; attempt++)); do
    if curl --fail --silent --show-error \
        --connect-timeout 3 --max-time 10 \
        --output /dev/null -- "$url"; then
        printf 'healthy: %s\n' "$url"
        exit 0
    fi

    printf 'attempt %s/%s failed: %s\n' "$attempt" "$max_attempts" "$url" >&2
    if (( attempt < max_attempts )); then
        sleep $((2 ** (attempt - 1)))
    fi
done

printf 'unhealthy: %s\n' "$url" >&2
exit 1
```

**关键点：** 连接超时和总超时都要设置；`curl --fail` 让 HTTP 4xx/5xx 返回失败；重试不应无上限，生产环境应使用指数退避、抖动、状态码白名单、TLS 校验、代理配置和指标。

### Q37：等待端口可用

**题目：** 在部署或测试中等待某个 TCP 端口监听，超过超时时间返回失败。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

host=${1:?"usage: $0 HOST PORT [TIMEOUT_SECONDS]"}
port=${2:?"usage: $0 HOST PORT [TIMEOUT_SECONDS]"}
timeout=${3:-30}
started_at=$SECONDS

while (( SECONDS - started_at < timeout )); do
    if nc -z -w 1 -- "$host" "$port" 2>/dev/null; then
        printf 'port is ready: %s:%s\n' "$host" "$port"
        exit 0
    fi
    sleep 1
done

printf 'timed out waiting for %s:%s\n' "$host" "$port" >&2
exit 1
```

**关键点：** TCP 端口可连接不代表应用健康，应进一步请求健康 API 或执行业务探针；端口和 host 要做格式校验；精确超时和 `SECONDS` 行为要结合 Shell 版本验证。

### Q38：防止脚本并发执行

**题目：** 一个定时任务不能同时运行多个实例，要求拿不到锁时快速退出。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

lock_file=${1:-/tmp/example-task.lock}
exec 9>"$lock_file"

if ! flock -n 9; then
    printf 'another instance is running\n' >&2
    exit 1
fi

printf 'lock acquired by pid=%s\n' "$$"
trap 'printf "releasing lock\n"' EXIT

# Put the protected operation here.
sleep 1
```

**关键点：** 文件描述符持有期间锁保持有效，脚本退出后内核释放锁；锁文件本身可以存在，不要仅用 `if [[ -e lock ]]` 判断，因为进程崩溃后会留下 stale file；`flock` 不是所有系统都有，macOS 可用 `mkdir` 原子创建或安装兼容工具。

### Q39：限制并发执行多个文件任务

**题目：** 对多个文件执行处理命令，最多同时运行 N 个任务，任何任务失败时最终返回非零。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

max_parallel=${1:?"usage: $0 MAX_PARALLEL FILE..."}
shift

if [[ ! $max_parallel =~ ^[1-9][0-9]*$ ]]; then
    printf 'max parallel must be positive\n' >&2
    exit 2
fi

pids=()
failed=0

process_file() {
    local file=$1
    [[ -f $file ]] || {
        printf 'not a regular file: %s\n' "$file" >&2
        return 1
    }
    sha256sum -- "$file" >/dev/null
}

wait_one() {
    local pid=$1
    if ! wait "$pid"; then
        failed=1
    fi
}

for file in "$@"; do
    process_file "$file" &
    pids+=("$!")
    if (( ${#pids[@]} >= max_parallel )); then
        wait_one "${pids[0]}"
        pids=("${pids[@]:1}")
    fi
done

for pid in "${pids[@]}"; do
    wait_one "$pid"
done

exit "$failed"
```

**关键点：** 这是简单的固定并发窗口，等待最早加入的 PID 可能降低完成顺序上的吞吐；生产实现应处理取消、子进程信号、任务 ID、日志隔离、失败重试和资源配额，任务本身也必须是幂等的。

### Q40：解析配置并安全启动服务

**题目：** 从简单的 `KEY=VALUE` 配置文件读取端口和环境，不执行配置内容，然后启动命令。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

config_file=${1:?"usage: $0 CONFIG_FILE"}
port=8080
environment=production

while IFS='=' read -r key value; do
    [[ -z $key || $key == \#* ]] && continue
    case $key in
        PORT) port=$value ;;
        ENVIRONMENT) environment=$value ;;
        *) printf 'unknown config key: %s\n' "$key" >&2; exit 2 ;;
    esac
done < "$config_file"

[[ $port =~ ^[0-9]+$ ]] || { printf 'invalid port\n' >&2; exit 2; }
[[ $environment =~ ^[a-zA-Z0-9_-]+$ ]] || {
    printf 'invalid environment\n' >&2
    exit 2
}

export APP_ENV="$environment"
exec /opt/example/bin/server --port "$port"
```

**关键点：** 不要使用 `source "$config_file"` 读取不可信配置，因为它会执行任意 Shell 代码；使用白名单解析键名和值；`exec` 让服务进程替换 Shell，便于信号和退出码正确传递。

---

## 五、运维现场 Coding 题

### Q41：检查服务状态并自动恢复

**题目：** 编写脚本检查 systemd 服务；服务不活跃时尝试重启，重启后再次验证，失败时输出诊断信息。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

service_name=${1:?"usage: $0 SERVICE_NAME"}

if [[ ! $service_name =~ ^[a-zA-Z0-9_.@-]+$ ]]; then
    printf 'invalid service name\n' >&2
    exit 2
fi

if systemctl is-active --quiet "$service_name"; then
    printf 'OK service=%s\n' "$service_name"
    exit 0
fi

printf 'service is inactive, restarting: %s\n' "$service_name" >&2
systemctl restart "$service_name"
sleep 2

if systemctl is-active --quiet "$service_name"; then
    printf 'RECOVERED service=%s\n' "$service_name"
    exit 0
fi

printf 'FAILED service=%s\n' "$service_name" >&2
systemctl status "$service_name" --no-pager >&2 || true
journalctl -u "$service_name" -n 50 --no-pager >&2 || true
exit 1
```

**关键点：** 重启不是万能恢复策略，必须限制重试并防止重启风暴；生产中还要检查依赖服务、健康接口、最近发布版本、资源使用、告警抑制和人工升级路径。

### Q42：检查进程内存并输出告警

**题目：** 根据进程名查找 RSS 最大的进程，超过阈值时返回非零。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

process_name=${1:?"usage: $0 PROCESS_NAME RSS_MB"}
threshold_mb=${2:?"usage: $0 PROCESS_NAME RSS_MB"}

[[ $process_name =~ ^[a-zA-Z0-9_.-]+$ ]] || {
    printf 'invalid process name\n' >&2
    exit 2
}
[[ $threshold_mb =~ ^[1-9][0-9]*$ ]] || {
    printf 'invalid threshold\n' >&2
    exit 2
}

rss_kb=$(ps -eo comm=,rss= | awk -v name="$process_name" '$1 == name { if ($2 > max) max = $2 } END { print max + 0 }')
rss_mb=$((rss_kb / 1024))

if (( rss_mb >= threshold_mb )); then
    printf 'ALERT process=%s rss_mb=%s threshold_mb=%s\n' \
        "$process_name" "$rss_mb" "$threshold_mb" >&2
    exit 1
fi

printf 'OK process=%s rss_mb=%s threshold_mb=%s\n' \
    "$process_name" "$rss_mb" "$threshold_mb"
```

**关键点：** RSS 是驻留内存，不等于完整进程内存或容器 memory usage；同名进程可能有多个实例；生产监控应使用稳定的 PID、服务标签和 Prometheus/cgroup 指标，并区分短时峰值与持续增长。

### Q43：统计最近日志中的 Top IP

**题目：** 从访问日志中找出最近 N 行请求最多的客户端 IP，并输出前 10 名。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

log_file=${1:?"usage: $0 ACCESS_LOG [LINES]"}
lines=${2:-100000}

[[ $lines =~ ^[1-9][0-9]*$ ]] || {
    printf 'lines must be positive\n' >&2
    exit 2
}

tail -n "$lines" -- "$log_file" \
    | awk 'NF > 0 { count[$1]++ } END { for (ip in count) print count[ip], ip }' \
    | sort -k1,1nr -k2,2 \
    | head -n 10
```

**关键点：** `$1` 只有在日志第一列是客户端 IP 时才正确；反向代理场景要确认真实客户端 IP 的信任边界，不能盲信任任意 `X-Forwarded-For`；大文件应使用日志系统或流式聚合，避免在生产主机上频繁扫描全量日志。

### Q44：批量检查主机连通性

**题目：** 从主机清单读取地址，跳过空行和注释，并行检查 SSH 连通性，输出成功和失败主机。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

hosts_file=${1:?"usage: $0 HOSTS_FILE"}
max_parallel=${2:-10}

check_host() {
    local host=$1
    if ssh -o BatchMode=yes -o ConnectTimeout=5 -- "$host" true >/dev/null 2>&1; then
        printf 'OK %s\n' "$host"
        return 0
    fi
    printf 'FAILED %s\n' "$host" >&2
    return 1
}

pids=()
failed=0
while IFS= read -r host || [[ -n $host ]]; do
    [[ -z $host || $host == \#* ]] && continue
    check_host "$host" &
    pids+=("$!")
    if (( ${#pids[@]} >= max_parallel )); then
        if ! wait "${pids[0]}"; then
            failed=1
        fi
        pids=("${pids[@]:1}")
    fi
done < "$hosts_file"

for pid in "${pids[@]}"; do
    if ! wait "$pid"; then
        failed=1
    fi
done

exit "$failed"
```

**关键点：** `BatchMode=yes` 避免脚本卡在密码提示；SSH 并发量必须受跳板机、目标主机和网络容量限制；生产脚本应使用密钥、known_hosts 校验、连接超时、审计和结果落盘，不应关闭主机密钥检查来“解决”失败。

### Q45：检查服务端口和 HTTP 健康状态

**题目：** 先检查 TCP 端口，再检查 HTTP 健康接口，输出耗时和状态码。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

url=${1:?"usage: $0 URL"}
expected_status=${2:-200}

result=$(curl --silent --show-error --output /dev/null \
    --write-out '%{http_code} %{time_total}' \
    --connect-timeout 3 --max-time 10 -- "$url")

read -r status elapsed <<< "$result"
if [[ $status == "$expected_status" ]]; then
    printf 'OK url=%s status=%s elapsed=%ss\n' "$url" "$status" "$elapsed"
    exit 0
fi

printf 'FAILED url=%s status=%s expected=%s elapsed=%ss\n' \
    "$url" "$status" "$expected_status" "$elapsed" >&2
exit 1
```

**关键点：** HTTP 200 也可能返回错误业务内容，应根据服务契约检查响应体或业务字段；监控需要区分 DNS、连接、TLS、HTTP 状态和响应时间；不要把完整响应中的 token 或个人信息写入日志。

### Q46：安全执行发布并支持快速回滚

**题目：** 使用版本目录和 `current` 符号链接发布服务。新版本检查失败时不切换，切换后健康检查失败时回滚。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

version=${1:?"usage: $0 VERSION"}
release_root=/opt/example/releases
current_link=/opt/example/current
health_url=http://127.0.0.1:8080/health
new_release="$release_root/$version"
previous_target=''

[[ $version =~ ^[a-zA-Z0-9._-]+$ ]] || {
    printf 'invalid version\n' >&2
    exit 2
}
[[ -d $new_release ]] || {
    printf 'release does not exist: %s\n' "$new_release" >&2
    exit 1
}

if [[ -L $current_link ]]; then
    previous_target=$(readlink "$current_link")
fi

if [[ ! -x $new_release/bin/server ]]; then
    printf 'release validation failed: server is not executable\n' >&2
    exit 1
fi

ln -sfn "$new_release" "${current_link}.next"
mv -Tf "${current_link}.next" "$current_link"

if ! curl --fail --silent --show-error --max-time 10 -- "$health_url" >/dev/null; then
    printf 'health check failed, rolling back\n' >&2
    if [[ -n $previous_target ]]; then
        ln -sfn "$previous_target" "${current_link}.rollback"
        mv -Tf "${current_link}.rollback" "$current_link"
    fi
    exit 1
fi

printf 'deployed version=%s\n' "$version"
```

**关键点：** 版本目录应不可变，切换符号链接要尽量原子；生产发布还要停止或重载服务、等待连接排空、验证数据库/API 兼容、记录发布 ID、限制并发发布并确保回滚版本仍然存在。

### Q47：检查并清理删除但仍被进程打开的日志

**题目：** 找出已从目录树删除、但仍被进程打开的文件，输出进程和文件大小；不要自动删除。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

printf 'deleted files still held open:\n'
lsof -nP +L1 2>/dev/null \
    | awk 'NR == 1 || $NF ~ /\(deleted\)$/ { print }'
```

**关键点：** 这类文件空间不会因为目录项删除而立即释放，只有最后一个文件描述符关闭后才释放；正确处理通常是让应用 reopen、重启或发送日志轮转信号，而不是再次 `rm`。命令输出格式可能随系统和 lsof 版本变化，自动化解析应优先使用机器可读接口。

### Q48：监控目录文件数量并触发告警

**题目：** 当某目录下文件数量超过阈值时告警，路径可能包含特殊字符。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

directory=${1:?"usage: $0 DIRECTORY LIMIT"}
limit=${2:?"usage: $0 DIRECTORY LIMIT"}

[[ -d $directory ]] || {
    printf 'directory does not exist: %s\n' "$directory" >&2
    exit 2
}
[[ $limit =~ ^[0-9]+$ ]] || {
    printf 'limit must be a non-negative integer\n' >&2
    exit 2
}

file_count=$(find "$directory" -xdev -type f -printf . | wc -c)
file_count=${file_count//[[:space:]]/}

if (( file_count > limit )); then
    printf 'ALERT directory=%s files=%s limit=%s\n' \
        "$directory" "$file_count" "$limit" >&2
    exit 1
fi

printf 'OK directory=%s files=%s limit=%s\n' "$directory" "$file_count" "$limit"
```

**关键点：** `find -printf` 是 GNU 扩展，跨平台时可使用 `find ... -print0 | awk -v RS='\\0' 'END { print NR }'` 或 Python；大量文件会产生 I/O 压力，生产监控应考虑目录分片、inode 指标和采样。

### Q49：批量执行命令并生成失败清单

**题目：** 从任务文件读取任务 ID，对每个 ID 执行命令，成功和失败分别写入结果文件，失败任务可重跑。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

task_file=${1:?"usage: $0 TASK_FILE"}
success_file=${2:-success.txt}
failed_file=${3:-failed.txt}

: > "$success_file"
: > "$failed_file"

while IFS= read -r task_id || [[ -n $task_id ]]; do
    [[ -z $task_id || $task_id == \#* ]] && continue
    if [[ $task_id =~ ^[a-zA-Z0-9._:-]+$ ]] \
        && /opt/example/bin/process-task --id "$task_id"; then
        printf '%s\n' "$task_id" >> "$success_file"
    else
        printf '%s\n' "$task_id" >> "$failed_file"
    fi
done < "$task_file"

if [[ -s $failed_file ]]; then
    printf 'some tasks failed; see %s\n' "$failed_file" >&2
    exit 1
fi
```

**关键点：** 参数使用 `--id "$task_id"` 而不是字符串拼接；结果文件应使用临时文件加原子替换，避免脚本中断留下半成品；任务命令必须幂等，重跑时要能识别已成功任务。

### Q50：编写优雅停止脚本

**题目：** 启动一个后台服务，收到 `TERM` 或 `INT` 后先通知子进程退出，等待超时后再强制终止。

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

command_to_run=(/opt/example/bin/server --config /etc/example/server.conf)
shutdown_timeout=${SHUTDOWN_TIMEOUT:-20}
child_pid=''

cleanup() {
    local status=$?
    if [[ -n $child_pid ]] && kill -0 "$child_pid" 2>/dev/null; then
        kill -TERM "$child_pid" 2>/dev/null || true
        for ((second = 0; second < shutdown_timeout; second++)); do
            if ! kill -0 "$child_pid" 2>/dev/null; then
                break
            fi
            sleep 1
        done
        if kill -0 "$child_pid" 2>/dev/null; then
            kill -KILL "$child_pid" 2>/dev/null || true
        fi
    fi
    exit "$status"
}

trap cleanup INT TERM EXIT
"${command_to_run[@]}" &
child_pid=$!
wait "$child_pid"
```

**关键点：** 使用数组避免命令参数重新解析；保存原始退出码；`TERM` 是请求优雅停止，`KILL` 只是最终兜底；生产环境要处理子进程树、进程组、连接排空、日志 flush 和 PID 复用风险。

---

## 六、排障追问与测试清单

### 1. Shell 现场题常见追问

- 如果路径包含空格、换行、通配符或以 `-` 开头，脚本是否仍然正确？
- `set -e` 在 `if`、管道、命令替换和函数中是否会按预期工作？
- 子命令失败时，脚本能否保留准确的退出码？
- 临时文件、锁文件和后台进程在异常退出时是否会清理？
- 脚本被重复执行、并发执行或收到 `TERM` 时行为是什么？
- 命令依赖是 GNU 还是 BSD？容器镜像是否包含这些工具？

### 2. 运维排障题常见追问

- CPU 高、load 高、iowait 高和网络延迟高如何区分？
- 磁盘空间满、inode 满和删除但仍打开的文件如何区分？
- 端口监听、TCP 可连接、TLS 成功和 HTTP 健康分别说明什么？
- 服务重启是否会造成数据丢失、请求失败或重启风暴？
- 日志轮转后应用是否仍写旧文件描述符？
- 容器资源限制与宿主机资源使用是否一致？

### 3. 最低测试集合

每个脚本至少验证：

1. 正常输入和最小合法输入。
2. 参数缺失、非法格式、空文件和不存在路径。
3. 路径包含空格、引号、通配符、换行或前导 `-`。
4. 外部命令失败、网络超时、服务不存在和权限不足。
5. 并发执行、重复执行、中途 `INT`/`TERM` 和机器重启。
6. 大文件、大目录、日志轮转和磁盘接近满的情况。

### 4. 推荐的 Shell 验证命令

```bash
bash -n script.sh
shellcheck script.sh
bash -u script.sh
printf '%s\n' 'input' | bash script.sh
```

破坏性脚本应先使用临时目录和 dry-run：

```bash
test_root=$(mktemp -d)
trap 'rm -rf -- "$test_root"' EXIT
mkdir -p -- "$test_root/data"
touch -- "$test_root/data/sample.log"
./cleanup.sh "$test_root/data" 0 --dry-run
```

### 5. 面试中应主动说明的工程化改进

- 将业务逻辑拆成可测试函数，主流程只负责参数、编排和退出码。
- 将 `date`、`sleep`、`curl`、`systemctl` 等外部依赖包装起来，便于 mock 或在容器中测试。
- 使用结构化日志、请求 ID、主机名、版本号和退出原因。
- 为重试、超时、并发和资源上限设置明确默认值与配置项。
- 通过 CI 执行 ShellCheck、语法检查、单元测试和受控集成测试。
- 破坏性运维操作必须有 dry-run、审批、审计和回滚路径。

---

## 七、面试前自检

### Shell 基础

- 我能解释 shebang、变量展开、双引号、数组、函数、退出码和文件描述符。
- 我能说清 `set -euo pipefail` 的作用和例外，而不是只会复制模板。
- 我能安全处理带空格和特殊字符的文件名，知道为什么要用 `find -print0`。
- 我能解释管道、子 Shell、命令替换、进程替换和 `PIPESTATUS`。
- 我能写出包含参数校验、日志、trap、清理和退出码的 Bash 脚本。

### Linux 运维

- 我能从 CPU、内存、磁盘、inode、网络、进程和日志多个维度定位故障。
- 我能区分端口可用、服务健康、业务可用和依赖正常。
- 我能解释 systemd、cron、logrotate、权限、umask、cgroup 和容器 PID 1。
- 我能设计重试、超时、限流、幂等、锁、回滚和告警抑制。
- 我能说明自动重启的风险，并定义停止、恢复、升级和人工接管条件。

### 现场表达

- 写代码前我会先确认 Shell、操作系统、工具版本和权限边界。
- 我会先给可验证的基础实现，再讨论 GNU/BSD 兼容和生产增强。
- 我能主动说出破坏性命令的 dry-run、审计、回滚和恢复方案。
- 项目案例使用真实数据，明确个人职责，不把团队成果全部归于自己。

> 运维现场 Coding 的目标不是写出命令最多的脚本，而是在故障、权限、并发和不完整信息下，仍能安全地自动化操作、清楚地暴露失败，并留下可恢复的路径。
