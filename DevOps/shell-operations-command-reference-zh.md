# 运维 Shell 常用命令实战手册

> 面向 Linux 日常巡检、故障排查、服务发布、日志分析和自动化运维。命令示例以 Bash 与 systemd 管理的 Linux 服务器为主；发行版、工具版本和权限不同，实际输出会有差异。
>
> **安全原则：** 先在只读或 `--dry-run` 模式确认目标，再执行写操作；生产环境先确认主机、环境、路径、影响范围和变更窗口。不要把密码、令牌、私钥或完整生产数据写入终端历史、脚本、日志或工单。

## 目录

1. [执行命令前的安全约定](#一执行命令前的安全约定)
2. [文件与目录](#二文件与目录)
3. [文本、日志与管道](#三文本日志与管道)
4. [权限、用户与配置](#四权限用户与配置)
5. [进程、资源与系统服务](#五进程资源与系统服务)
6. [网络与远程操作](#六网络与远程操作)
7. [计划任务、归档与数据传输](#七计划任务归档与数据传输)
8. [可靠 Shell 脚本模式](#八可靠-shell-脚本模式)
9. [真实故障排查场景](#九真实故障排查场景)
10. [高频命令速查](#十高频命令速查)

---

## 一、执行命令前的安全约定

### 1. 确认主机、身份和当前目录

在生产环境执行任何命令前，先确认自己连接的主机、当前用户和目标路径。

```bash
hostnamectl --static
id
pwd
```

**输入：** 在目标主机执行以上三条命令。

**示例输出：**

```text
payments-prod-01
uid=1001(ops) gid=1001(ops) groups=1001(ops),27(sudo)
/srv/payments
```

**使用场景：** 值班人员同时连接测试和生产环境时，先核对主机名和路径，避免在错误环境执行重启、删除或迁移操作。

### 2. 使用命令帮助和安全预览

```bash
man rsync
find /var/log/myapp -type f -name '*.log' -mtime +14 -print
rsync -avzn --delete /srv/release/ ops@web-02:/srv/release/
```

- `man`：阅读本机安装版本的手册。
- `find ... -print`：先列出将被处理的文件；确认无误后才考虑 `-delete`。
- `rsync -n`：演练模式，不会传输或删除文件。

**示例输出：**

```text
/var/log/myapp/api-2026-09-01.log
/var/log/myapp/worker-2026-09-01.log
deleting old/static/app.7f3d.js
static/app.9c21.js
```

**注意：** `rm -rf`、`find -delete`、`rsync --delete`、`chmod -R`、`chown -R` 都可能造成大范围破坏。执行前务必使用绝对路径、预览目标并获得变更批准。

### 3. 退出码、重定向和管道

Unix 命令通常以退出码表达结果：`0` 表示成功，非 `0` 表示失败或特殊状态。不要只看命令是否有输出。

```bash
curl -fsS https://api.example.com/health >/dev/null
printf 'exit=%s\n' "$?"

grep -q 'ERROR' /var/log/myapp/app.log
printf 'exit=%s\n' "$?"
```

**示例输出：**

```text
exit=0
exit=1
```

第二个命令中的 `1` 仅表示没有匹配行，不一定代表系统错误；自动化脚本要按具体命令的语义处理退出码。

常用重定向：

```bash
command > stdout.log              # 覆盖写入标准输出
command >> stdout.log             # 追加标准输出
command 2> stderr.log             # 写入标准错误
command > all.log 2>&1            # 合并标准输出和标准错误
command | tee -a deploy.log       # 终端显示并追加到文件
```

---

## 二、文件与目录

### 4. `ls`、`find`、`stat`：定位文件并确认元数据

```bash
ls -lah /etc/nginx/conf.d/
find /srv/myapp -type f -name '*.log' -mtime -1 -size +100M -print
stat /srv/myapp/releases/current
```

**作用：**

- `ls -lah`：以易读大小显示目录中的隐藏文件、权限、所有者和修改时间。
- `find`：递归按类型、名称、时间、大小等条件定位文件。
- `stat`：显示单个文件或链接的详细元数据；比 `ls` 更适合确认时间戳、inode 和权限。

**示例输出：**

```text
total 16K
drwxr-xr-x 2 root root 4.0K Oct 21 09:12 .
-rw-r----- 1 root nginx 1.8K Oct 21 08:54 api.conf

/srv/myapp/logs/access-2026-10-21.log

  File: /srv/myapp/releases/current -> /srv/myapp/releases/20261021-085500
  Access: (0775/lrwxrwxrwx)  Uid: ( 1001/     ops)   Gid: ( 1001/     ops)
```

**真实场景：** 发布后 Nginx 返回 404。先用 `stat` 确认 `current` 符号链接是否指向预期发布目录，再检查该目录内静态文件是否存在，而不是立即重启服务。

### 5. `df`、`du`：区分文件系统已满和目录异常膨胀

```bash
df -hT /var
du -xhd1 /var | sort -h
```

**作用：**

- `df -hT /var`：查看 `/var` 所在文件系统容量、使用率和文件系统类型。
- `du -xhd1 /var`：统计 `/var` 下第一层目录大小；`-x` 防止跨越挂载点，`sort -h` 按可读容量排序。

**示例输出：**

```text
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/nvme0n1p3 ext4   80G   76G  1.0G  99% /var

2.1G    /var/cache
14G     /var/lib
58G     /var/log
76G     /var
```

**真实场景：** 告警提示磁盘使用率 99%。`df` 先确定哪个挂载点满了，`du` 再定位异常目录。若 `df` 很高但 `du` 合计明显偏小，可能是进程仍打开了已删除文件，见 [Q17：`lsof`](#17-lsof定位端口与已删除但仍被占用的文件)。

### 6. `cp`、`mv`、`ln -sfn`：安全切换发布版本

```bash
cp -a /etc/myapp/app.env /etc/myapp/app.env.bak.20261021
ln -sfn /srv/myapp/releases/20261021-085500 /srv/myapp/current
readlink -f /srv/myapp/current
```

**作用：**

- `cp -a`：尽量保留权限、时间戳和链接属性，适合修改配置前备份。
- `ln -sfn`：将 `current` 符号链接原子地切换到新发布目录；`-n` 避免把链接当作目录跟随。
- `readlink -f`：解析链接最终指向，做发布后校验。

**示例输出：**

```text
/srv/myapp/releases/20261021-085500
```

**真实场景：** 采用不可变发布目录时，部署新版本后切换 `current` 链接，再平滑重载服务。若健康检查失败，切回上一个已知正常的发布目录并重载，比原地覆盖文件更易回滚。

### 7. `tar`：打包、解包和预览归档内容

```bash
tar -tzf backup-20261021.tar.gz | head -20
tar -xzf release-20261021.tar.gz -C /srv/myapp/releases/20261021-085500
tar -czf nginx-config-20261021.tar.gz -C /etc nginx
```

**作用：**

- `-t`：列出压缩包内容，解包前先检查路径。
- `-x`：解包；`-C` 指定目标目录。
- `-c`：创建压缩包。
- `-z`：使用 gzip；压缩格式应与归档扩展名一致。

**示例输出：**

```text
nginx/
nginx/nginx.conf
nginx/conf.d/api.conf
```

**真实场景：** 从制品库下载发布包后，先列表检查它是否含有意外的绝对路径或上级目录，再解包到新的版本目录。不要直接把来源不明的压缩包解到 `/` 或应用根目录。

---

## 三、文本、日志与管道

### 8. `cat`、`less`、`head`、`tail`：查看配置和日志

```bash
head -n 20 /etc/nginx/nginx.conf
tail -n 100 /var/log/myapp/app.log
tail -F /var/log/myapp/app.log
less -S /var/log/myapp/app.log
```

**作用：**

- `head -n`：显示文件开头若干行，常用于检查文件头、CSV 字段或配置。
- `tail -n`：显示末尾若干行，适合查看最新错误。
- `tail -F`：持续跟踪文件名，即使日志轮转后文件被替换也会重新打开；按 `Ctrl-C` 退出。
- `less -S`：分页且不自动折行，使用 `/关键字` 搜索、`n` 查找下一处、`q` 退出。

**示例输出：**

```text
2026-10-21T09:31:18Z INFO request_id=ab91 method=GET path=/health status=200 duration_ms=4
2026-10-21T09:31:25Z ERROR request_id=f72c upstream timeout service=payment timeout_s=3
```

**真实场景：** 线上刚发生 502 时，先查看最近 100 行再持续跟踪新增日志，并同时记录 `request_id`，以便串联反向代理、应用和下游服务的日志。

### 9. `grep`：按模式过滤日志

```bash
grep -nE 'ERROR|CRITICAL' /var/log/myapp/app.log | tail -50
grep -RIn --include='*.conf' 'proxy_read_timeout' /etc/nginx
grep -c ' status=500 ' /var/log/myapp/access.log
```

**作用：**

- `-n`：显示行号。
- `-E`：启用扩展正则，例如 `A|B`。
- `-R`：递归查找；`-I` 忽略二进制文件。
- `-c`：只统计匹配行数。

**示例输出：**

```text
341:2026-10-21T09:31:25Z ERROR request_id=f72c upstream timeout service=payment
/etc/nginx/conf.d/api.conf:18:proxy_read_timeout 10s;
27
```

**真实场景：** 客户报告间歇性超时。先用 `grep` 获取错误时间和请求 ID，再从同一时间窗口的代理日志、应用日志和下游日志中关联问题。

### 10. `awk`、`cut`、`sort`、`uniq`：快速聚合访问日志

假设访问日志格式如下：

```text
203.0.113.8 - - [21/Oct/2026:09:31:25 +0000] "GET /api/orders HTTP/1.1" 500 231 "-" "curl/8.0"
```

统计返回 5xx 最多的请求路径：

```bash
awk '$9 ~ /^5[0-9][0-9]$/ {print $7}' /var/log/nginx/access.log \
  | sort \
  | uniq -c \
  | sort -nr \
  | head -10
```

**示例输出：**

```text
    127 /api/orders
     42 /api/payments/confirm
      8 /health
```

**解释：**

- `awk '$9 ... {print $7}'`：若第 9 列是 5xx，则输出第 7 列路径。
- `sort`：让相同行相邻。
- `uniq -c`：计数相邻重复行。
- `sort -nr`：按数字倒序。

**真实场景：** HTTP 5xx 比例上升时，用该聚合快速判断是单一接口故障还是全站问题。日志格式变更后字段位置可能不同，应先抽样确认字段而不是盲目复用命令。

### 11. `sed`：安全查看和受控替换文本

```bash
sed -n '1,80p' /etc/myapp/app.env
sed -n '/^\[Service\]/,/^\[/p' /etc/systemd/system/myapp.service
sed -i.bak 's/^WORKERS=.*/WORKERS=8/' /etc/myapp/app.env
```

**作用：**

- `-n '1,80p'`：仅打印第 1 至 80 行。
- `/开始/,/结束/p`：打印一个配置段。
- `-i.bak`：原地替换并保留 `.bak` 备份；GNU/Linux 的 `sed -i` 和 macOS 的参数略有差异。

**示例输出：**

```text
[Service]
User=myapp
EnvironmentFile=/etc/myapp/app.env
ExecStart=/srv/myapp/current/bin/start
```

**真实场景：** 临时调整 worker 数量时，先用 `sed -n` 检查目标行是否唯一，再生成带备份的修改；随后进行配置语法验证和服务平滑重载。不要对来源不明或未经转义的变量拼接 `sed` 替换表达式。

### 12. `jq`：查看和提取 JSON（安装后使用）

```bash
curl -fsS https://api.example.com/health | jq .
printf '%s\n' '{"service":"orders","status":"ok","latency_ms":12}' \
  | jq -r '.service + " " + .status'
```

**示例输出：**

```json
{
  "service": "orders",
  "status": "ok",
  "version": "2026.10.21"
}
```

```text
orders ok
```

**真实场景：** 健康检查、云 API 和容器编排 API 经常返回 JSON。`jq` 比 `grep`/`cut` 更能正确处理 JSON 字符串转义、字段顺序和嵌套结构；脚本依赖它时，应明确声明并检查该工具已安装。

---

## 四、权限、用户与配置

### 13. `id`、`getent`、`sudo -l`：确认身份与权限来源

```bash
id deploy
getent passwd myapp
getent group nginx
sudo -l
```

**作用：**

- `id`：显示用户的 UID、主组和附加组。
- `getent`：通过系统的名称服务配置查询用户、用户组等；比仅查看本地 `/etc/passwd` 更适合接入 LDAP/SSSD 的环境。
- `sudo -l`：列出当前用户被授予的 sudo 命令范围。

**示例输出：**

```text
uid=1002(deploy) gid=1002(deploy) groups=1002(deploy),998(release)
myapp:x:997:997:Application user:/srv/myapp:/usr/sbin/nologin
User ops may run the following commands on payments-prod-01:
    (root) /bin/systemctl reload myapp, /usr/bin/journalctl -u myapp
```

**真实场景：** 应用因文件权限无法读配置时，先确认服务进程实际运行用户和文件组，而不是不加判断地执行 `chmod 777`。生产 sudo 权限应遵循最小权限原则。

### 14. `chmod`、`chown`、`umask`：管理访问权限

```bash
stat -c '%A %U:%G %n' /etc/myapp/app.env
sudo chown root:myapp /etc/myapp/app.env
sudo chmod 0640 /etc/myapp/app.env
umask
```

**示例输出：**

```text
-rw-r----- root:myapp /etc/myapp/app.env
0022
```

**解释：**

- `0640` 表示所有者可读写、同组可读、其他用户无权限。
- `chown root:myapp` 适用于由 root 管理但应用组需要读取的配置。
- `umask 0022` 意味着新建普通文件通常不会对组/其他用户开放写权限。

**真实场景：** 应用密钥文件应只让服务账户或必要用户组读取。更严格的私钥通常使用 `0600`；权限设计需要与服务运行用户和密钥管理方案一致。

### 15. `diff`、`cmp`、`sha256sum`：比较配置和制品完整性

```bash
diff -u /etc/myapp/app.env.bak.20261021 /etc/myapp/app.env
sha256sum release-20261021.tar.gz
sha256sum -c release-20261021.tar.gz.sha256
```

**示例输出：**

```diff
-WORKERS=4
+WORKERS=8
```

```text
release-20261021.tar.gz: OK
```

**真实场景：** 发布前用校验和验证下载的制品未损坏或被替换；配置变更后用统一 diff 格式记录精确变化，便于评审和回滚。校验文件本身也必须来自可信渠道。

---

## 五、进程、资源与系统服务

### 16. `ps`、`pgrep`、`top`：定位进程和资源热点

```bash
ps -eo pid,ppid,user,%cpu,%mem,etime,stat,cmd --sort=-%cpu | head -11
pgrep -af 'gunicorn|uvicorn'
top -b -n 1 | head -25
```

**作用：**

- `ps`：按 CPU 降序显示进程关键字段。
- `pgrep -af`：按模式查找进程，并显示完整命令行。
- `top -b -n 1`：以批处理模式抓取单次系统快照，便于保存到工单。

**示例输出：**

```text
  PID  PPID USER     %CPU %MEM     ELAPSED STAT CMD
24187     1 myapp     182  3.4    02:14:11 Sl   /srv/myapp/venv/bin/gunicorn config.wsgi
 9182     1 postgres   21  1.8 1-03:42:08 Ss   postgres: app appdb 10.0.1.15(43112) idle

24187 /srv/myapp/venv/bin/gunicorn config.wsgi --workers 4
```

**真实场景：** CPU 告警发生时，先确认高 CPU 的 PID、运行用户、父进程和完整命令，再结合应用日志与部署记录判断是正常批处理、死循环还是流量上升。

### 17. `lsof`：定位端口与已删除但仍被占用的文件

```bash
sudo lsof -nP -iTCP:8080 -sTCP:LISTEN
sudo lsof +L1
```

**作用：**

- 第一条显示监听 TCP 8080 的进程；`-nP` 避免 DNS 和服务名解析造成延迟。
- `+L1` 显示链接数小于 1、即目录项已删除但仍被进程打开的文件。

**示例输出：**

```text
COMMAND   PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
gunicorn 24187 myapp   7u  IPv4  90321      0t0  TCP *:8080 (LISTEN)

COMMAND   PID  USER   FD   TYPE DEVICE SIZE/OFF NLINK NAME
gunicorn 24187 myapp   5w   REG  259,3  12G     0 /var/log/myapp/app.log (deleted)
```

**真实场景：** 日志被删除后磁盘空间没有释放。第二条命令确认是哪个进程持有已删除文件；使用受控的服务重启或日志重新打开流程释放空间，而不是继续删除其他文件。

### 18. `kill`：按升级顺序停止异常进程

```bash
kill -TERM 24187
sleep 5
kill -0 24187 && echo "process still running"
kill -KILL 24187
```

**作用：**

- `TERM`（默认信号）请求进程优雅退出，允许它关闭连接、刷新缓冲区。
- `kill -0` 不发送信号，仅检查进程是否存在且当前用户是否有权限。
- `KILL` 不能被捕获或清理，只应在确认进程卡死且影响已评估后使用。

**示例输出：**

```text
process still running
```

**真实场景：** 失控 worker 消耗 CPU 且不响应优雅停止。先发送 `TERM` 并等待；若仍存在，检查服务管理器是否会自动拉起它、是否正在写数据，再决定是否使用 `KILL`。对于 systemd 管理的服务，优先使用 `systemctl`，不要绕过它直接杀主进程。

### 19. `free`、`vmstat`、`iostat`、`dmesg`：诊断内存和 I/O

```bash
free -h
vmstat 1 5
iostat -xz 1 3
sudo dmesg -T --level=err,warn | tail -50
```

**作用：**

- `free -h`：查看内存、缓存和可用内存；重点看 `available`，不应只看 `free`。
- `vmstat 1 5`：每秒采样五次，观察运行队列、上下文切换、swap 和 I/O 等趋势。
- `iostat -xz`：查看设备 I/O、队列和利用率；常由 `sysstat` 包提供。
- `dmesg`：查看内核信息，如 OOM Killer、磁盘错误和网卡问题。

**示例输出：**

```text
               total        used        free      shared  buff/cache   available
Mem:            15Gi        12Gi       512Mi       210Mi        2Gi        2Gi
Swap:          2.0Gi       1.7Gi       300Mi

procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 4  3 1789940 524288  82512 204800  118  256 8192 16384 6421 9230 45  8  4 43  0
```

**真实场景：** API 延迟变高而 CPU 使用率不高，`vmstat` 的 `wa` 很高、`iostat` 显示设备接近饱和时，问题可能是磁盘 I/O，而不是增加 Web worker 能解决的。若 `dmesg` 出现 OOM 记录，需要定位内存增长源、设置合理的资源上限并评估重启影响。

### 20. `systemctl` 和 `journalctl`：管理 systemd 服务及日志

```bash
sudo systemctl status myapp --no-pager
sudo systemctl is-active myapp
sudo systemctl reload nginx
sudo journalctl -u myapp --since '30 minutes ago' --no-pager
sudo journalctl -u myapp -f
```

**作用：**

- `status`：显示服务状态、最近日志和主进程信息。
- `is-active`：适合自动化健康检查；非 active 时会返回非零退出码。
- `reload`：让支持 reload 的服务重新读取配置，通常比 restart 中断更小；须先验证配置。
- `journalctl -u`：按 systemd unit 过滤日志；`-f` 持续跟踪。

**示例输出：**

```text
● myapp.service - My application
     Active: active (running) since Tue 2026-10-21 09:00:01 UTC; 35min ago
   Main PID: 24187 (gunicorn)

Oct 21 09:31:25 payments-prod-01 myapp[24187]: ERROR upstream timeout request_id=f72c
```

**真实场景：** 修改 Nginx 配置后，先执行 `nginx -t`，再 `systemctl reload nginx`，最后调用健康检查并查看错误日志。对于应用服务，区分 reload 是否被 unit 支持；不支持时不要假设配置已经生效。

---

## 六、网络与远程操作

### 21. `ip` 和 `ss`：检查地址、路由和监听端口

```bash
ip -br addr
ip route
sudo ss -lntp
sudo ss -tanp '( sport = :443 or dport = :443 )'
```

**作用：**

- `ip -br addr`：简洁显示网卡与 IP。
- `ip route`：显示路由和默认网关。
- `ss -lntp`：显示 TCP 监听端口与所属进程；通常比旧的 `netstat` 更常用。
- 最后一条查看与 443 端口相关的连接状态，筛查大量 `SYN-RECV`、`TIME-WAIT` 等情况。

**示例输出：**

```text
lo               UNKNOWN        127.0.0.1/8 ::1/128
ens5             UP             10.20.30.15/24
default via 10.20.30.1 dev ens5

LISTEN 0 4096 0.0.0.0:443 0.0.0.0:* users:(("nginx",pid=1312,fd=6))
```

**真实场景：** 服务启动成功但无法被外部访问时，按“进程是否监听、监听在哪个地址、防火墙/安全组、路由和负载均衡健康检查”逐层验证，不能只检查应用进程存在。

### 22. `curl`：验证 HTTP、TLS 和接口响应

```bash
curl -fsS -o /dev/null -w 'code=%{http_code} total=%{time_total}s\n' \
  https://api.example.com/health

curl -fsS -D - -o /dev/null https://api.example.com/health

curl --connect-timeout 3 --max-time 10 -fsS \
  -H 'Accept: application/json' \
  https://api.example.com/v1/status
```

**作用：**

- `-f`：HTTP 4xx/5xx 返回非零；`-sS`：静默但仍显示错误。
- `-o /dev/null -w`：丢弃响应体，仅输出状态码和总耗时。
- `-D -`：输出响应头。
- 连接和总超时可避免监控脚本无限等待。

**示例输出：**

```text
code=200 total=0.043821s
HTTP/2 200
content-type: application/json
x-request-id: 9f4c7b
```

**真实场景：** 发布后从负载均衡外部、应用宿主机本机和同 VPC 的客户端分别调用健康接口。三处结果不同可以帮助区分 DNS、TLS、网络策略、代理和应用自身问题。

### 23. `dig`、`getent hosts`、`nc`：验证 DNS 和 TCP 可达性

```bash
dig +short api.example.com
getent hosts api.example.com
nc -vz -w 3 db.internal.example.com 5432
```

**作用：**

- `dig`：直接查询 DNS；`+short` 只显示记录值。
- `getent hosts`：按主机实际名称服务配置解析，能体现 `/etc/hosts`、DNS、LDAP/NSS 等差异。
- `nc -vz`：尝试建立 TCP 连接并输出结果；`-w 3` 限制等待时间。不同发行版的 netcat 参数可能不同。

**示例输出：**

```text
198.51.100.24
198.51.100.24 api.example.com
Connection to db.internal.example.com 5432 port [tcp/postgresql] succeeded!
```

**真实场景：** 应用无法连接数据库时，先从应用主机检查名称解析和目标端口可达性，再检查数据库侧监听地址、访问控制、防火墙或云安全组。TCP 成功只证明网络层可达，不证明认证、TLS 或 SQL 权限正确。

### 24. `ssh` 和 `rsync`：受控远程操作与文件同步

```bash
ssh -o BatchMode=yes -o ConnectTimeout=5 ops@web-02 'hostnamectl --static'
rsync -avzn --delete /srv/build/static/ ops@web-02:/srv/myapp/static/
rsync -avz --delete --chown=myapp:myapp /srv/build/static/ ops@web-02:/srv/myapp/static/
```

**作用：**

- `BatchMode=yes`：禁止 SSH 等待交互式密码输入，适合自动化任务；应使用受保护的密钥或工作负载身份。
- `ConnectTimeout`：避免连接卡住。
- `rsync -a`：递归保留常见属性；`-v` 显示过程；`-z` 压缩传输。
- `-n`：演练模式；先检查删除和更新清单再去掉。
- `--chown`：目标端支持时指定文件所有者；需确保远端权限允许。

**示例输出：**

```text
web-02
deleting old/static/admin.css
static/app.9c21.js
static/app.9c21.css
```

**真实场景：** 同步静态文件前一定执行 `--dry-run`，特别是带 `--delete` 时。生产环境建议通过受控部署系统和最小权限账户执行，不要在命令行传递口令。

---

## 七、计划任务、归档与数据传输

### 25. `crontab`：查看和配置定时任务

```bash
crontab -l
crontab -e
```

示例：每天 02:15 执行备份，并记录输出。

```cron
15 2 * * * /usr/local/bin/backup-myapp.sh >> /var/log/myapp/backup.log 2>&1
```

**输入：** 将以上行写入执行该任务用户的 crontab。

**示例输出：**

```text
15 2 * * * /usr/local/bin/backup-myapp.sh >> /var/log/myapp/backup.log 2>&1
```

**真实场景：** 定时备份、清理临时文件、证书检查等。Cron 的环境变量和交互式 shell 不同，应在脚本中使用绝对路径、显式设置必要环境变量和时区，并将输出写入可监控日志。关键任务应由调度平台统一管理、监控成功与失败，不能只依赖“Cron 没报错”。

### 26. `date`：统一时间格式，避免时区歧义

```bash
date -u '+%Y-%m-%dT%H:%M:%SZ'
date '+%F %T %z'
```

**示例输出：**

```text
2026-10-21T09:31:25Z
2026-10-21 17:31:25 +0800
```

**真实场景：** 关联多台服务器、日志平台和第三方 API 的故障时间时使用 UTC ISO 8601 格式。脚本生成备份名时可使用 `date -u '+%Y%m%dT%H%M%SZ'`，避免本地时区、夏令时和不同地区的含义冲突。

### 27. `flock`：防止定时任务重叠执行

```bash
flock -n /var/lock/myapp-backup.lock \
  /usr/local/bin/backup-myapp.sh
```

**示例输出：**

```text
flock: failed to acquire lock: Resource temporarily unavailable
```

**作用：** `-n` 表示锁已存在时立即失败。若备份执行时间超过调度间隔，后一次任务不会与前一次同时运行。

**真实场景：** 每 15 分钟汇总数据的任务偶发执行 30 分钟，重叠运行会重复上传或争抢数据库资源。使用锁保护后，应监控锁获取失败并调查为何前一任务超时，而不是永远静默跳过。

---

## 八、可靠 Shell 脚本模式

### 28. 脚本基线：参数校验、清理与日志

以下脚本演示一个可用于发布后健康检查的最小模式：

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

readonly endpoint="${1:?usage: $0 <health-check-url>}"
readonly timeout_seconds=10
tmp_response="$(mktemp)"

cleanup() {
  rm -f -- "$tmp_response"
}
trap cleanup EXIT

if ! curl --connect-timeout 3 --max-time "$timeout_seconds" \
    -fsS "$endpoint" >"$tmp_response"; then
  printf 'health check failed: %s\n' "$endpoint" >&2
  exit 1
fi

if ! grep -qx 'ok' "$tmp_response"; then
  printf 'unexpected health response from %s: %s\n' \
    "$endpoint" "$(tr '\n' ' ' <"$tmp_response")" >&2
  exit 1
fi

printf 'healthy endpoint=%s checked_at=%s\n' \
  "$endpoint" "$(date -u '+%Y-%m-%dT%H:%M:%SZ')"
```

**输入：**

```bash
./check-health.sh https://api.example.com/health
```

**成功输出：**

```text
healthy endpoint=https://api.example.com/health checked_at=2026-10-21T09:31:25Z
```

**失败输出：**

```text
health check failed: https://api.example.com/health
```

**关键点：**

- `set -Eeo pipefail`：命令或管道失败时尽早终止；但条件判断、预期失败和清理逻辑仍要显式处理，不能把它当作完整错误处理方案。
- `set -u`：引用未定义变量会失败，减少拼写错误造成的危险默认值。
- `IFS=$'\n\t'`：降低按空格意外拆分字符串的风险；遍历文件名时仍应使用 `find -print0` 和 `read -r -d ''`。
- `mktemp` 和 `trap`：创建唯一临时文件，并在退出时清理。
- 始终引用变量，如 `"$endpoint"`；参数校验用 `${1:?usage...}` 给出明确错误。

### 29. 处理包含空格或特殊字符的文件名

```bash
find /srv/uploads -type f -name '*.csv' -print0 |
  while IFS= read -r -d '' file; do
    printf 'processing: %s\n' "$file"
  done
```

**示例输出：**

```text
processing: /srv/uploads/customer 2026-10.csv
processing: /srv/uploads/客户数据.csv
```

**真实场景：** 用户上传的文件名可能带空格、换行或非 ASCII 字符。不要写 `for file in $(find ...)`，它会按空白拆分并可能处理错误的文件。

### 30. 为脚本加入可观测性和幂等性

自动化脚本至少应输出：开始/结束时间、目标、变更结果、失败原因和退出码；但不得输出密钥、访问令牌、完整个人数据。

```bash
log() {
  printf '%s level=%s event=%s\n' \
    "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" "$1" "$2"
}

if [[ -f /srv/myapp/.migrated-20261021 ]]; then
  log INFO "migration_already_applied"
  exit 0
fi

log INFO "migration_started"
# 执行可安全重试的迁移命令
touch /srv/myapp/.migrated-20261021
log INFO "migration_completed"
```

**真实场景：** CI/CD 因网络抖动重试部署步骤时，脚本必须能识别已完成状态，避免重复导入数据、重复创建资源或重复发送通知。标记文件仅适用于单机简单场景；分布式系统需用数据库唯一约束、事务、锁或工作流引擎保证幂等。

---

## 九、真实故障排查场景

### 场景 A：告警显示磁盘空间 95%

**目标：** 找到增长源，安全释放空间，并避免再次发生。

```bash
df -hT /var
du -xhd1 /var | sort -h
sudo lsof +L1
sudo journalctl --disk-usage
```

**判断路径：**

1. `df` 确认满的是哪个挂载点。
2. `du` 定位是日志、缓存、容器数据还是应用数据增长。
3. 若两者数字不一致，`lsof +L1` 排查已删除但仍被占用的日志。
4. 对日志使用已批准的 logrotate/保留策略；对 systemd journal 先查看占用和既定保留配置。
5. 释放后再次运行 `df` 验证容量，并追溯增长原因、补充阈值和容量规划。

**不要做：** 未确认路径就递归删除 `/var` 下文件；删除正在写的日志却不让服务重新打开文件；将磁盘告警简单静音。

### 场景 B：发布后接口返回 502

**目标：** 区分反向代理、应用进程、端口监听和下游依赖问题。

```bash
sudo nginx -t
sudo systemctl status nginx myapp --no-pager
sudo ss -lntp | grep ':8080'
curl -fsS -D - http://127.0.0.1:8080/health
sudo journalctl -u nginx -u myapp --since '15 minutes ago' --no-pager
```

**示例输出：**

```text
nginx: configuration file /etc/nginx/nginx.conf test is successful
LISTEN 0 4096 127.0.0.1:8080 0.0.0.0:* users:(("gunicorn",pid=24187,fd=7))
HTTP/1.1 200 OK
```

**判断路径：**

- Nginx 配置失败：回滚或修复配置，不要重载错误配置。
- 应用未监听：检查服务状态、启动日志、环境变量和最近部署。
- 本地健康检查正常但外部失败：检查 Nginx upstream、TLS、负载均衡和网络策略。
- 本地也失败：用日志中的请求 ID 检查应用、数据库、缓存或下游 API。

### 场景 C：数据库连接超时

**目标：** 按 DNS、网络、数据库监听、连接池和应用配置分层排查。

```bash
getent hosts db.internal.example.com
nc -vz -w 3 db.internal.example.com 5432
env | grep -E '^(DATABASE|DB_)'
sudo ss -tanp | grep ':5432'
```

**示例输出：**

```text
10.20.40.12 db.internal.example.com
Connection to db.internal.example.com 5432 port [tcp/postgresql] succeeded!
DATABASE_URL=postgresql://...
```

**判断路径：**

- DNS 失败：检查名称服务、私有 DNS 和 `/etc/resolv.conf` 管理方式。
- TCP 失败：检查数据库监听地址、防火墙、安全组、网络路由和维护事件。
- TCP 成功但应用失败：检查 TLS、认证、数据库最大连接数、应用连接池耗尽及慢事务。
- 避免把完整 `DATABASE_URL` 记录到共享日志；输出时应脱敏。

### 场景 D：某个定时任务重复执行

**目标：** 确认调度源，防止任务重叠，并恢复正确状态。

```bash
crontab -l
systemctl list-timers --all
pgrep -af 'daily-reconciliation'
sudo journalctl -u reconciliation.service --since today --no-pager
```

**判断路径：**

1. 确认任务是否被 Cron、systemd timer、CI/CD 或多个节点重复调度。
2. 确认旧实例是否仍运行、是否有超时或锁失效。
3. 为单机脚本加 `flock`；多节点任务使用分布式锁、数据库唯一约束或调度平台的单实例语义。
4. 对已经重复产生的数据进行审计和幂等修复，而不是只停止进程。

---

## 十、高频命令速查

| 目标 | 首选命令 | 典型用途 |
|---|---|---|
| 确认环境 | `hostnamectl --static`、`id`、`pwd` | 防止误操作主机和目录 |
| 查文件 | `find`、`stat`、`ls -lah` | 按时间、大小、权限定位文件 |
| 查磁盘 | `df -hT`、`du -xhd1` | 区分挂载点满和目录膨胀 |
| 看实时日志 | `tail -F`、`journalctl -f` | 发布和故障期间追踪新日志 |
| 过滤/聚合日志 | `grep`、`awk`、`sort`、`uniq`、`jq` | 定位错误、统计接口、读取 JSON |
| 查进程 | `ps`、`pgrep -af`、`top` | 找到资源热点和完整启动命令 |
| 查端口/删除文件 | `ss -lntp`、`lsof` | 排查监听冲突、磁盘未释放 |
| 管服务 | `systemctl`、`journalctl -u` | 验证、重载、观察 systemd 服务 |
| 查资源瓶颈 | `free -h`、`vmstat`、`iostat`、`dmesg` | 内存、swap、I/O、内核错误 |
| 查网络 | `ip route`、`curl`、`dig`、`nc` | 路由、HTTP、DNS、TCP 分层诊断 |
| 远程同步 | `ssh`、`rsync --dry-run` | 受控远程巡检和文件发布 |
| 定时与互斥 | `crontab`、`systemctl list-timers`、`flock` | 管理任务并避免重复执行 |

最后的原则是：**命令只是证据收集和受控变更工具。** 在生产环境中，把每一次操作与明确的假设、可验证结果、变更记录、监控和回滚方案结合起来，才是可靠运维。
