# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ansible-based Linux system initialization automation project (基于 Ansible 的 Linux 系统初始化项目). Primary target is **Rocky Linux 9**, with Ubuntu/Debian planned for the future.

The goal is to standardize post-provisioning initialization of new hosts (physical, cloud, or virtual) through idempotent, repeatable Ansible automation. This project is scoped strictly to **system initialization** — not application deployment, deep security hardening, or container orchestration.

## Architecture

Role-based Ansible architecture. The main playbook (`playbooks/init.yml`) orchestrates roles executed in sequence against hosts defined in inventory files.

```
Playbook (playbooks/init.yml)
    → Inventory (inventories/rocky9.yml) + Group Vars (group_vars/rocky9.yml)
    → Roles (sequential):
        1. init-facts      — gather/validate system information via ansible_facts
        2. init-kernel     — kernel parameter tuning
        3. init-sysctl     — system control parameters (TCP, file handles, etc.)
        4. init-users      — ops user creation, sudo, shell/home permissions
        5. init-ssh        — disable root login, disable password auth, security params
        6. init-services   — disable unnecessary services, maintain minimal service set
        7. init-packages   — install ops tools, remove high-risk default packages
```

OS detection uses `ansible_facts` to differentiate distributions via variables and role conditionals.

## Project Structure

```
ops-os-init/
├── ansible.cfg                          # Ansible 全局配置
├── .ansible-lint                        # Lint 规则配置
├── inventories/
│   └── rocky9.yml                       # Rocky 9 主机清单
├── group_vars/
│   └── rocky9.yml                       # 所有 Role 的可配置变量（集中管理）
├── playbooks/
│   └── init.yml                         # 主 Playbook（编排 7 个 Role）
└── roles/
    ├── init-facts/                      # 基础系统配置
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/
    │       ├── chrony.conf.j2
    │       └── resolv.conf.j2
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
    │   └── files/
    │       ├── ops.pub                  # 占位符，需替换为真实公钥
    │       └── admin.pub                # 占位符，需替换为真实公钥
    ├── init-ssh/                        # SSH 安全加固
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/
    │       ├── sshd_config.d-99-ops-init.conf.j2
    │       └── sshd_config.d-99-ops-emergency.conf.j2
    ├── init-services/                   # 服务管理
    │   ├── defaults/main.yml
    │   └── tasks/main.yml
    └── init-packages/                   # 包管理
        ├── defaults/main.yml
        └── tasks/main.yml
```

## Variable Management

All configurable variables are centralized in `group_vars/rocky9.yml`. Each role also has `defaults/main.yml` as fallback. The group_vars file takes precedence and serves as the single source of truth for customization.

Key variable prefixes:
- `ops_hostname`, `ops_timezone`, `ops_locale`, `ops_dns_*`, `ops_ntp_*` — init-facts
- `ops_limits` — init-kernel
- `ops_sysctl_params` — init-sysctl
- `ops_users`, `ops_users_lock` — init-users
- `ops_ssh_*`, `ops_ssh_emergency_*` — init-ssh
- `ops_services_disable`, `ops_services_enable` — init-services
- `ops_packages_install`, `ops_packages_remove` — init-packages

## Common Commands

```bash
# Run full initialization against Rocky 9 inventory
ansible-playbook playbooks/init.yml -i inventories/rocky9.yml

# Dry run (check mode)
ansible-playbook playbooks/init.yml -i inventories/rocky9.yml --check

# Run specific role by tag (available tags: init-facts, init-kernel, init-sysctl, init-users, init-ssh, init-services, init-packages)
# Short tags also available: facts, kernel, sysctl, users, ssh, services, packages
ansible-playbook playbooks/init.yml -i inventories/rocky9.yml --tags init-facts
ansible-playbook playbooks/init.yml -i inventories/rocky9.yml --tags ssh,users

# Syntax check
ansible-playbook playbooks/init.yml --syntax-check

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
- **Future-ready**: structure should accommodate evolution into a Linux Baseline / Hardening project

## Documentation Language

User-facing documentation is written in **Simplified Chinese**. Ansible task names and code comments may use English.
