# ops-os-init

> 基于 Ansible 的 Linux 系统初始化项目
> 首个支持系统：**Rocky Linux 9**

---

## 📌 项目背景

在新主机（物理机 / 云主机 / 虚拟机）交付后，通常需要进行一系列**基础初始化操作**，例如：

- 内核与系统参数调优
- 禁用不必要或不安全的系统服务
- 创建必要的运维用户
- 统一基础系统行为与配置

本项目通过 **Ansible 自动化** 将这些初始化步骤标准化、可重复执行，减少人工配置差异，为后续运维、部署、安全加固打下基础。

---

## 🎯 项目目标

- 提供 **Linux 系统初始化（Init）** 的自动化实现
- 当前聚焦 **Rocky Linux 9**（CentOS 9 兼容）
- 所有操作遵循：幂等（Idempotent）、可重复执行、不破坏系统默认安全模型
- 为未来演进为 **Linux Baseline / Hardening** 项目预留结构

---

## 🚀 快速开始

### 前置条件

- 控制机已安装 Ansible >= 2.14
- 目标主机为 Rocky Linux 9 / CentOS 9
- 控制机可通过 SSH 连接目标主机

### 第一步：配置主机清单

编辑 `inventories/rocky9.yml`，取消注释并填入实际主机地址：

```yaml
# root + SSH Key（推荐）
db:
  hosts:
    db1:
      ansible_host: 203.0.113.20
      ansible_user: root
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519

# 普通用户 + sudo
cdn:
  hosts:
    cdn1:
      ansible_host: 203.0.113.30
      ansible_user: ops
      ansible_become_password: "your_sudo_password"
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519
```

支持 4 种连接方式：root 密码、root SSH Key、普通用户 + sudo、非标准端口。详见文件内注释。

### 第二步：替换 SSH 公钥

```bash
# 将真实公钥覆盖占位符文件
cp ~/.ssh/id_ed25519.pub roles/init-users/files/ops.pub
cp ~/.ssh/id_ed25519_admin.pub roles/init-users/files/admin.pub
```

### 第三步：自定义变量（可选）

所有可配置变量集中在 `group_vars/rocky9.yml`，按需修改：

- 时区、DNS、NTP
- sysctl 参数
- 用户列表、SSH 端口
- 安装/移除的包列表
- Shell Alias

### 第四步：执行

```bash
# 语法检查
ansible-playbook playbooks/init.yml --syntax-check

# 干跑（不做任何实际改动，仅预览）
ansible-playbook playbooks/init.yml -i inventories/rocky9.yml --check

# 正式执行全部初始化
ansible-playbook playbooks/init.yml -i inventories/rocky9.yml

# 只执行某个 Role
ansible-playbook playbooks/init.yml -i inventories/rocky9.yml --tags init-ssh

# 组合执行多个 Role
ansible-playbook playbooks/init.yml -i inventories/rocky9.yml --tags users,ssh

# 只对某个分组执行
ansible-playbook playbooks/init.yml -i inventories/rocky9.yml --limit db
```

### 第五步：验证

执行完成后，Playbook 会输出初始化摘要。请务必验证 SSH 连接：

```bash
# 验证普通用户登录
ssh -p 22 ops@<host>

# 验证应急端口（root 可登录）
ssh -p 22222 root@<host>
```

---

## 📂 项目结构

```text
ops-os-init/
├── ansible.cfg                              # Ansible 全局配置
├── .ansible-lint                            # Lint 规则
├── inventories/
│   └── rocky9.yml                           # 主机清单（支持分组）
├── group_vars/
│   ├── rocky9.yml                           # 公共变量（所有节点继承）
│   ├── db.yml                               # 数据库节点覆盖参数
│   ├── cdn.yml                              # CDN / 反向代理节点覆盖参数
│   └── docker.yml                           # 容器宿主机覆盖参数
├── playbooks/
│   └── init.yml                             # 主 Playbook
└── roles/
    ├── init-facts/                          # 基础系统配置
    ├── init-kernel/                         # 内核资源限制
    ├── init-sysctl/                         # 系统控制参数调优
    ├── init-users/                          # 用户与权限管理
    ├── init-ssh/                            # SSH 安全加固
    ├── init-services/                       # 服务管理
    └── init-packages/                       # 包管理 + Docker 安装
```

---

## 🧱 Role 说明

### init-facts — 基础系统配置

| 操作 | 说明 |
|---|---|
| 主机名 | 可选，留空跳过 |
| 时区 | 默认 `Asia/Shanghai` |
| Locale | 默认 `en_US.UTF-8` |
| DNS | Google `8.8.8.8` + Cloudflare `1.1.1.1` |
| NTP | Cloudflare + Google + pool.ntp.org |
| rsyslog / logrotate | 确保安装并运行 |
| Shell Alias | 写入 `/etc/profile.d/ops-alias.sh`，全局生效 |

### init-kernel — 内核资源限制

写入 `/etc/security/limits.d/99-ops-init.conf`：

- 文件句柄 `nofile`: 655360
- 进程数 `nproc`: 131072
- 内存锁 `memlock`: unlimited
- Core dump: 禁用

### init-sysctl — 系统控制参数

写入 `/etc/sysctl.d/99-ops-init.conf` 并立即加载：

- TCP 优化（fin_timeout、tw_reuse、syncookies、缓冲区等）
- 网络核心（somaxconn 32768、netdev_max_backlog 16384）
- IPv4 安全（禁用 ICMP redirect、source route）
- IPv6 默认禁用
- 文件系统（file-max 2M、inotify 524288）
- 虚拟内存（swappiness 10）

### init-users — 用户与权限

- 创建 `ops`（sudo 免密码）和 `admin`（sudo 需密码）
- 部署 SSH 公钥
- 锁定无用系统账户（lp、sync、halt、games 等）

### init-ssh — SSH 安全加固

**主 SSH（端口 22）**：禁止 root 登录、禁止密码认证、MaxAuthTries 3

**应急 SSH（端口 22222）**：独立 sshd 实例，允许 root 登录和密码认证，用于回滚

### init-services — 服务管理

- 禁用：cups、avahi-daemon、bluetooth、kdump、rpcbind 等
- 启用：chronyd、rsyslog、sshd、crond、NetworkManager

### init-packages — 包管理

- 安装 26 个运维工具（vim、curl、htop、tmux、jq、tcpdump 等）
- 移除 cockpit、PackageKit
- **Docker 分组**：自动添加 Docker 官方仓库并安装 Docker CE + Compose 插件

---

## 🏷️ 主机分组与变量覆盖

主机按角色分组，每组有独立的参数覆盖文件：

```
rocky9（父组 — group_vars/rocky9.yml）
├── db（数据库 — group_vars/db.yml）
├── cdn（CDN/代理 — group_vars/cdn.yml）
├── docker（容器宿主机 — group_vars/docker.yml）
└── general（通用节点）
```

**变量优先级**：主机变量 > 子组变量 > 父组变量(rocky9) > role defaults

### 各分组关键差异

| 参数 | 通用 | db | cdn | docker |
|---|---|---|---|---|
| `vm.swappiness` | 10 | **1** | 10 | 10 |
| `vm.dirty_ratio` | 20 | **40** | 20 | 20 |
| `core dump` | 禁用 | **开启** | 禁用 | 禁用 |
| `ip_forward` | 0 | 0 | 0 | **1** |
| `disable_ipv6` | 1 | 1 | 1 | **0** |
| `somaxconn` | 32768 | 32768 | **65535** | 32768 |
| `nofile` | 655360 | 655360 | **1048576** | **1048576** |
| `AllowTcpForwarding` | no | **yes** | no | **yes** |
| Docker 安装 | 否 | 否 | 否 | **是** |

---

## 🔖 可用 Tags

| Tag | 说明 |
|---|---|
| `init-facts` / `facts` | 基础系统配置 |
| `init-kernel` / `kernel` | 内核资源限制 |
| `init-sysctl` / `sysctl` | 系统控制参数 |
| `init-users` / `users` | 用户与权限 |
| `init-ssh` / `ssh` | SSH 加固 |
| `init-services` / `services` | 服务管理 |
| `init-packages` / `packages` | 包管理 |

---

## ⚠️ 注意事项

1. **SSH 公钥必须替换**：`roles/init-users/files/` 下的 `.pub` 文件是占位符，不替换会导致无法通过端口 22 登录
2. **应急端口 22222**：SSH 加固后的回滚通道，允许 root + 密码登录
3. **IPv6 默认禁用**：如有 IPv6 需求，修改 `ops_sysctl_params` 中的 `disable_ipv6` 为 0
4. **容器场景**：必须将主机放入 `docker` 分组，否则 `ip_forward` 为 0 会导致容器网络不通
5. **limits 生效**：内核资源限制需要用户重新登录后生效
6. **resolv.conf 覆盖**：DNS 配置会全量覆盖 `/etc/resolv.conf`

---

## 🚫 当前明确不包含的内容

- ❌ 业务服务部署（Nginx / MySQL / Redis 等）
- ❌ 防火墙复杂策略
- ❌ 深度安全加固（CIS 全量、复杂 auditd）
- ❌ 应用层配置
- ❌ Kubernetes 编排

---

## 🧭 支持的系统

| 系统 | 状态 |
|---|---|
| Rocky Linux 9 / CentOS 9 | ✅ 已支持 |
| Ubuntu / Debian | ⏳ 规划中 |
