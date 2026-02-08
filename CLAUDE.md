# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ansible-based Linux system initialization automation project (基于 Ansible 的 Linux 系统初始化项目). Supported operating systems: **Rocky Linux 9** and **Ubuntu 22.04**.

The goal is to standardize post-provisioning initialization of new hosts (physical, cloud, or virtual) through idempotent, repeatable Ansible automation. This project is scoped strictly to **system initialization** — not application deployment, deep security hardening, or container orchestration.

## Architecture

Role-based Ansible architecture. Each OS has its own playbook and inventory, but all share the same set of roles. Roles use `ansible_os_family` conditionals (`RedHat` / `Debian`) to handle distribution-specific differences.

```
Playbook (playbooks/init-rocky9.yml | playbooks/init-ubuntu22.yml)
    → Inventory (inventory/rocky9/ | inventory/ubuntu22/) + Group Vars
    → Roles (sequential, shared across OS):
        1.  init-facts      — gather/validate system information, locale, NTP, DNS
        2.  init-kernel     — kernel parameter tuning (limits.d)
        3.  init-sysctl     — system control parameters (TCP, file handles, etc.)
        4.  init-users      — ops user creation, sudo, PAM policies
        5.  init-ssh        — disable root login, disable password auth, security params
        6.  init-services   — disable unnecessary services, maintain minimal service set
        7.  init-packages   — install ops tools, remove high-risk default packages
        8.  init-audit      — auditd configuration and audit rules
        9.  init-cron       — cron/at access control
        10. init-fileperm   — critical file permission hardening
```

OS detection uses `ansible_os_family` (`RedHat` or `Debian`) to differentiate distributions via `when` conditionals in tasks.

## Project Structure

```
ops-os-init/
├── ansible.cfg                          # Ansible 全局配置
├── .ansible-lint                        # Lint 规则配置
├── inventory/
│   ├── rocky9/                          # Rocky 9 主机清单
│   │   ├── hosts.yml                    # 所有主机（general/db/cdn/docker 分组）
│   │   └── group_vars/
│   │       ├── rocky9.yml              # 父组全局变量 + 连接配置
│   │       ├── cdn.yml                 # CDN 子组覆盖变量
│   │       ├── db.yml                  # 数据库子组覆盖变量
│   │       └── docker.yml             # 容器子组覆盖变量
│   └── ubuntu22/                        # Ubuntu 22.04 主机清单
│       ├── hosts.yml                    # 所有主机
│       └── group_vars/
│           └── ubuntu22.yml            # 父组全局变量 + 连接配置
├── playbooks/
│   ├── init-rocky9.yml                  # Rocky 9 主 Playbook
│   └── init-ubuntu22.yml               # Ubuntu 22.04 主 Playbook
└── roles/
    ├── init-facts/                      # 基础系统配置
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/
    │       ├── chrony.conf.j2
    │       ├── resolv.conf.j2
    │       └── ops-alias.sh.j2
    ├── init-kernel/                     # 内核资源限制
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   └── templates/
    │       └── limits.d-99-ops-init.conf.j2
    ├── init-sysctl/                     # 系统控制参数调优
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   └── templates/
    │       └── sysctl.d-99-ops-init.conf.j2
    ├── init-users/                      # 用户与权限管理
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   ├── files/
    │   │   └── op.pub
    │   └── templates/
    │       ├── pwquality.conf.j2
    │       ├── faillock.conf.j2
    │       └── profile.d-99-ops-security.sh.j2
    ├── init-ssh/                        # SSH 安全加固
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/
    │       ├── sshd_config.d-00-ops-init.conf.j2
    │       ├── sshd_config.d-99-ops-emergency.conf.j2
    │       └── issue.net.j2
    ├── init-services/                   # 服务管理
    │   ├── defaults/main.yml
    │   └── tasks/main.yml
    ├── init-packages/                   # 包管理
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/
    │       ├── rocky.repo.j2
    │       ├── epel.repo.j2
    │       └── sources.list.j2          # Ubuntu APT 源模板
    ├── init-audit/                      # 日志审计
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/
    │       └── audit.rules.d-99-ops-init.rules.j2
    ├── init-cron/                       # 定时任务安全
    │   ├── defaults/main.yml
    │   └── tasks/main.yml
    └── init-fileperm/                   # 关键文件权限
        ├── defaults/main.yml
        └── tasks/main.yml
```

## Variable Management

All configurable variables are centralized in group_vars files per OS:
- Rocky 9: `inventory/rocky9/group_vars/rocky9.yml`
- Ubuntu 22: `inventory/ubuntu22/group_vars/ubuntu22.yml`

Each role also has `defaults/main.yml` as fallback. The group_vars file takes precedence and serves as the single source of truth for customization.

Ansible connection variables (`ansible_user`, `ansible_port`, `ansible_ssh_private_key_file`, etc.) are also managed in group_vars, not in inventory files. Inventory files only define host lists; individual hosts can override connection vars at host level if needed.

Key variable prefixes:
- `ops_hostname`, `ops_timezone`, `ops_locale`, `ops_dns_*`, `ops_ntp_*` — init-facts
- `ops_limits` — init-kernel
- `ops_sysctl_params` — init-sysctl
- `ops_users`, `ops_users_lock` — init-users
- `ops_ssh_*`, `ops_ssh_emergency_*` — init-ssh
- `ops_services_disable`, `ops_services_enable` — init-services
- `ops_packages_install`, `ops_packages_remove`, `ops_rocky_mirror`, `ops_epel_mirror`, `ops_ubuntu_mirror` — init-packages
- `ops_audit_*` — init-audit
- `ops_cron_allow_users`, `ops_cron_at_allow_users` — init-cron
- `ops_file_permissions` — init-fileperm

### Key OS Differences in Variables

| Variable | Rocky 9 | Ubuntu 22 |
|----------|---------|-----------|
| `ops_users[].groups` | `wheel` | `sudo` |
| `ops_services_enable` | `chronyd`, `sshd`, `crond` | `chrony`, `ssh`, `cron` |
| `ops_packages_install` | `vim-enhanced`, `bind-utils`, `procps-ng`, `yum-utils` | `vim`, `dnsutils`, `procps`, `apt-utils` |
| `ops_packages_remove` | `cockpit*`, `PackageKit*` | `snapd`, `popularity-contest`, `ubuntu-advantage-tools` |
| `ops_file_permissions` (GRUB) | `/boot/grub2/grub.cfg` | `/boot/grub/grub.cfg` |

## Common Commands

```bash
# --- Rocky Linux 9 ---
ansible-playbook playbooks/init-rocky9.yml -i inventory/rocky9

# Dry run (check mode)
ansible-playbook playbooks/init-rocky9.yml -i inventory/rocky9 --check

# Run specific role by tag
ansible-playbook playbooks/init-rocky9.yml -i inventory/rocky9 --tags init-facts
ansible-playbook playbooks/init-rocky9.yml -i inventory/rocky9 --tags ssh,users

# --- Ubuntu 22.04 ---
ansible-playbook playbooks/init-ubuntu22.yml -i inventory/ubuntu22

# Dry run (check mode)
ansible-playbook playbooks/init-ubuntu22.yml -i inventory/ubuntu22 --check

# Run specific role by tag
ansible-playbook playbooks/init-ubuntu22.yml -i inventory/ubuntu22 --tags init-facts
ansible-playbook playbooks/init-ubuntu22.yml -i inventory/ubuntu22 --tags ssh,users

# --- Common ---
# Available tags: init-facts, init-kernel, init-sysctl, init-users, init-ssh, init-services, init-packages, init-audit, init-cron, init-fileperm
# Short tags also available: facts, kernel, sysctl, users, ssh, services, packages, audit, cron, fileperm

# Syntax check
ansible-playbook playbooks/init-rocky9.yml --syntax-check
ansible-playbook playbooks/init-ubuntu22.yml --syntax-check

# Lint
ansible-lint
```

### Emergency SSH Access

SSH hardening disables root login and password auth on port 22. An emergency sshd instance runs on port **22222** with root login and password auth enabled for rollback:

```bash
ssh -p 22222 root@<host>
```

## Key Design Principles

- **Idempotent**: all tasks must be safe to run repeatedly
- **Non-destructive**: never break the system's default security model; no aggressive kernel changes
- **Emergency rollback**: SSH hardening must preserve fallback access capability
- **Clear scope boundary**: initialization only — no business services, no complex firewall policies, no CIS-full hardening, no container/K8s content
- **Multi-OS support**: roles use `ansible_os_family` conditionals to handle RedHat/Debian differences; OS-specific variables are managed in separate group_vars files
- **Future-ready**: structure should accommodate evolution into a Linux Baseline / Hardening project

## Documentation Language

User-facing documentation is written in **Simplified Chinese**. Ansible task names and code comments may use English.
