# 从零搭建 NetBox 系统

本文以 Ubuntu 24.04 LTS、Docker Compose、Nginx 和 Let's Encrypt HTTPS 为例，部署一套单机 NetBox。Docker Compose 会运行 NetBox、PostgreSQL、Valkey（Redis 兼容服务）和后台 Worker，适合作为生产起点。

> 将文中的 `netbox.example.com`、`admin@example.com` 和 `/opt/netbox-docker` 替换成实际值。不要将密码、`SECRET_KEY` 或 API Token 提交到 Git 仓库。

## 1. 规划与前置条件

1. 准备 Ubuntu 24.04 LTS 服务器。小规模使用至少配置 2 vCPU、4 GB 内存、40 GB 磁盘；数据量增加后应扩容并监控数据库容量。
2. 为服务器配置静态 IP 和 DNS `A` 或 `AAAA` 记录，例如 `netbox.example.com`。
3. 确保入口可访问 TCP 80 和 443。Let's Encrypt 需要通过 80 端口完成域名验证。
4. 使用有 `sudo` 权限的普通账户操作，不长期用 `root` 直接运维。
5. 预先定义站点、设备、机柜、VLAN、VRF、前缀和 IP 地址的命名规范、状态和角色。

NetBox 是网络资源的权威数据源，不是网络设备配置下发平台。先统一数据规范，后续自动化和审计才可靠。

## 2. 更新系统并安装基础工具

```bash
sudo apt update
sudo apt -y upgrade
sudo apt install -y ca-certificates curl gnupg git openssl ufw
sudo reboot
```

重新登录后检查系统与磁盘空间：

```bash
lsb_release -a
df -h
```

## 3. 安装 Docker Engine 与 Compose

以下命令使用 Docker 官方 APT 仓库安装 Docker Engine 和 Compose 插件：

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

让当前用户执行 Docker，然后退出并重新登录以使组权限生效：

```bash
sudo usermod -aG docker "$USER"
exit
```

重新登录后验证安装：

```bash
docker --version
docker compose version
docker run --rm hello-world
```

## 4. 配置主机防火墙

先放行 SSH，再只对外开放 Web 端口：

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
sudo ufw status verbose
```

不要对外开放 NetBox 容器端口 8000。后面会使其仅监听本机回环地址，再由 Nginx 提供 HTTPS。

## 5. 获取并固定 NetBox Docker 版本

官方 `netbox-docker` 仓库维护 Compose 定义及镜像。生产环境应固定已验证的 Git 标签和相应镜像版本，不能使用会漂移的 `latest` 标签。

```bash
sudo mkdir -p /opt
sudo chown "$USER":"$USER" /opt
git clone --branch release --single-branch https://github.com/netbox-community/netbox-docker.git /opt/netbox-docker
cd /opt/netbox-docker
git fetch --tags
git tag --sort=-version:refname | head -n 10
```

从列表中选择稳定版本。以下 `5.1.0` 仅作示例，实际应选择已阅读发行说明并验证过的版本：

```bash
git checkout 5.1.0
git describe --tags --always
```

记录该标签以及镜像版本。`docker-compose.yml` 中的默认 `VERSION` 应与当前 `netbox-docker` 版本保持匹配。

## 6. 替换默认密钥和密码

示例环境文件中的密码不能用于生产。先收紧权限：

```bash
chmod 700 /opt/netbox-docker
chmod 600 /opt/netbox-docker/env/*.env
cd /opt/netbox-docker
```

为每项秘密生成不同随机值，每次命令都会输出一个新值。将输出保存在受控密码管理器中：

```bash
openssl rand -base64 48
```

使用编辑器修改下列文件：

| 文件 | 必须设置的变量 | 要求 |
| --- | --- | --- |
| `env/postgres.env` | `POSTGRES_PASSWORD` | PostgreSQL 密码 |
| `env/netbox.env` | `DB_PASSWORD` | 必须与 `POSTGRES_PASSWORD` 一致 |
| `env/redis.env` | `REDIS_PASSWORD` | 任务队列 Valkey 密码 |
| `env/redis-cache.env` | `REDIS_PASSWORD` | 缓存 Valkey 密码 |
| `env/netbox.env` | `REDIS_PASSWORD`、`REDIS_CACHE_PASSWORD` | 分别对应两个 Valkey 密码 |
| `env/netbox.env` | `SECRET_KEY` | 至少 50 个字符的随机字符串，必须长期保存 |
| `env/netbox.env` | `API_TOKEN_PEPPER_1` | 保护 API Token 的随机字符串，必须长期保存 |

在 `env/netbox.env` 中同时建议设为：

```dotenv
TIME_ZONE=Asia/Shanghai
LOGIN_REQUIRED=true
CORS_ORIGIN_ALLOW_ALL=false
```

不要删除或随意变更 `SECRET_KEY` 与 `API_TOKEN_PEPPER_1`，否则会影响会话和已生成的 Token。成熟环境应将这些值迁移到 Docker secrets 或外部密钥管理系统。

## 7. 配置域名和本地监听端口

复制官方覆盖文件：

```bash
cd /opt/netbox-docker
cp docker-compose.override.yml.example docker-compose.override.yml
```

编辑 `docker-compose.override.yml`，确保 `netbox` 服务至少包含以下配置。端口必须绑定在 `127.0.0.1`，防止绕开 Nginx 和 HTTPS：

```yaml
services:
  netbox:
    ports:
      - "127.0.0.1:8000:8080"
    environment:
      ALLOWED_HOSTS: "netbox.example.com"
      CSRF_TRUSTED_ORIGINS: "https://netbox.example.com"
      TIME_ZONE: "Asia/Shanghai"
      LOGIN_REQUIRED: "true"
      CORS_ORIGIN_ALLOW_ALL: "false"
```

有多个域名时，`ALLOWED_HOSTS` 用空格分隔；`CSRF_TRUSTED_ORIGINS` 使用空格分隔完整的 `https://` URL。预检最终 Compose 配置：

```bash
docker compose config
docker compose config | grep 8000
```

输出中应存在 `127.0.0.1:8000:8080`，不能出现 `0.0.0.0:8000`。

## 8. 启动服务并创建管理员

首次下载镜像和初始化数据库需要几分钟：

```bash
cd /opt/netbox-docker
docker compose pull
docker compose up -d
docker compose ps
```

等待 `netbox` 变为 `healthy`。初始化较慢时查看日志：

```bash
docker compose logs -f netbox
```

按 `Ctrl+C` 退出日志查看但不会停止容器。随后交互式创建第一个管理员，密码不会写入 Shell 历史：

```bash
docker compose exec netbox /opt/netbox/netbox/manage.py createsuperuser
```

从服务器本机验证登录页响应：

```bash
curl -I http://127.0.0.1:8000/login/
```

预期结果为 `200 OK` 或重定向。若服务未就绪，运行 `docker compose ps` 与 `docker compose logs --tail=100 netbox` 排查数据库、Valkey、磁盘和密码配置。

## 9. 配置 Nginx 和 HTTPS

安装 Nginx 与 Certbot：

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
sudo rm -f /etc/nginx/sites-enabled/default
```

创建 `/etc/nginx/sites-available/netbox` 并替换域名：

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name netbox.example.com;

    client_max_body_size 25m;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }
}
```

启用站点并验证 HTTP 代理：

```bash
sudo ln -s /etc/nginx/sites-available/netbox /etc/nginx/sites-enabled/netbox
sudo nginx -t
sudo systemctl enable --now nginx
curl -I http://netbox.example.com/login/
```

确认 DNS 指向当前主机、TCP 80 能从外网访问后，申请并强制使用 HTTPS：

```bash
sudo certbot --nginx -d netbox.example.com --redirect --agree-tos -m admin@example.com
sudo systemctl status certbot.timer
sudo certbot renew --dry-run
```

浏览器访问 `https://netbox.example.com/`，使用第 8 节创建的管理员登录。HTTPS 确认长期可用后，向 `docker-compose.override.yml` 的 `environment` 中补充以下设置并重启：

```yaml
      SECURE_SSL_REDIRECT: "true"
      SECURE_HSTS_SECONDS: "31536000"
```

```bash
cd /opt/netbox-docker
docker compose up -d
```

HSTS 会要求浏览器持续使用 HTTPS，因此不要在仍需 HTTP 的测试环境启用。

## 10. 首次录入网络资源

登录系统后按此顺序建立基础数据：

1. 在“组织”中创建租户组、租户、区域、站点组和站点。
2. 在“DCIM”中创建厂商、设备类型、设备角色、平台、机柜和设备。
3. 在“IPAM”中创建 RIR、VRF、VLAN、前缀和 IP 地址；是否使用 VRF 和全局唯一地址空间应按网络设计统一决定。
4. 为设备添加接口和 IP 地址，设置设备的主要 IPv4 或 IPv6 地址。
5. 记录接口间的电缆链路；有虚拟化环境时，补充集群、虚拟机和虚拟接口。
6. 创建普通用户组与最小权限，日常管理不要使用超级管理员。
7. 为自动化系统创建专用服务账号与最小权限 API Token，禁止共享管理员 Token。

先手工输入少量代表性数据验证模型，再使用 CSV、REST API 或 Ansible 批量导入。批量导入应先在测试环境试运行并保留源文件。

## 11. 日常检查

常用命令：

```bash
cd /opt/netbox-docker
docker compose ps
docker compose logs --tail=100 netbox
docker compose logs --tail=100 netbox-worker
docker compose exec netbox /opt/netbox/netbox/manage.py check
```

定期检查磁盘、容器健康状态、证书续期、数据库备份和 API 错误。需要 Prometheus 监控时，确认认证和网络边界后启用 `METRICS_ENABLED=true`，并仅向可信网络暴露 `/metrics`。

## 12. 备份与恢复演练

必须备份 PostgreSQL 数据库、媒体附件、脚本、报表和配置文件。只备份 Compose 文件无法恢复 NetBox 数据。

创建仅管理员可读的备份目录：

```bash
sudo install -d -m 0700 -o "$USER" -g "$USER" /var/backups/netbox
cd /opt/netbox-docker
```

导出数据库，并将备份复制到另一台服务器或对象存储：

```bash
docker compose exec -T postgres pg_dump -U netbox netbox | gzip > "/var/backups/netbox/postgres-$(date +%F-%H%M%S).sql.gz"
```

导出应用文件和配置：

```bash
backup_time=$(date +%F-%H%M%S)
mkdir -p "/var/backups/netbox/$backup_time"
docker compose cp netbox:/opt/netbox/netbox/media "/var/backups/netbox/$backup_time/media"
docker compose cp netbox:/opt/netbox/netbox/scripts "/var/backups/netbox/$backup_time/scripts"
docker compose cp netbox:/opt/netbox/netbox/reports "/var/backups/netbox/$backup_time/reports"
cp -a configuration env docker-compose.yml docker-compose.override.yml "/var/backups/netbox/$backup_time/"
```

至少每季度在隔离测试主机恢复一次。恢复时应先部署**相同版本**的 `netbox-docker`，向空数据库导入 SQL，再恢复媒体、脚本、报表和配置，最后启动服务并验证登录、设备查询、IP 地址和附件。不要在未备份的生产数据库直接执行恢复。

## 13. 升级步骤

升级前阅读目标 NetBox 和 `netbox-docker` 的发行说明，检查破坏性变更、数据库迁移和插件兼容性。先在测试环境用生产备份演练：

```bash
cd /opt/netbox-docker

# 1. 完成第 12 节备份，并确认备份可读取
# 2. 在测试环境验证目标版本
git fetch --tags
git checkout <目标-netbox-docker-标签>
docker compose pull
docker compose up -d
docker compose ps
docker compose exec netbox /opt/netbox/netbox/manage.py check
```

首次跨大版本启动可能较慢。用 `docker compose logs -f netbox` 观察迁移完成，不要在迁移中强制删除容器或卷。升级后验证 Web 登录、API、后台 Worker 和关键自动化任务。

## 14. 常见故障

| 现象 | 优先检查 | 常见处理 |
| --- | --- | --- |
| `DisallowedHost` 或 HTTP 400 | `ALLOWED_HOSTS` | 添加实际域名后执行 `docker compose up -d` |
| HTTPS 提交失败或 CSRF 错误 | `CSRF_TRUSTED_ORIGINS` 与 `X-Forwarded-Proto` | 填完整 `https://域名`，确认 Nginx 配置已加载 |
| `netbox` 一直未健康 | `docker compose logs --tail=200 netbox` | 核对数据库与 Valkey 密码一致性，检查磁盘和内存 |
| 后台任务不执行 | `netbox-worker` 日志 | 确认 Worker 和 Valkey 正常运行 |
| 证书申请失败 | DNS 与 TCP 80 连通性 | 确认域名解析和公网访问，排除端口冲突 |

## 15. 上线检查清单

- [ ] 已固定并记录 `netbox-docker` Git 标签和镜像版本。
- [ ] 已替换示例密码、`SECRET_KEY` 与 `API_TOKEN_PEPPER_1`，并安全保存。
- [ ] 容器仅监听 `127.0.0.1:8000`，公网只开放 80 和 443。
- [ ] 已配置 HTTPS、证书续期、`ALLOWED_HOSTS` 和 `CSRF_TRUSTED_ORIGINS`。
- [ ] 已建立普通运维账号、最小权限用户组和专用自动化账号。
- [ ] 已完成数据库、附件、脚本、报表和配置的异机备份。
- [ ] 已在隔离环境完成恢复演练。
- [ ] 已建立容器、容量、证书和备份失败的监控告警。

## 参考资料

- [NetBox 官方安装文档](https://netboxlabs.com/docs/netbox/en/stable/installation/)
- [netbox-docker 官方仓库](https://github.com/netbox-community/netbox-docker)
- [netbox-docker Wiki](https://github.com/netbox-community/netbox-docker/wiki)
- [NetBox 官方升级文档](https://netboxlabs.com/docs/netbox/en/stable/installation/upgrading/)
