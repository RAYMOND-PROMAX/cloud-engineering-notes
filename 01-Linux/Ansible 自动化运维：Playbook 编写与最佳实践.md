# Ansible 自动化运维：Playbook 编写与最佳实践

## 一、问题背景：Ansible 用起来容易，"用好"很难

Ansible 的门槛极低。装个 `ansible-core`，写三行 YAML，`ansible -m ping` 就能跑通。所以几乎每个团队都有一段"Ansible 时代"，但其中很大一部分最后变成了这样：

- site.yml 有 3000 行，一个 Play 里混着 Nginx、MySQL、Java、监控、防火墙，改一行要跑 40 分钟。
- when 条件写了十几层，某台机器走了一条谁也没预料到的分支，配错了一个参数，三天后才发现。
- 有人 ansible-playbook -u root 跑了一次，把 lineinfile 的 regexp 写错，全量改写了 200 台机器的 /etc/ssh/sshd_config，所有人都被锁在外面。
- CI 里跑 --check 显示全部 OK，上线后行为的却不一样，因为 --check 模式对 command/shell 是不执行的。
- 同一份 Playbook，A 同事跑没问题，B 同事跑报 AnsibleUndefinedVariable，因为变量藏在某人本机的 group_vars/all.yml 里。

这些不是"Ansible 坑人"，而是**把配置管理当成了脚本堆叠**。Ansible 的设计哲学是"声明式、幂等、可重复"，但它的语言松散（YAML + Jinja2），从不说"你这样写不对"，所以会奖励快而惩罚懒于设计了。

我经历过一次典型事故：某次发布用 `shell: systemctl restart {{ service_name }}` 循环重启 120 台机器上的 6 个服务，`service_name` 定义在 `group_vars/web.yml`，但 inventory 里有个主机同时属于 `web` 和 `db` 两组，`group_vars` 的合并顺序导致它取到了 `db` 组的同名变量，把数据库重启了。事后看，`group_vars` 命名冲突 + 没有 `assert` 校验 + 没有 `--limit` 分批，三个小问题叠成了 P1。

这篇文章的目标是给出一套**能长期维护**的 Ansible 实践：目录结构、变量层级、Role 设计、幂等性、错误处理、幂等验证、CI 集成、安全和性能。所有示例基于 ansible-core 2.21（Ansible 社区包 14.x）验证。

## 二、版本与环境边界

### 2.1 版本矩阵

| 组件 | 本文基线 | 说明 |
| --- | --- | --- |
| Ansible 社区包 | 14.x | 2026 年当前版本，捆绑 ansible-core 2.21 |
| ansible-core | 2.21.x（“The Rain Song”） | 2.21.0 发布于 2026-05-18 |
| 上一个受支持版本 | 13.x（ansible-core 2.20） | 已进入维护尾声 |
| 已 EOL | 11.x（2.18）、12.x（2.19） | 不要新上 |
| 控制节点 Python | 3.12 / 3.13 | 2.21 支持范围见官方 requirements |
| 被管节点 Python | 3.9+ | 受支持的目标下限，老系统要用 ansible_python_interpreter |

**关键提醒**：`ansible`（社区包）和 `ansible-core` 是两个东西。`pip install ansible-core` 只装核心，不含 200+ 集合（collections）；`pip install ansible==14.*` 装全套。生产控制节点我建议用 `ansible-core` + 明确声明的 `requirements.yml`，**不要装社区全家桶**，因为社区包的版本升级会连带升级所有集合，可能在你不经意时改变模块行为。

### 2.2 ansible-core 2.21 的破坏性变更（升级必读）

2.21 移除了一批长期弃用的东西，存量 Playbook 升级会直接失败：

| 移除项 | 影响 | 替代 |
| --- | --- | --- |
| ansible.builtin.paramiko 连接插件 | ansible_connection=paramiko 直接报插件不存在 | 用 ssh 连接插件，或装 ansible-pylibssh |
| ansible.module_utils.compat.paramiko | 自研模块若 import 它会崩 | 改用 ssh 或 libssh |
| get_bin_path(required=...) 参数 | 自研模块传 required= 会 TypeError | 先判断 None |
| module_utils.basic.safe_eval | 自研模块调用会 AttributeError | 用 ast.literal_eval |
| interpreter_discovery=auto_legacy/auto_legacy_silent | 配置项不认 | 用 auto 或 auto_silent |
| git 模块的 gpg_whitelist 别名 | 参数报错 | 用 gpg_allowlist |
| StrategyBase._load_included_file 的 handle_stats_and_callbacks | 自研策略插件崩 | 按新签名改 |

另外 2.21.0 引入两项重要行为变化：

- `ansible-galaxy` 默认拒绝安装 `requires_ansible` 不兼容的集合。以前会装上然后运行时炸，现在直接跳过。如果你依赖某个老集合，需要设 COLLECTIONS_ON_ANSIBLE_VERSION_MISMATCH=ignore（不推荐，只作过渡）。
- `rc != 0` 但没有 `failed` 键的任务结果不再被判为失败（2.22 起彻底生效，2.21 是弃用警告）。自研 action 插件必须显式设 failed。

升级前的检查清单：

```bash
# 1. 记录当前版本
ansible --version
ansible-core --version 2>/dev/null || true
ansible-galaxy collection list

# 2. 在隔离环境装新版本
python3 -m venv /opt/ansible-2.21
/opt/ansible-2.21/bin/pip install 'ansible-core==2.21.*'

# 3. 用新版本做语法检查（不执行）
/opt/ansible-2.21/bin/ansible-playbook -i inventories/prod/hosts.yml \
  site.yml --syntax-check

# 4. 用新版本跑 check 模式（在测试环境）
/opt/ansible-2.21/bin/ansible-playbook -i inventories/test/hosts.yml \
  site.yml --check --diff
```

`syntax-check` 只查 YAML 和任务结构，不查变量是否存在、模块参数是否正确。真正的坑要在 `--check` 阶段暴露。

### 2.3 控制节点与被管节点的假设

| 项 | 假设 |
| --- | --- |
| 控制节点 | Rocky Linux 9.4，Python 3.12，ansible-core 2.21，4 核 8G |
| 被管节点 | Rocky Linux 9.x / Ubuntu 22.04 / Ubuntu 24.04，Python 3.9+ |
| 连接方式 | SSH Key，ansible 用户 + NOPASSWD sudo 白名单 |
| 规模 | 200~500 台，分组按业务线 + 环境 + 角色 |

## 三、核心知识：先搞清执行模型

很多人写了两年的 Playbook，还没搞清楚 Ansible 的执行模型。这会导致一些"看起来莫名其妙"的行为。

### 3.1 执行流程

```
ansible-playbook
   ↓ 解析 inventory（静态文件 / 动态脚本 / 插件）
   ↓ 解析 playbook（YAML → 任务列表）
   ↓ 每个 play 针对一批 host
       ↓ 收集 facts（除非 gather_facts: false）
       ↓ 按 strategy（线性 / free）下发任务
           ↓ 对每台主机：渲染模板 → 传输模块 → 执行 → 回收结果
       ↓ 触发 handler（在 play 末尾或 flush_handlers）
   ↓ 汇总结果
```

关键点：

**任务是"按主机并行、按顺序串行"的。** 一个 play 里 5 个任务，对 100 台机器来说是"100 台同时跑任务 1，全部完成后跑任务 2"。这叫 `linear` 策略（默认），`serial: 5` 可以让它每批 5 台。

**变量在任务开始前渲染。** 一个任务里的 `{{ var }}` 在执行到该任务时求值，所以前一个任务 `set_fact` 的值可以被后一个任务用。但 `when` 条件、`loop` 的列表都是任务级求值。

**facts 是昂贵的。** 默认每个 play 开始都会在所有主机上跑 `setup` 模块，收集几百个变量。500 台机器大约需要 60~120 秒。不需要时关掉：

```yaml
- hosts: web
  gather_facts: false
```

**handler 不是"立刻执行"的。** 一个任务 `notify` 了 handler，handler 会等到 play 结束（或 `meta: flush_handlers`）才执行，且**同一 handler 在多台主机上只执行一次**（在每台主机上各一次，但同一台主机多次 notify 只执行一次）。

### 3.2 幂等性：Ansible 的核心契约

**幂等的定义**：同一个任务执行一次和执行 N 次，系统状态相同，且第二次之后 `changed=false`。

为什么重要：如果任务不幂等，你永远不知道"再跑一次"是安全的还是灾难。这直接决定你能不能放心地用 Ansible 做日常配置收敛。

哪些模块是幂等的：`copy`、`template`、`file`、`user`、`package`、`service`、`lineinfile`、`blockinfile`、`sysctl`、`mount`、`cron`。

哪些不是：

| 写法 | 问题 | 修正 |
| --- | --- | --- |
| shell: mkdir /data/app | 每次都 changed | file: path=/data/app state=directory |
| shell: systemctl restart nginx | 每次都重启 | service: name=nginx state=restarted（本身也不幂等，用 state=started + handler） |
| command: tar -xzf a.tgz | 每次都解压 | unarchive: src=a.tgz dest=/opt creates=/opt/a/bin |
| lineinfile: line="X=1" 无 regexp | 每次都追加 | 加 regexp: '^X=' |
| shell: echo x >> /etc/f | 追加 | 用 blockinfile 或 lineinfile |

**强制检查方法**：连续跑两遍，第二次必须 `changed=0`。

```bash
ansible-playbook -i inventories/test/hosts.yml site.yml
ansible-playbook -i inventories/test/hosts.yml site.yml | tee /tmp/run2.log
grep -E 'changed=[1-9]' /tmp/run2.log && echo "NOT IDEMPOTENT" || echo "OK"
```

这个检查应该进 CI，每次提交都跑一遍（在测试环境的容器里）。

### 3.3 变量优先级

Ansible 有 22 级变量优先级，记住全部没意义，记住关键几层就够：

```
低 → 高
role defaults/main.yml
inventory group_vars/all
inventory group_vars/<group>
inventory host_vars/<host>
play 的 vars
role 的 vars/main.yml
play 的 vars_files
任务级 set_fact / register
extra vars（-e）
```

**实践原则**：

- 默认值放 roles/<role>/defaults/main.yml（最低优先级，最容易被覆盖）。
- 环境差异放 group_vars/<env>.yml。
- 主机特有放 host_vars/（少用，容易失控）。
- 除调试外不用 `-e`，因为 -e 会覆盖一切，导致"本地能跑线上不能跑"。

**变量名要带角色前缀**，避免 8.1 节那个事故：

```yaml
# roles/nginx/defaults/main.yml  —— 好
nginx_port: 80
nginx_worker_processes: auto

# 不好
port: 80          # 跟 mysql_port、app_port 冲突的风险
```

### 3.4 目录结构：目录即架构

一个能活过三年的 Ansible 仓库，结构基本是固定的：

```
ansible-repo/
├── ansible.cfg
├── requirements.yml
├── site.yml
├── inventories/
│   ├── prod/
│   │   ├── hosts.yml
│   │   ├── group_vars/
│   │   │   ├── all.yml
│   │   │   ├── all/
│   │   │   │   └── vault.yml
│   │   │   ├── web.yml
│   │   │   └── db.yml
│   │   └── host_vars/
│   │       └── web01.example.com.yml
│   └── test/
│       ├── hosts.yml
│       └── group_vars/
├── playbooks/
│   ├── nginx-deploy.yml
│   ├── mysql-upgrade.yml
│   └── os-baseline.yml
├── roles/
│   ├── common/
│   │   ├── defaults/main.yml
│   │   ├── vars/main.yml
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   ├── templates/
│   │   ├── files/
│   │   ├── meta/main.yml
│   │   └── README.md
│   ├── nginx/
│   └── mysql/
├── filter_plugins/
├── library/            # 自研模块
└── molecule/           # 角色测试
```

**每个角色的职责边界**：

- 一个角色只做一件事，nginx 角色装 Nginx 并配好站点，不负责开防火墙。防火墙归 firewall 角色。
- 角色之间通过 dependencies 声明依赖，不在 tasks 里 include_role 一个别的业务角色。
- 角色的 defaults 里给所有变量一个能开箱即用的默认值，让角色能独立测试。

### 3.5 ansible.cfg 基线

`ansible.cfg` 放在仓库根目录，Ansible 会优先读它（顺序：`$ANSIBLE_CONFIG` → `./ansible.cfg` → `~/.ansible.cfg` → `/etc/ansible/ansible.cfg`）。

```ini
[defaults]
inventory            = inventories/prod/hosts.yml
roles_path           = roles
collections_path     = collections
host_key_checking    = True
retry_files_enabled  = False
interpreter_python   = auto_silent
stdout_callback      = yaml
callbacks_enabled    = profile_tasks, timer
forks                = 50
timeout              = 20
gathering            = smart
fact_caching         = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600
deprecation_warnings = True
command_warnings     = False
vault_password_file  = 
log_path             = /var/log/ansible/ansible.log
display_skipped_hosts = False

[privilege_escalation]
become               = True
become_method        = sudo
become_user          = root
become_ask_pass      = False

[ssh_connection]
pipelining           = True
ssh_args             = -o ControlMaster=auto -o ControlPersist=300s -o PreferredAuthentications=publickey
control_path         = /tmp/ansible-ssh-%%h-%%p-%%r
retries              = 3
```

逐项说明关键配置：

**host_key_checking = True**

默认是 `False`，很多人照抄就把这行写成 `False`。但关掉意味着 SSH 不校验主机指纹，中间人攻击无法发现。生产必须 `True`，用 `ssh-keyscan` 预先填充 `known_hosts`：

```bash
# 在控制节点上预先收集指纹（在安全的网络环境下执行）
for h in $(awk '/ansible_host/ {print $2}' inventories/prod/hosts.yml | tr -d ' '); do
    ssh-keyscan -T 3 "$h" >> ~/.ssh/known_hosts 2>/dev/null
done
```

**pipelining = True**

把模块的 Python 代码通过 stdin 直接传给远端 Python，省掉"上传临时文件 → 执行 → 删除"三步。性能提升 20~50%。**前提是被管节点 sudo 不能有 requiretty**：

```bash
# 在被管节点检查
sudo grep -r requiretty /etc/sudoers /etc/sudoers.d/ 2>/dev/null
```

如果有 `Defaults requiretty`，pipelining 会报 `sudo: sorry, you must have a tty to run sudo`，必须去掉（Rocky 9 默认没有，老 CentOS 6 有不少）。

**control_path 与 ControlPersist**

SSH 连接复用，对多任务场景效果显著。`ControlPersist=300s` 表示连接空闲 300 秒才关。

**stdout_callback = yaml**

输出可读性远好于默认的 `skipping: [host] => (item=...)`。缺点是日志里 YAML 格式较长，如果日志平台解析麻烦就改回 `default`。

**callbacks_enabled = profile_tasks, timer**

这两个回调能告诉你哪个任务最慢，是性能优化的入口：

```
Tuesday 12 March 2026  20:14:31 +0800 (0:00:02.341)  0:01:12.551 *****
===============================================================================
role_name : task name -------------------------------------------- 32.41s
other_role : another task ---------------------------------------  8.12s
```

**forks = 50**

并发主机数。200 台机器的话，50 并发意味着 4 轮。太高会打爆控制节点的 fd 和被管节点的 sshd（`MaxStartups`）。控制节点 `ulimit -n` 要够：`50 × 6 ≈ 300`。

**log_path**

默认不记录日志。生产必须开，且要保证目录可写：

```bash
sudo install -d -m 0750 -o ansible -g ansible /var/log/ansible
```

**become 放在 [privilege_escalation] 而不是每个任务里写**

这样所有任务默认 `become`，不需要每行重复。个别不需要的用 `become: false` 关掉。

## 四、整体思路：从"能跑"到"能维护"

我用的落地路径是四步，每一步都有可验收的产出：

**第一步：基线统一（1~2 周）**

目标：所有机器能被同一套 `os-baseline` 角色收敛到一致状态（时区、语言、YUM 源、内核参数、基础包、SSH 配置、监控 agent）。

验收：`ansible-playbook os-baseline.yml --check --diff` 在全部主机上返回 `changed=0`。

**第二步：服务角色化（3~4 周）**

目标：把每个中间件/服务抽成独立角色，`defaults` 里给默认值，`handlers` 里管重启。

验收：任何角色能单独在一个干净的容器里跑通，且二次执行 `changed=0`。

**第三步：环境分层（1 周）**

目标：同一套 playbook 能跑测试和生产，差异全部在 `group_vars/<env>/`。

验收：`diff -r inventories/test/group_vars inventories/prod/group_vars` 的差异都是值，不是结构。

**第四步：纳入 CI（1 周）**

目标：每次 MR 自动跑 lint + syntax-check + 容器内 `--check` + 幂等测试。

验收：有三个以上检查项的 PR 门禁。

## 五、实战：从零写一个生产级角色

以 `nginx` 角色为例，走一遍完整流程。这个角色要满足：幂等、可测试、配置可覆盖、有健康检查、支持平滑 reload。

### 5.1 inventory 设计

`inventories/prod/hosts.yml`：

```yaml
all:
  children:
    prod:
      children:
        web:
          hosts:
            web01.example.com:
              ansible_host: 10.0.2.11
            web02.example.com:
              ansible_host: 10.0.2.12
          vars:
            nginx_role: web
        api:
          hosts:
            api01.example.com:
              ansible_host: 10.0.2.21
            api02.example.com:
              ansible_host: 10.0.2.22
          vars:
            nginx_role: api
        db:
          hosts:
            db01.example.com:
              ansible_host: 10.0.3.11
        monitor:
          hosts:
            mon01.example.com:
              ansible_host: 10.0.9.20
  vars:
    ansible_user: ansible
    ansible_become_method: sudo
    # 注意：不要在变量里写 ~，Ansible 不会做波浪号展开，
    # 会去找字面名为 "~" 的目录。用绝对路径，或依赖 ssh 默认配置。
    # ansible_ssh_private_key_file: /home/deploy/.ssh/ansible_ed25519
    ansible_python_interpreter: /usr/bin/python3
```

**ansible_ssh_private_key_file 的 ~ 陷阱**：变量值里的 `~` 不会被 shell 展开（因为不经过 shell），Ansible 会当成字面路径 `/workdir/~/...`，报 `no such identity`。要么写绝对路径，要么把私钥配进 `~/.ssh/config`，让 inventory 只写 `ansible_user`。

```
# ~/.ssh/config
Host *.example.com
    User ansible
    IdentityFile ~/.ssh/ansible_ed25519
    IdentitiesOnly yes
```

`IdentitiesOnly yes` 很关键：不加的话 ssh 会把 `~/.ssh/` 下所有 key 都试一遍，目标机 `MaxAuthTries` 很快耗尽，报 `Too many authentication failures`。

**为什么用 YAML inventory 而不是 INI**：YAML 能表达嵌套组和组级变量，且可以被程序解析（做 CMDB 同步）。INI 格式在组嵌套时语法别扭。

**主机名用域名而不是 IP**，`ansible_host` 存 IP。这样 `web01.example.com` 这个逻辑名稳定，IP 变了只改一处。

**不要用 [web:children] 那种 INI 嵌套**，YAML 更清晰。

检查 inventory：

```bash
ansible-inventory -i inventories/prod/hosts.yml --graph
```

输出：

```
@all:
  |--@prod:
  |  |--@api:
  |  |  |--api01.example.com
  |  |  |--api02.example.com
  |  |--@db:
  |  |  |--db01.example.com
  |  |--@monitor:
  |  |  |--mon01.example.com
  |  |--@web:
  |  |  |--web01.example.com
  |  |  |--web02.example.com
  |--@ungrouped:
```

```bash
# 验证某台主机的最终变量（调试神器）
ansible-inventory -i inventories/prod/hosts.yml --host web01.example.com
```

`--host` 输出的是"合并后"的变量，能直接看出优先级冲突。

### 5.2 角色骨架

```bash
ansible-galaxy init roles/nginx --role-skeleton=roles/nginx
# 或者手工建目录
mkdir -p roles/nginx/{defaults,vars,tasks,handlers,templates,files,meta}
```

`roles/nginx/defaults/main.yml`：

```yaml
---
# 所有变量都可覆盖，给能开箱即用的默认值
nginx_version: "1.30.6"
nginx_port: 80
nginx_ssl_port: 443
nginx_user: nginx
nginx_worker_processes: auto
nginx_worker_connections: 10240

nginx_conf_dir: /etc/nginx
nginx_log_dir: /var/log/nginx
nginx_pid_file: /run/nginx.pid

# 是否配置 TLS
nginx_tls_enabled: false
nginx_tls_cert: /etc/nginx/certs/server.pem
nginx_tls_key: /etc/nginx/certs/server.key

# upstream 列表，结构化的，便于模板遍历
nginx_upstreams: []
# 例：
# nginx_upstreams:
#   - name: app
#     method: least_conn
#     keepalive: 64
#     servers:
#       - "10.0.2.11:8080"
#       - "10.0.2.12:8080"

# 站点配置
nginx_sites: []
# 例：
# nginx_sites:
#   - name: app
#     server_name: app.example.com
#     upstream: app
#     locations:
#       - path: /
#         proxy: true

# 健康检查端点
nginx_health_path: /healthz

# 是否在 reload 前做健康检查
nginx_healthcheck_on_reload: true
```

`roles/nginx/meta/main.yml`：

```yaml
---
galaxy_info:
  role_name: nginx
  author: ops-team
  description: Install and configure Nginx as a reverse proxy
  license: MIT
  min_ansible_version: "2.20"
  platforms:
    - name: EL
      versions: ["9"]
    - name: Ubuntu
      versions: ["jammy", "noble"]
  galaxy_tags:
    - web
    - nginx

dependencies:
  - role: common
    vars:
      common_timezone: Asia/Shanghai
```

`min_ansible_version: "2.20"` 很重要，是老版本提前报错的唯一手段。

### 5.3 tasks：主流程

`roles/nginx/tasks/main.yml`：

```yaml
---
- name: Validate required variables
  ansible.builtin.assert:
    that:
      - nginx_port | int > 0
      - nginx_worker_connections | int > 1024
      - not (nginx_tls_enabled | bool) or (nginx_tls_cert is defined and nginx_tls_key is defined)
    fail_msg: "Invalid nginx role variables, check group_vars"
    quiet: true
  tags: [always, nginx]

- name: Include OS-specific tasks
  ansible.builtin.include_tasks: "{{ ansible_os_family | lower }}.yml"
  tags: [nginx, packages]

- name: Create nginx directories
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    owner: root
    group: root
    mode: "0755"
  loop:
    - "{{ nginx_conf_dir }}"
    - "{{ nginx_conf_dir }}/conf.d"
    - "{{ nginx_conf_dir }}/snippets"
    - "{{ nginx_log_dir }}"
    - /var/cache/nginx
  tags: [nginx, config]

- name: Deploy nginx.conf
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: "{{ nginx_conf_dir }}/nginx.conf"
    owner: root
    group: root
    mode: "0644"
    validate: "nginx -t -c %s"
    backup: true
  notify: Reload nginx
  tags: [nginx, config]

- name: Deploy upstream snippets
  ansible.builtin.template:
    src: upstream.j2
    dest: "{{ nginx_conf_dir }}/conf.d/upstream-{{ item.name }}.conf"
    owner: root
    group: root
    mode: "0644"
    validate: "nginx -t -c /etc/nginx/nginx.conf"
  loop: "{{ nginx_upstreams }}"
  loop_control:
    label: "{{ item.name }}"
  notify: Reload nginx
  tags: [nginx, config]

- name: Deploy site configs
  ansible.builtin.template:
    src: site.j2
    dest: "{{ nginx_conf_dir }}/conf.d/site-{{ item.name }}.conf"
    owner: root
    group: root
    mode: "0644"
    validate: "nginx -t -c /etc/nginx/nginx.conf"
  loop: "{{ nginx_sites }}"
  loop_control:
    label: "{{ item.name }}"
  notify: Reload nginx
  tags: [nginx, config]

- name: Deploy TLS certificates
  ansible.builtin.copy:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
    owner: root
    group: root
    mode: "0600"
    decrypt: true
  loop:
    - { src: "certs/{{ inventory_hostname }}.pem", dest: "{{ nginx_tls_cert }}" }
    - { src: "certs/{{ inventory_hostname }}.key", dest: "{{ nginx_tls_key }}" }
  when: nginx_tls_enabled | bool
  notify: Reload nginx
  no_log: true
  tags: [nginx, tls]

- name: Remove default site
  ansible.builtin.file:
    path: "{{ nginx_conf_dir }}/conf.d/default.conf"
    state: absent
  notify: Reload nginx
  tags: [nginx, config]

- name: Ensure nginx is enabled and started
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true
  tags: [nginx, service]

- name: Verify nginx is answering locally
  ansible.builtin.uri:
    url: "http://127.0.0.1:{{ nginx_port }}{{ nginx_health_path }}"
    status_code: 200
    timeout: 5
  retries: 5
  delay: 2
  register: nginx_health
  until: nginx_health.status == 200
  changed_when: false
  tags: [nginx, verify]
```

这个 main.yml 里几个设计要点需要单独讲。

**validate: "nginx -t -c %s"**

这是 `template`/`copy` 模块的杀手级参数。它在写入目标文件之前，先把渲染结果放到临时文件，执行 `nginx -t -c <临时文件>`，失败则不写入。**这行配置能挡住 90% 的"改配置把 Nginx 弄挂了"的事故。**

注意 `nginx -t -c %s` 里的 `%s` 是临时文件路径，必须放在 `-c` 后面。如果直接写 `nginx -t`，它会校验 `/etc/nginx/nginx.conf`（旧文件），新配置的错误检测不到。

**backup: true**

改配置前自动备份到 `dest.1234.2026-03-12@20:14:31~`。虽然占空间，但排障时有用。定期清理：

```bash
find /etc/nginx -maxdepth 2 -name '*.conf.[0-9]*' -mtime +30 -delete
```

**notify: Reload nginx 与 handler**

```yaml
# roles/nginx/handlers/main.yml
---
- name: Reload nginx
  ansible.builtin.service:
    name: nginx
    state: reloaded
```

注意是 `state: reloaded` 而不是 `restarted`。Nginx 的 reload 是平滑的，restart 会断连接。

handler 只会在 play 末尾执行一次，即使有 5 个任务都 notify 它。这是特性不是 bug——避免重复 reload。

**loop_control.label**

不设 `label` 时，loop 的输出会把整个 item 打印出来（比如一个包含 20 个字段的 dict），日志难以阅读。设 `label: "{{ item.name }}"` 只显示名字。

**assert 前置校验**

第一个任务用 `assert` 校验变量，失败就停。这比让错误在执行到一半时爆发好得多。`quiet: true` 避免成功时输出一堆 “assertion passed”。

**为什么写成 not A or B 而不是 A and B**

`assert` 的 `that` 列表里每一项都必须是"期望成立"的表达式，任何一项为 `False` 就失败。想表达"开 TLS 时必须有证书路径"，直觉写法是：

```yaml
# 错误：没开 TLS 时也要求证书存在
- nginx_tls_cert is defined
```

因为 `assert` 不理解"条件依赖"，它只看这一项的真假。正确写法是把"条件"和"约束"合成一个蕴含式 `A → B`，在布尔代数里等价的**只有**`not A or B`：

| A（开 TLS） | B（证书已定义） | not A or B | 结论 |
| --- | --- | --- | --- |
| False | False | True | 没开 TLS，不要求证书，通过 |
| False | True | True | 没开 TLS，定义了证书也无所谓，通过 |
| True | False | False | 开了 TLS 但没证书，失败 |
| True | True | True | 通过 |

> 注意：Jinja2 **没有**`implies` 运算符，写 `A implies B` 会直接报 `TemplateSyntaxError: expected token 'name', got 'implies'`。网上一些示例是伪代码，别抄。

**ansible.builtin.uri 做健康检查**

`retries: 5, delay: 2` 配合 `until`，最多等 10 秒。`changed_when: false` 让它不报告 changed（健康检查不是状态变更）。

**OS 分支用 include_tasks 而不是 when: ansible_os_family == 'RedHat' 散落在每个任务**

```yaml
# roles/nginx/tasks/redhat.yml
---
- name: Add nginx official repo
  ansible.builtin.yum_repository:
    name: nginx-stable
    description: nginx stable repo
    baseurl: https://nginx.org/packages/rhel/$releasever/$basearch/
    gpgcheck: true
    gpgkey: https://nginx.org/keys/nginx_signing.key
    enabled: true
    module_hotfixes: true

- name: Install nginx
  ansible.builtin.dnf:
    name: "nginx-{{ nginx_version }}"
    state: present
    disable_gpg_check: false

- name: Ensure firewalld http is open
  ansible.posix.firewalld:
    service: "{{ item }}"
    permanent: true
    immediate: true
    state: enabled
  loop:
    - http
    - https
  when: ansible_facts.services['firewalld.service'] is defined
        and ansible_facts.services['firewalld.service'].state == 'running'
```

```yaml
# roles/nginx/tasks/debian.yml
---
- name: Install prerequisites
  ansible.builtin.apt:
    name: [ca-certificates, curl, gnupg]
    state: present
    update_cache: true
    cache_valid_time: 3600

- name: Add nginx signing key
  ansible.builtin.get_url:
    url: https://nginx.org/keys/nginx_signing.key
    dest: /usr/share/keyrings/nginx-archive-keyring.gpg
    mode: "0644"

- name: Add nginx apt repo
  ansible.builtin.deb822_repository:
    name: nginx
    types: [deb]
    uris: http://nginx.org/packages/ubuntu
    suites: "{{ ansible_distribution_release }}"
    components: [nginx]
    signed_by: /usr/share/keyrings/nginx-archive-keyring.gpg
    state: present

- name: Install nginx
  ansible.builtin.apt:
    name: "nginx={{ nginx_version }}-1~{{ ansible_distribution_release }}"
    state: present
    update_cache: true
```

**注意**：2.21 起 `apt_key` 和 `apt_repository` 已弃用，官方推荐 `deb822_repository`。上面的写法就是新推荐方式，Ubuntu 24.04（noble）默认支持。

`module_hotfixes: true` 是 RedHat 系的关键参数。RHEL 9 的 AppStream 模块流会屏蔽第三方仓库的 nginx 包，不加这个参数 `dnf` 会说"找不到包"。

### 5.4 模板：nginx.conf.j2

```
{# roles/nginx/templates/nginx.conf.j2 #}
# Managed by Ansible - do not edit manually
# Role: nginx  Host: {{ inventory_hostname }}  Generated: {{ ansible_date_time.iso8601 | default('unknown') }}
user {{ nginx_user }};
worker_processes {{ nginx_worker_processes }};
worker_rlimit_nofile {{ (nginx_worker_connections | int) * 2 }};

error_log {{ nginx_log_dir }}/error.log warn;
pid {{ nginx_pid_file }};

events {
    worker_connections {{ nginx_worker_connections }};
    multi_accept on;
    use epoll;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent rt=$request_time '
                    'urt=$upstream_response_time ua=$upstream_addr';

    access_log {{ nginx_log_dir }}/access.log main buffer=32k flush=5s;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    keepalive_requests 1000;
    server_tokens off;
    client_max_body_size {{ nginx_client_max_body_size | default('20m') }};

    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;

{% if nginx_tls_enabled | bool %}
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:20m;
    ssl_session_timeout 1h;
    ssl_session_tickets on;
{% endif %}

    include /etc/nginx/conf.d/*.conf;
}
```

模板变量 `{{ ansible_date_time.iso8601 }}` 有个陷阱：**每次渲染这个值都变，导致 changed 永远为 true，幂等性被破坏**。如果只需要"知道什么时候部署的"，用文件 mtime 而不是写进内容。如果一定要写，用 `{{ ansible_date_time.iso8601 | default('unknown') }}` 并且接受不幂等，或者改成只在首次部署时写：

```yaml
- name: Check if nginx.conf exists
  ansible.builtin.stat:
    path: "{{ nginx_conf_dir }}/nginx.conf"
  register: nginx_conf_stat

- name: Set generation timestamp
  ansible.builtin.set_fact:
    nginx_generated_at: >-
      {{ (nginx_conf_stat.stat.exists | default(false))
         | ternary('preserved', ansible_date_time.iso8601) }}
```

`{{ .. | ternary(a, b) }}` 是 Jinja 的三元，注意参数顺序是 `ternary(if_true, if_false)`。这种写法能让"首次部署写时间，后续保留"。

### 5.5 handlers：进阶用法

基本 handler 只能"重启服务"。实际需要更细的控制：

```yaml
# roles/nginx/handlers/main.yml
---
- name: Validate nginx config
  ansible.builtin.command:
    cmd: nginx -t
  changed_when: false
  listen: "reload nginx"

- name: Reload nginx
  ansible.builtin.service:
    name: nginx
    state: reloaded
  listen: "reload nginx"

- name: Wait for nginx to be ready
  ansible.builtin.uri:
    url: "http://127.0.0.1:{{ nginx_port }}{{ nginx_health_path }}"
    status_code: 200
  register: nginx_ready
  until: nginx_ready.status == 200
  retries: 10
  delay: 1
  changed_when: false
  listen: "reload nginx"
```

**listen 是 handler 的"订阅主题"**。多个 handler 监听同一个 `listen` 名字时，会按定义顺序执行。上面的配置实现"校验 → reload → 等就绪"三步，而调用方只需 `notify: reload nginx`。

`listen` 的另一个好处是让 `include_role` 里的任务也能 notify（普通 handler 名字在同一 role 内可见，跨 role 需要 `listen`）。

**meta: flush_handlers**

如果需要在 play 中间就执行 handler（比如 reload 后马上要验证），用：

```yaml
- name: Flush handlers to apply config now
  ansible.builtin.meta: flush_handlers

- name: Verify after reload
  ansible.builtin.uri:
    url: "https://{{ nginx_sites[0].server_name }}{{ nginx_health_path }}"
    validate_certs: false
    status_code: 200
```

注意 `flush_handlers` 之后，之前 notify 过的 handler 不会在 play 末尾再执行一次。

**handler 里不要放 restart 之外的破坏性操作**。handler 是"配置收敛后的动作"，不是业务逻辑的挂载点。

## 六、Playbook 编写规范

### 6.1 一个可维护的 site.yml

```yaml
---
# site.yml - 全量收敛入口
- name: OS baseline for all hosts
  hosts: prod
  gather_facts: true
  serial: "30%"
  max_fail_percentage: 10
  roles:
    - role: common
    - role: security_baseline

- name: Configure web tier
  hosts: web
  gather_facts: false
  serial: "20%"
  max_fail_percentage: 5
  roles:
    - role: nginx
    - role: node_exporter

- name: Configure api tier
  hosts: api
  gather_facts: false
  serial: "20%"
  max_fail_percentage: 5
  roles:
    - role: nginx
    - role: node_exporter

- name: Configure db tier
  hosts: db
  gather_facts: false
  serial: 1
  max_fail_percentage: 0
  roles:
    - role: mysql
```

**serial: "30%" 与 max_fail_percentage**

`serial` 让 play 分批执行。30%，500 台就是每批 150 台。这对"降低变更爆炸半径"是关键。

`max_fail_percentage: 10` 表示"如果失败主机超过本批的 10%，停止整批"。没有这个参数，Ansible 会一直跑完所有批次，即使前面全炸了。

**db 层 serial: 1 + max_fail_percentage: 0**

数据库不能并行变更，任何一台失败都要立即停止。

**gather_facts: false 的前提**

关掉 facts 能省时间，但角色里如果用了 `ansible_os_family`、`ansible_distribution_release`，就会报未定义。两种解法：在更早的 play 里收集并把 facts 缓存（`fact_caching`），或者用 `setup` 模块只收集需要的子集：

```yaml
- name: Gather minimal facts
  ansible.builtin.setup:
    gather_subset:
      - "!all"
      - "!min"
      - os_family
      - distribution
      - distribution_release
```

`gather_subset` 用排除语法 `!all` 再逐步加回，能省 50% 的收集时间。

### 6.2 任务命名：给半年后的自己

**规范**：每个任务的 `name` 必须是"动词 + 宾语 + 关键限定"，第一个字母大写，不用句号结尾。

```yaml
# 好
- name: Install nginx {{ nginx_version }} from official repo
- name: Deploy site config for {{ item.name }}
- name: Ensure /data is mounted from /dev/vdb1

# 差
- name: install nginx
- name: copy files
- name: "{{ item }}"     # 这样等于没名字
```

为什么要较真：`--check` 的输出、日志、失败报告全靠 `name`。一个叫 “copy files” 的任务失败了，你不知道它复制了什么。

**no_log 的使用**

涉及密码、Token 的任务必须 `no_log: true`，否则会原文打印到 stdout 和 `log_path`：

```yaml
- name: Create app database user
  community.mysql.mysql_user:
    name: "{{ app_db_user }}"
    password: "{{ app_db_password }}"
    priv: "{{ app_db_name }}.*:ALL"
    state: present
  no_log: true
```

`no_log: true` 会让失败时的输出也被隐藏，排查困难。折中做法是只对敏感参数隐藏：

```yaml
- name: Create app database user
  community.mysql.mysql_user:
    name: "{{ app_db_user }}"
    password: "{{ app_db_password }}"
    priv: "{{ app_db_name }}.*:ALL"
  no_log: "{{ not (ansible_verbosity >= 2) }}"
```

这样 `-vv` 时能看到输出（调试用），默认隐藏。**注意这会让 -vv 泄露密码到终端，只在受控环境用。**

### 6.3 条件：`when` 的正确写法

`when` 的表达式是 Jinja2 裸表达式，**不要加 {{ }}**：

```yaml
# 正确
when: ansible_os_family == "RedHat"

# 错误（虽然有时能跑，但会输出警告）
when: "{{ ansible_os_family == 'RedHat' }}"
```

**复合条件用 YAML 列表（隐含 AND）**：

```yaml
when:
  - ansible_os_family == "RedHat"
  - ansible_distribution_major_version | int >= 9
  - nginx_tls_enabled | bool
```

比 `and` 连接更可读，也更容易在 diff 里看出改了哪个条件。

**is defined 与默认值**

```yaml
# 危险：变量未定义时整个 play 失败
when: nginx_extra_config | length > 0

# 安全
when: (nginx_extra_config | default([])) | length > 0

# 更安全：在 defaults 里给默认值，业务代码里就不用 default
```

**规范：所有变量在 defaults/main.yml 里都要有定义**，`tasks` 里不写 `| default()`。这样"变量未定义"永远是一个真实的 bug，而不是被 `default` 掩盖。

**changed_when 与 failed_when**

`command`/`shell` 默认"永远 changed"，且以 `rc` 判断成功。这两个都可以改：

```yaml
- name: Check current kernel version
  ansible.builtin.command:
    cmd: uname -r
  register: kernel_ver
  changed_when: false

- name: Check if app is running
  ansible.builtin.shell:
    cmd: |
      set -o pipefail
      pgrep -f 'java.*app.jar' | wc -l
  register: app_procs
  changed_when: false
  failed_when: false
```

`command` 模块**不支持管道、重定向、变量展开**。需要这些必须用 `shell`，且 `shell` 里首行加 `set -o pipefail`，否则管道的第一个命令失败会被最后一个命令的退出码掩盖。

这是最容易出错的点：

```yaml
# 危险：grep 没找到时 rc=1，但 wc -l 返回 0，整体 rc=0
- shell: systemctl status nginx | grep active | wc -l

# 正确
- shell: |
    set -o pipefail
    systemctl status nginx | grep -c active
```

### 6.4 循环：`loop` 优先，`with_*` 只用于特殊情况

`loop` 是 2.5 引入的新语法，推荐全部用它。`with_items`/`with_dict` 等是老语法，虽然还能用但要逐步迁移。

```yaml
# 推荐
- name: Create app directories
  ansible.builtin.file:
    path: "{{ item.path }}"
    state: directory
    owner: "{{ item.owner | default('app') }}"
    mode: "{{ item.mode | default('0755') }}"
  loop:
    - { path: /data/app/logs, mode: "0755" }
    - { path: /data/app/conf, mode: "0750" }
    - { path: /data/app/tmp,  mode: "1777" }
  loop_control:
    label: "{{ item.path }}"
```

**loop 配合 until 只在单台机器上有效**，多台时每台独立循环。

**loop_control 的三个常用参数**：

```yaml
loop_control:
  label: "{{ item.name }}"      # 日志里显示什么
  pause: 1                       # 每次循环间隔秒数（对 API 限流有用）
  index_var: idx                 # 当前索引，从 0 开始
```

**并发执行 loop**（2.12+ 支持 `loop` 并行？不支持，但可以用 `async`）：

```yaml
- name: Download packages in parallel
  ansible.builtin.get_url:
    url: "{{ item.url }}"
    dest: "{{ item.dest }}"
    mode: "0644"
  loop: "{{ downloads }}"
  loop_control:
    label: "{{ item.dest }}"
  async: 300
  poll: 0
  register: dl_jobs

- name: Wait for downloads
  ansible.builtin.async_status:
    jid: "{{ item.ansible_job_id }}"
  register: dl_result
  until: dl_result.finished
  retries: 60
  delay: 5
  loop: "{{ dl_jobs.results }}"
  loop_control:
    label: "{{ item.item.dest }}"
```

`async: 300 poll: 0` 表示"异步执行，最多 300 秒，不等待"。后面用 `async_status` 轮询。这个模式适合"大批量下载"或"批量执行慢命令"。

### 6.5 blocks：错误处理与标签

```yaml
- name: Deploy nginx with rollback on failure
  block:
    - name: Backup current config
      ansible.builtin.copy:
        src: /etc/nginx/nginx.conf
        dest: /etc/nginx/nginx.conf.rollback
        remote_src: true
        mode: "0644"

    - name: Deploy new config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        validate: "nginx -t -c %s"
      notify: reload nginx

    - name: Flush handlers
      ansible.builtin.meta: flush_handlers

    - name: Verify site responds
      ansible.builtin.uri:
        url: "http://127.0.0.1:{{ nginx_port }}{{ nginx_health_path }}"
        status_code: 200
        timeout: 5
      register: verify

  rescue:
    - name: Rollback config
      ansible.builtin.copy:
        src: /etc/nginx/nginx.conf.rollback
        dest: /etc/nginx/nginx.conf
        remote_src: true
        mode: "0644"

    - name: Reload after rollback
      ansible.builtin.service:
        name: nginx
        state: reloaded

    - name: Fail the play loudly
      ansible.builtin.fail:
        msg: "nginx deploy failed on {{ inventory_hostname }}, rolled back"

  always:
    - name: Record deploy result
      ansible.builtin.lineinfile:
        path: /var/log/nginx-deploy.log
        line: "{{ ansible_date_time.iso8601 }} {{ inventory_hostname }} {{ ansible_failed_task.name | default('success') }}"
        create: true
        mode: "0644"
```

这是**部署类 Playbook 的标准骨架**：备份 → 部署 → 验证 → 失败回滚 → 记录。

`rescue` 只在 `block` 里有任务失败时执行。`always` 无论如何都执行。

**注意**：`block` 里的失败默认会让主机从 play 中移除（`any_errors_fatal` 未设时，其他主机会继续）。如果需要"一台失败就停全部"，在 play 级加 `any_errors_fatal: true`。

**Tags：让 Playbook 可切分**

```yaml
- name: Configure web tier
  hosts: web
  roles:
    - role: nginx
      tags: [nginx]
    - role: node_exporter
      tags: [monitoring]
```

```bash
# 只跑配置，不装包
ansible-playbook site.yml --tags config

# 跳过危险任务
ansible-playbook site.yml --skip-tags dangerous

# 列出所有 tags（写 Playbook 时常用）
ansible-playbook site.yml --list-tags

# 列出即将执行的任务（不执行，最佳彩排方式）
ansible-playbook site.yml --list-tasks --tags nginx
```

**规范**：每个角色至少打两个 tag（角色名 + 功能分类），且每个任务最好有自己的 tag。`always` tag 的任务在任何 `--tags` 下都会执行，只用于"必须的前提检查"。

### 6.6 变量文件与环境分层

```
inventories/
├── prod/
│   ├── hosts.yml
│   └── group_vars/
│       ├── all.yml           # 所有生产环境共享
│       ├── all/
│       │   └── vault.yml     # 加密的凭证
│       ├── web.yml           # web 组
│       └── db.yml
└── test/
    ├── hosts.yml
    └── group_vars/
        ├── all.yml
        └── web.yml
```

`inventories/prod/group_vars/all.yml`：

```yaml
---
env_name: prod
env_domain: example.com
ntp_servers:
  - ntp1.aliyun.com
  - ntp2.aliyun.com

# 所有环境共享的通用参数
common_timezone: Asia/Shanghai
common_admin_users:
  - ops-a
  - ops-b

# 引用 vault 里的变量（vault.yml 里定义）
# db_root_password: !vault |...
```

`inventories/prod/group_vars/web.yml`：

```yaml
---
nginx_port: 80
nginx_tls_enabled: true
nginx_worker_connections: 20480

nginx_upstreams:
  - name: app
    method: least_conn
    keepalive: 64
    servers:
      - "10.0.2.21:8080"
      - "10.0.2.22:8080"

nginx_sites:
  - name: app
    server_name: "app.{{ env_domain }}"
    upstream: app
    locations:
      - path: /
        proxy: true
      - path: /static/
        proxy: true
        cache: true

node_exporter_version: "1.10.2"
```

**{{ env_domain }} 在 group_vars 里能引用另一个 group_vars 变量**，因为变量在渲染时统一解析。这比把域名写死在每个地方好维护。

**环境差异只允许"值不同"，不允许"结构不同"**。否则测试环境跑通不代表生产能跑。检查：

```bash
# 对比两套环境 group_vars 的键结构（不是值）
for f in all web db; do
  echo "=== ${f} ==="
  diff <(yq -r 'keys[]' "inventories/test/group_vars/${f}.yml" 2>/dev/null) \
       <(yq -r 'keys[]' "inventories/prod/group_vars/${f}.yml" 2>/dev/null)
done
```

有差异就要问：为什么生产多一个变量？是不是测试环境缺配置？

## 七、Vault：凭证管理

### 7.1 基本用法

```bash
# 生成 vault 密码文件（只做一次，存到受控位置）
openssl rand -base64 32 > ~/.ansible/vault-pass
chmod 600 ~/.ansible/vault-pass

# 创建加密变量文件
ansible-vault create inventories/prod/group_vars/all/vault.yml

# 编辑
ansible-vault edit inventories/prod/group_vars/all/vault.yml

# 加密已有文件
ansible-vault encrypt inventories/prod/group_vars/all/secrets.yml

# 查看（不修改）
ansible-vault view inventories/prod/group_vars/all/vault.yml

# 改密码
ansible-vault rekey inventories/prod/group_vars/all/vault.yml
```

`vault.yml` 的内容规范：

```yaml
---
# 命名约定：vault_ 前缀，与明文变量区分
vault_db_root_password: "S3cur3-P@ss-2026"
vault_app_db_password: "An0ther-P@ss"
vault_api_token: "ghp_xxxxxxxxxxxx"

# 同时提供"明文别名"，业务里引用明文名，方便 grep
db_root_password: "{{ vault_db_root_password }}"
app_db_password: "{{ vault_app_db_password }}"
api_token: "{{ vault_api_token }}"
```

为什么加别名层：如果 tasks 里直接写 `{{ vault_app_db_password }}`，全仓库都是 `vault_` 前缀；加了别名层后，tasks 里写 `{{ app_db_password }}`，改成"从外部密钥系统读取"时只改这一层。

### 7.2 密码的传递方式（优先级从高到低）

| 方式 | 命令 | 适用 |
| --- | --- | --- |
| 环境变量 | ANSIBLE_VAULT_PASSWORD_FILE | CI/CD |
| 配置项 | ansible.cfg 的 vault_password_file | 个人开发机 |
| 命令行 | --vault-password-file | 临时 |
| 交互输入 | --ask-vault-pass | 手工操作，推荐日常用 |

**绝对不要在 ansible.cfg 里写明文密码，也不要把 vault 密码文件提交到 Git。**

CI 里的做法：

```yaml
# .gitlab-ci.yml 片段
deploy:
  script:
    - export ANSIBLE_VAULT_PASSWORD_FILE="${VAULT_PASS_FILE}"
    - ansible-playbook -i inventories/prod/hosts.yml site.yml --limit web
  variables:
    VAULT_PASS_FILE: /run/secrets/vault_pass
  secrets:
    - vault_pass
```

GitLab 的 `secrets:` 需要先在实例的 `vault` 配置里把外部密钥管理器（Vault、AWS Secrets Manager、GCP Secret Manager、Azure Key Vault）注册为 provider，Job 里才能按名字引用。文件被挂到 `/run/secrets/<name>`，Job 结束即销毁，不会落到构建缓存或制品里。如果实例没配 `secrets:`，退而用 `File` 类型的 CI/CD 变量（在 Settings → CI/CD → Variables 里把类型选成 File），效果接近，但值存在 GitLab 的数据库里而不是外部 KMS。

### 7.3 更安全的方案：不落盘的凭证

Vault 的问题是"密码文件存在于磁盘"。更高要求的场景（等保三级、金融）应该用外部密钥管理：

```yaml
# 用 community.hashi_vault 从 Vault 读
- name: Get db password from Vault
  community.hashi_vault.vault_kv2_get:
    url: "https://vault.example.com:8200"
    path: "secret/data/prod/db"
    auth_method: approle
    role_id: "{{ lookup('env', 'VAULT_ROLE_ID') }}"
    secret_id: "{{ lookup('env', 'VAULT_SECRET_ID') }}"
  register: db_secret
  no_log: true
  delegate_to: localhost

- name: Create db user
  community.mysql.mysql_user:
    name: app
    password: "{{ db_secret.secret.data.password }}"
    priv: "app.*:ALL"
  no_log: true
  when: db_secret is succeeded
```

`delegate_to: localhost` 让查询在控制节点执行，被管节点不接触 Vault 凭证。

**注意**：`lookup('env', ...)` 会在控制节点读环境变量，这是 CI 注入凭证的标准方式。**不要用 lookup('file', ...) 读本地密码文件**，那等于绕过了 vault 的意义。

### 7.4 vault 的常见误用

**误用一：把 vault 密码文件放在仓库里**

`git log` 能查到历史，即使后来删了。一旦泄露，所有历史密码都暴露，必须全部轮换。

**误用二：一个 vault 密码覆盖所有环境**

生产 vault 密码泄露 = 生产凭证泄露。至少测试和生产用两个独立的 vault 密码文件。

**误用三：ansible-vault encrypt_string 后把密文贴到 Playbook 里**

```yaml
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  66386439653236336...
```

这本身没问题，但密文混在业务配置里，改一个变量要 edit 整个文件，容易误伤。建议放独立的 `vault.yml`。

**误用四：以为 vault 加密了 inventory**

vault 只加密变量文件。inventory 里的 `ansible_password` 是明文的。正确做法是用 SSH Key，不用密码认证。

## 八、故障排查：四个真实案例

### 8.1 案例一：group_vars 同名变量导致重启错服务

**现象**：发布脚本执行 `systemctl restart {{ service_name }}`，有一台机器上的 MySQL 被重启了，业务中断 3 分钟。

**初判**：脚本只针对 `web` 组，为什么会碰 MySQL？

**检查**：

```bash
# 1. 看这台机器属于哪些组
ansible-inventory -i inventories/prod/hosts.yml --host app01.example.com | head -20
```

输出里 `group_names` 是：

```json
"group_names": ["prod", "web", "db"]
```

这台机器同时属于 `web` 和 `db`（早期为了 SSH 跳板复用留下的）。而 `group_vars` 的加载顺序是按字母序：`db.yml` 在 `web.yml` 之后，所以 `service_name` 取到了 `db.yml` 里的 `mysql`。

```bash
# 2. 确认变量来源
ansible -i inventories/prod/hosts.yml app01.example.com -m debug -a "var=service_name"
```

**根因**：`service_name` 这种通用名在不同组的 `group_vars` 里重复定义，加载顺序不定（实际是字母序，但不应该依赖）。

**修复**（三处）：

```yaml
# 1. 变量名加前缀
# inventories/prod/group_vars/web.yml
web_service_name: nginx
# inventories/prod/group_vars/db.yml
db_service_name: mysqld
```

```yaml
# 2. 在 role 的 defaults 里用局部名，通过参数传入
# roles/service/defaults/main.yml
svc_name: ""

# roles/service/tasks/main.yml
- name: Restart {{ svc_name }}
  ansible.builtin.service:
    name: "{{ svc_name }}"
    state: restarted
  when: svc_name | length > 0
```

```yaml
# 3. 加断言，变量不对就停
- name: Validate service name matches host role
  ansible.builtin.assert:
    that:
      - web_service_name == 'nginx'
    fail_msg: "web_service_name must be nginx on web hosts"
```

**验证**：

```bash
# 在每台机器上确认变量解析结果，不允许有歧义
ansible -i inventories/prod/hosts.yml web -m debug -a "var=web_service_name" | sort | uniq -c
```

**回滚预案**：变量改名是纯正向变更，回滚就是改回原名。但因为原行为是错的，不建议回滚。

**复盘要点**：

1. 不要让一台机器同时属于互斥的业务组。如果因为网络原因必须这样，用独立的 bastion_access 组，不要复用业务组。
2. 变量名必须带角色/组前缀，这是硬规范。
3. 部署脚本第一行应该是 assert，把预期钉死。

### 8.2 案例二：`--check` 通过但实际执行失败

**现象**：CI 里 `ansible-playbook --check` 全绿，合并后真实执行，在 `Install package` 任务失败：`No package matching 'nginx-1.30.6' found`。

**初判**：`--check` 不可能真的执行 `dnf install`，所以它"通过"是假象。

**检查**：

```bash
# 1. 确认 --check 的跳过行为
ansible-playbook site.yml --check -vvv 2>&1 | grep -A2 'No package'
```

输出里能看到 `skipped: [web01] => {"changed": false, "msg": "check mode not supported for dnf"}` 之类。

`dnf`/`apt`/`yum` 模块在 check 模式下**只检查"包是否已安装"**，不会检查"这个版本是否存在"。所以版本号写错时，`--check` 依然通过。

**2. 确认版本号从哪来**

```bash
ansible -i inventories/prod/hosts.yml web01.example.com -m debug -a "var=nginx_version"
```

输出 `1.30.6`，但仓库里只有 `1.30.4`。因为 `nginx_version` 是手工填的，没跟仓库同步。

**根因**：版本变量是硬编码的，没人负责同步。

**修复**：

```yaml
# 改成"查询可用版本"
- name: Get available nginx version from repo
  ansible.builtin.command:
    cmd: dnf --showduplicates list nginx --available
  register: nginx_avail
  changed_when: false
  check_mode: false    # 关键：check 模式下也要跑

- name: Assert requested version exists
  ansible.builtin.assert:
    that:
      - nginx_version in nginx_avail.stdout
    fail_msg: |
      nginx {{ nginx_version }} not available.
      Available: {{ nginx_avail.stdout_lines | select('match', '^nginx\\.') | list }}
```

`check_mode: false` 让这个任务在 `--check` 时也真的执行，从而让 `assert` 在 check 阶段就能发现版本不存在。

**更彻底的方案**：CI 里不要在裸机上跑 `--check`，而是在**容器里跑真实执行**：

```yaml
# .github/workflows/ansible.yml 片段
jobs:
  idempotency:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Start target container
        run: |
          docker run -d --name target --privileged \
            -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
            geerlingguy/docker-rockylinux9-ansible:latest \
            /usr/sbin/init
      - name: Prepare inventory
        run: |
          printf 'all:\n  hosts:\n    target:\n      ansible_connection: docker\n' \
            > /tmp/inv.yml
      - name: First run (must change)
        run: ansible-playbook -i /tmp/inv.yml site.yml
      - name: Second run (must be idempotent)
        run: |
          ansible-playbook -i /tmp/inv.yml site.yml | tee /tmp/run2.log
          if grep -qE 'changed=[1-9]' /tmp/run2.log; then
            echo "Playbook is not idempotent"; exit 1
          fi
```

**验证**：CI 通过后，再在预发环境跑一次真实执行，两遍都是 `changed=0` 才允许上生产。

**复盘要点**：

1. --check 是"尽力而为"的模拟，对包管理、command、shell 都不准。它不能替代真实执行。
2. 版本号不要硬编码，从仓库查询或用 latest（但要接受不可控）。
3. 幂等测试必须在真实环境跑两遍。

### 8.3 案例三：pipelining 导致 sudo 失败

**现象**：开启 `pipelining = True` 后，任务报错：

```
fatal: [web01.example.com]: FAILED! => {"msg": "sudo: sorry, you must have a tty to run sudo"}
```

**初判**：pipelining 通过 stdin 传模块代码，sudo 在无 TTY 时被拒。

**检查**：

```bash
# 1. 在被管节点查看 sudoers 配置
sudo grep -rn 'requiretty' /etc/sudoers /etc/sudoers.d/ 2>/dev/null

# 2. 确认 pipelining 生效
ansible-config dump | grep -i pipelining
```

**根因**：`/etc/sudoers.d/legacy` 里有一行 `Defaults requiretty`（从老系统迁移时带过来的）。

**修复**（两种，选其一）：

```bash
# 方案 A：去掉 requiretty（推荐）
sudo sed -i '/^Defaults\s*requiretty/d' /etc/sudoers.d/legacy
sudo visudo -c
```

```bash
# 方案 B：只对 ansible 用户去 requiretty
echo 'Defaults:ansible !requiretty' | sudo tee /etc/sudoers.d/ansible
sudo chmod 0440 /etc/sudoers.d/ansible
sudo visudo -c
```

**注意**：改 sudoers 是高危操作。`visudo -c` 校验通过才不会把所有人锁在外面。**改之前先开一个 root shell 保活**，万一改坏了还能修。

```bash
# 保活方式
sudo -i    # 另开一个窗口，保持不动
# 在另一个窗口改 sudoers，改完 visudo -c 校验
```

**验证**：

```bash
ansible -i inventories/prod/hosts.yml web -m ping
ansible -i inventories/prod/hosts.yml web -m command -a "id -u" -b
```

第二条应该返回 `0`（root 的 uid）。

**回滚预案**：删掉 `/etc/sudoers.d/ansible`，或在 `/etc/sudoers.d/legacy` 里恢复 `Defaults requiretty`。修复期间保持一个 root shell 开着。

**复盘要点**：

1. pipelining 不是"开了就好"，要先检查 sudoers。
2. 老系统迁移时带过来的 sudoers 配置是隐形炸弹，统一基线时要清理。
3. 修改 sudoers 前必须开保活 root shell，这条要写进运维手册。

### 8.4 案例四：`forks` 太高导致连接失败

**现象**：500 台机器的全量执行，报大量 `Failed to connect to the host via ssh: ssh: connect to host ... port 22: Connection refused`。

**初判**：被管节点的 sshd 拒绝了连接，可能是 `MaxStartups` 限制。

**检查**：

```bash
# 1. 在被管节点看 sshd 日志
sudo journalctl -u sshd --since "10 min ago" | grep -Ei 'maxstartups|drop'
```

典型输出：

```
sshd[12345]: error: maximum authentication attempts exceeded
sshd[12346]: drop connection #11 from [10.0.1.5]:51234 on [10.0.2.11]:22 past MaxStartups
```

**2. 看 sshd 配置**

```bash
sudo sshd -T | grep -E 'maxstartups|maxsessions|maxauthtries'
```

输出 `maxstartups 10:30:100`，意思是"未认证连接超过 10 个开始 30% 概率丢弃，超过 100 个全丢"。

**根因**：`forks = 50` 时，控制节点同时向 50 台机器建连。但如果某个 play 用了 `serial: 100`，同一批 100 台同时建连，就会超过 `MaxStartups`。

**修复**（三个层面）：

```bash
# 1. 调整被管节点 sshd（通过 Ansible 基线角色统一改）
```

```yaml
# roles/common/templates/sshd_hardening.conf.j2
MaxStartups 100:30:200
MaxSessions 50
MaxAuthTries 4
LoginGraceTime 20
ClientAliveInterval 300
ClientAliveCountMax 3
```

```bash
# 2. 调整 forks，不要超过单机能承受的建连速率
# ansible.cfg
forks = 50
```

```bash
# 3. 调整 serial，控制每批的并发
```

```yaml
serial: 20
```

**验证**：

```bash
# 压测连接建立能力（在控制节点执行，不实际登录）
for i in $(seq 1 100); do
  (timeout 3 bash -c "echo > /dev/tcp/10.0.2.11/22" 2>/dev/null && echo ok || echo fail) &
done | sort | uniq -c
```

如果 `fail` 占比超过 1%，说明 `MaxStartups` 还是不够。

**回滚预案**：`sshd` 改完必须 `sshd -t` 校验后再 reload：

```bash
sudo sshd -t && sudo systemctl reload sshd
```

`sshd -t` 不通过就**千万不要 reload**，否则现有连接还在但新连接全失败，等于把自己锁在外面。**改 sshd 前同样要开保活 shell。**

**复盘要点**：

1. forks 是"同时操作的主机数"，不是"绝对并发任务数"。要跟 serial 一起看。
2. 被管节点的 sshd 限制是全局的，不只是给 Ansible 用，调之前确认不影响其他系统。
3. 大规模执行前先在 5 台机器上试，观察建连错误率。

### 8.5 通用排查路径

**第一步：确认是"连不上"还是"跑错了"**

```bash
# 连通性单独测
ansible -i inventories/prod/hosts.yml prod -m ping --one-line

# 只跑 facts 收集
ansible -i inventories/prod/hosts.yml prod -m setup -a 'gather_subset=!all' --one-line
```

**第二步：看变量解析结果**

```bash
# 单机全量变量
ansible-inventory --host web01.example.com

# 单个变量
ansible -i inventories/prod/hosts.yml web01.example.com -m debug -a "var=nginx_port"
```

**第三步：看任务展开后的实际模块参数**

```bash
ansible-playbook site.yml --limit web01.example.com -vvv --start-at-task "Deploy nginx.conf" 2>&1 | grep -A30 'TASK'
```

`-vvv` 会打印渲染后的模块参数，能看出 `{{ }}` 解析成了什么。

**第四步：在目标机上手工复现**

用 `-vvv` 拿到的模块参数，手工执行一遍。80% 的问题在这一步定位。

**第五步：看日志**

```bash
# 控制节点
tail -100 /var/log/ansible/ansible.log

# 被管节点
sudo journalctl -u sshd --since "5 min ago"
sudo tail -50 /var/log/messages
```

## 九、性能优化：大规模执行的实操

### 9.1 优化项与收益

| 优化项 | 典型收益 | 代价 |
| --- | --- | --- |
| pipelining = True | 20~50% 时间 | 需 sudo 无 requiretty |
| gather_facts: false + gather_subset | 10~30% 时间 | 需要 facts 时要单独收集 |
| fact_caching = jsonfile | 第二次起省 facts 时间 | 缓存过期策略要设计 |
| forks 提高 | 线性提升到瓶颈 | 打爆 sshd / 控制节点 fd |
| SSH ControlPersist | 20~40% | 无 |
| strategy = free | 大任务集下 30%+ | 任务顺序不保证，慎用 |
| 减少 include_tasks 层数 | 5~10% | 结构可能变差 |

### 9.2 `strategy: free` 的适用与风险

默认 `linear` 策略是"所有主机完成当前任务才进入下一个"。如果某台机器特别慢（比如装了 50 个包），所有机器都等它。

`free` 策略让每台机器独立推进，速度不受最慢主机影响：

```yaml
- hosts: web
  strategy: free
  serial: 20
```

**风险**：任务执行顺序在主机间不再同步。如果有"先所有机器停服务，再所有机器升级，再集体启动"这类需要全局同步的流程，`free` 会导致部分机器还在跑旧版本时另一部分已经启动。

**判断依据**：无状态配置（装包、写配置、同步文件）可以用 `free`；需要集群协调的（数据库主从切换、滚动重启）必须 `linear`。

### 9.3 facts 缓存的正确配置

```ini
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /var/cache/ansible/facts
fact_caching_timeout = 3600
```

```bash
sudo install -d -m 0750 -o ansible -g ansible /var/cache/ansible/facts
```

`gathering = smart` 表示"有缓存就用缓存，没有就收集"。`fact_caching_timeout = 3600` 一小时过期。

**注意**：缓存会导致"机器重启后 facts 变了但 Ansible 还用旧值"。如果是"IP 变了"这类事实，必须手动清缓存：

```bash
ansible -i inventories/prod/hosts.yml web01.example.com -m setup --flush-cache
```

### 9.4 分批与滚动发布的完整模式

```yaml
- name: Rolling restart web tier
  hosts: web
  gather_facts: false
  serial: 2                    # 2 台一批
  max_fail_percentage: 0
  any_errors_fatal: true
  order: sorted                # 按 inventory 顺序，不随机

  tasks:
    - name: Drain from load balancer
      delegate_to: "{{ lb_host }}"
      ansible.builtin.shell:
        cmd: |
          set -o pipefail
          echo "set server be_web/{{ inventory_hostname }} state drain" \
            | socat stdio /var/run/haproxy/admin.sock
      changed_when: true
      tags: [restart]

    - name: Wait for connections to drain
      delegate_to: "{{ lb_host }}"
      ansible.builtin.shell:
        cmd: |
          set -o pipefail
          for i in $(seq 1 30); do
            n=$(echo "show stat" | socat stdio /var/run/haproxy/admin.sock \
                 | awk -F, -v h="{{ inventory_hostname }}" '$2==h {print $5}')
            [ "${n:-0}" -eq 0 ] && exit 0
            sleep 2
          done
          exit 1
      register: drain_wait
      until: drain_wait.rc == 0
      retries: 3
      delay: 5
      changed_when: false
      tags: [restart]

    - name: Restart application
      ansible.builtin.service:
        name: app
        state: restarted
      tags: [restart]

    - name: Wait for app health
      ansible.builtin.uri:
        url: "http://{{ ansible_host }}:8080/actuator/health"
        status_code: 200
      register: app_health
      until: app_health.status == 200
      retries: 30
      delay: 5
      changed_when: false
      tags: [restart]

    - name: Bring back into rotation
      delegate_to: "{{ lb_host }}"
      ansible.builtin.shell:
        cmd: |
          set -o pipefail
          echo "set server be_web/{{ inventory_hostname }} state ready" \
            | socat stdio /var/run/haproxy/admin.sock
      changed_when: true
      tags: [restart]

    - name: Let new instance warm up
      ansible.builtin.pause:
        seconds: 30
      tags: [restart]
```

**为什么需要"最后再等 30 秒"**：JVM 应用刚启动时 JIT 还没预热，直接接到满量流量会响应变慢甚至超时。等 30 秒让预热探针跑一会儿，或者用 HAProxy 的 `slowstart` 参数（见 HAProxy 那篇）。

**order: sorted**：默认按 inventory 顺序，但 `serial` 分批时组内顺序可能变。显式指定 `sorted`（按主机名字典序）更稳。

**这是高可用变更的标准模板**，每一步都有验证，失败会停在原地而不是继续往下。`any_errors_fatal: true` 保证一台失败整批停止。

**风险与回滚**：

- 如果 Drain 成功但 Restart 失败，机器处于"已摘除 + 服务挂了"状态。此时流量不受影响（已被摘除），但要人工介入。
- 回滚方式：echo "set server be_web/<host> state ready" | socat stdio $SOCK 把它加回去（前提是服务已恢复）。
- 执行窗口：必须避开业务高峰，且提前通知。数据库相关的滚动变更要额外确认主从状态。

## 十、安全：Ansible 本身就是一个高权限系统

Ansible 用 SSH 登录所有机器，且通常有 root 权限。这意味着 Ansible 控制节点被攻破 = 全部机器被攻破。安全设计不能省。

### 10.1 连接层的安全

**用专用账号 + 受限 sudo，不要直接 root 登录。**

```bash
# 在被管节点创建 ansible 用户
sudo useradd -m -s /bin/bash ansible
sudo install -d -m 0700 -o ansible -g ansible /home/ansible/.ssh

# 授权公钥
printf '%s\n' "${ANSIBLE_PUBKEY}" | sudo tee /home/ansible/.ssh/authorized_keys
sudo chown ansible:ansible /home/ansible/.ssh/authorized_keys
sudo chmod 0600 /home/ansible/.ssh/authorized_keys
```

```bash
# sudo 白名单（原则：只给具体命令，不给 ALL）
# /etc/sudoers.d/ansible
Cmnd_Alias ANSIBLE_CMDS = /usr/bin/systemctl, /usr/bin/dnf, /usr/bin/apt-get, \
                          /usr/sbin/nginx, /usr/bin/tee /etc/nginx/conf.d/*, \
                          /usr/bin/find, /usr/bin/tar
ansible ALL=(root) NOPASSWD: ANSIBLE_CMDS
Defaults:ansible !requiretty
Defaults:ansible env_keep += "LANG LC_*"
```

实际项目中，纯白名单很难维护（Ansible 要执行临时脚本）。折中方案是限制"哪些机器能被 ansible 用户 sudo"和"从哪些源 IP 能连"：

```bash
# 限制 ansible 用户只能从控制节点登录
# /etc/ssh/sshd_config.d/ansible.conf
Match User ansible
    AllowUsers ansible@10.0.1.0/24
    PermitRootLogin no
    PasswordAuthentication no
```

```bash
sudo sshd -t && sudo systemctl reload sshd
```

**authorized_keys 加限制。**

```bash
# 在 authorized_keys 前加选项，限制来源和命令
from="10.0.1.0/24",no-agent-forwarding,no-port-forwarding,no-pty,no-user-rc ssh-ed25519 AAAA...
```

`no-pty` 会跟某些模块冲突（需要 TTY 的），测试后再上。

### 10.2 密钥管理

- 控制节点的私钥文件 chmod 600，属主是自己的运维账号，不要放共享目录。
- 用 ssh-agent + AddKeysToAgent 而不是裸文件，减少私钥落盘时间。
- 不要用同一个密钥访问所有环境。测试和生产分开，泄露时能限制影响面。
- 定期轮换密钥，轮换流程要能在不中断服务的情况下完成（先加新key，再删旧key）。

```bash
# 密钥轮换流程
# 1. 生成新密钥
ssh-keygen -t ed25519 -C "ansible-2026q2" -f ~/.ssh/ansible_2026q2 -N ""

# 2. 用旧密钥分发新公钥（Ansible 做这件事）
ansible-playbook -i inventories/prod/hosts.yml security/rotate-key.yml \
  -e new_pubkey_file=~/.ssh/ansible_2026q2.pub

# 3. 用新密钥验证
ansible -i inventories/prod/hosts.yml prod -m ping \
  --private-key ~/.ssh/ansible_2026q2

# 4. 确认全部可用后，删除旧公钥
ansible-playbook -i inventories/prod/hosts.yml security/rotate-key.yml \
  -e remove_old=true
```

### 10.3 Playbook 层的安全

**不要回传整个 register 结果**。`register` 会把模块返回值全部写入日志，如果里面有敏感信息（比如 `mysql_user` 返回的密码），就泄露了。

```yaml
# 危险
- name: Create user
  community.mysql.mysql_user:
    name: app
    password: "{{ app_pw }}"
  register: user_result      # 结果里有 password 字段，会进日志

# 安全
- name: Create user
  community.mysql.mysql_user:
    name: app
    password: "{{ app_pw }}"
  no_log: true
  register: user_result
```

**no_log 与 debug 冲突**：如果开了 `no_log`，`debug: var=user_result` 会输出 `VARIABLE IS NOT DEFINED!`，因为 Ansible 把值替换成了占位符。这是设计如此，不是 bug。

**禁止在生产 Playbook 里出现 shell + 用户输入**：

```yaml
# 危险：命令注入
- name: Delete old files
  ansible.builtin.shell: "rm -rf /data/{{ user_input }}/*"

# 安全：用模块
- name: Delete old files
  ansible.builtin.file:
    path: "/data/{{ user_input }}"
    state: absent
```

如果非要用 shell，用 `argv` 形式避免注入：

```yaml
- name: Delete old files safely
  ansible.builtin.command:
    argv:
      - rm
      - -rf
      - "/data/{{ user_input }}"
```

**argv 不经过 shell 解析，参数不会被拆词或执行**。

**批量删除必须加保护**：

```yaml
- name: Remove old release directories
  ansible.builtin.file:
    path: "{{ item }}"
    state: absent
  loop: "{{ old_releases }}"
  when:
    - item is match('^/data/releases/\\d{8}-\\d{6}$')
    - item != current_release
  vars:
    current_release: "/data/releases/{{ app_release }}"
```

`match` 正则校验路径格式，`item != current_release` 防误删当前版本。**这是"批量删除"必须有的双重保护。**

### 10.4 变更管控

生产变更必须有的：

| 要求 | 落地方式 |
| --- | --- |
| 权限控制 | 控制节点访问需堡垒机；Ansible 账号与应用发布账号分离 |
| 执行窗口 | 变更单里声明时间窗，CI 里加时间检查 |
| 灰度 | serial + --limit 分批 |
| 可追溯 | log_path + Git 提交记录 + 变更单关联 |
| 可回滚 | 每个变更都有对应的回滚 Playbook |
| 审批 | MR 必须两人 review，涉及生产的变更需要主管 approve |

CI 里加时间窗检查的示例：

```bash
#!/usr/bin/env bash
# check_window.sh - 生产变更时间窗校验
set -euo pipefail

ALLOWED_START=1     # 01:00
ALLOWED_END=5       # 05:00
hour=$(date +%-H)

if [ "${hour}" -lt "${ALLOWED_START}" ] || [ "${hour}" -ge "${ALLOWED_END}" ]; then
    echo "ERROR: production change outside allowed window (${ALLOWED_START}:00-${ALLOWED_END}:00), now $(date '+%H:%M')"
    echo "Set CHANGETICKET=<id> and use --force-window only with approval"
    exit 1
fi
echo "Window OK"
```

## 十一、测试：Molecule 与容器验证

### 11.1 为什么必须测

角色不测会有两个后果：一是"只在某台机器上跑过"，二是"改了 A 影响了 B 不知道"。Molecule 能在容器里起一个干净系统，跑一遍 converge、再跑一遍检查幂等、再跑一次 verify。

### 11.2 Molecule 配置

```bash
python3 -m pip install 'molecule>=25' 'molecule-plugins[docker]'
cd roles/nginx
molecule init scenario --driver-name docker
```

`roles/nginx/molecule/default/molecule.yml`：

```yaml
---
dependency:
  name: galaxy
  options:
    ignore-certs: false
    ignore-role-url: false

driver:
  name: docker

platforms:
  - name: rockylinux9
    image: geerlingguy/docker-rockylinux9-ansible:latest
    pre_build_image: true
    privileged: true
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    command: /usr/sbin/init
    capabilities:
      - SYS_ADMIN
    tmpfs:
      - /run
      - /tmp

  - name: ubuntu2404
    image: geerlingguy/docker-ubuntu2404-ansible:latest
    pre_build_image: true
    privileged: true
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    command: /usr/sbin/init
    capabilities:
      - SYS_ADMIN
    tmpfs:
      - /run
      - /tmp

provisioner:
  name: ansible
  config_options:
    defaults:
      callbacks_enabled: profile_tasks
      interpreter_python: auto_silent
  inventory:
    group_vars:
      all:
        nginx_port: 80
        nginx_worker_connections: 4096
        nginx_upstreams:
          - name: app
            method: least_conn
            keepalive: 32
            servers:
              - "127.0.0.1:9000"
        nginx_sites:
          - name: test
            server_name: test.local
            upstream: app
            locations:
              - path: /
                proxy: true
  playbooks:
    converge: converge.yml
    verify: verify.yml

verifier:
  name: ansible

scenario:
  test_sequence:
    - dependency
    - destroy
    - syntax
    - create
    - prepare
    - converge
    - idempotence
    - side_effect
    - verify
    - destroy
```

**privileged: true + /sys/fs/cgroup 挂载**是 systemd 容器能跑的前提。没有这两个，`systemd` 起不来，`service` 模块会失败。

`pre_build_image: true` 表示用现成镜像，不在本地 build，快很多。

### 11.3 converge 与 verify

`roles/nginx/molecule/default/converge.yml`：

```yaml
---
- name: Converge
  hosts: all
  become: true
  gather_facts: true
  roles:
    - role: nginx
```

`roles/nginx/molecule/default/verify.yml`：

```yaml
---
- name: Verify
  hosts: all
  become: true
  gather_facts: false

  tasks:
    - name: Check nginx binary exists
      ansible.builtin.stat:
        path: /usr/sbin/nginx
      register: nginx_bin

    - name: Assert nginx installed
      ansible.builtin.assert:
        that:
          - nginx_bin.stat.exists
          - nginx_bin.stat.executable

    - name: Check nginx is running
      ansible.builtin.service_facts:

    - name: Assert nginx service active
      ansible.builtin.assert:
        that:
          - ansible_facts.services['nginx.service'].state == 'running'
          - ansible_facts.services['nginx.service'].status == 'enabled'

    - name: Check config is valid
      ansible.builtin.command:
        cmd: nginx -t
      register: nginx_test
      changed_when: false
      failed_when: nginx_test.rc != 0

    - name: Curl local endpoint
      ansible.builtin.uri:
        url: "http://127.0.0.1/healthz"
        status_code: 200

    - name: Check config header marks ansible management
      ansible.builtin.slurp:
        src: /etc/nginx/nginx.conf
      register: conf

    - name: Assert managed header present
      ansible.builtin.assert:
        that:
          - "'Managed by Ansible' in (conf.content | b64decode)"
```

`verify` 检查的不是"任务有没有跑"，而是"系统状态对不对"。这两者差别很大：任务可能成功但结果不对（比如模板渲染出错但仍是合法 YAML）。

### 11.4 幂等测试的自动化

Molecule 的 `idempotence` 步骤会自动跑第二遍 converge 并检查 `changed=0`。如果失败，输出会告诉你哪个任务不幂等：

```
INFO     Idempotence completed successfully.
```

或者：

```
CRITICAL Idempotence test failed because of the following tasks:
* [rockylinux9] => nginx : Deploy nginx.conf
```

这条信息直接指出问题任务，比手工对比日志快得多。

**注意**：`idempotence` 对 `command`/`shell` 任务会误报（默认老是 changed），所以这些任务必须显式 `changed_when`。

### 11.5 测试覆盖的判断标准

一个角色的测试至少要覆盖：

- 全新安装（空容器 converge）。
- 二次执行幂等（idempotence 步骤）。
- 版本升级（旧版本容器上的 converge）。
- 配置变更后服务正常（verify 里 curl）。
- 多 OS（至少 EL9 + Ubuntu 24.04）。

不需要覆盖的：数据迁移、跨机器依赖（这些用集成环境测）。

## 十二、Ansible 与 CI/CD 的集成

### 12.1 一条可用的流水线

```yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - deploy-staging
  - deploy-prod

variables:
  ANSIBLE_FORCE_COLOR: "1"
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

.ansible_base:
  image: python:3.12-slim
  before_script:
    - pip install --quiet 'ansible-core==2.21.*' 'ansible-lint>=25' 'yamllint'
    - ansible-galaxy collection install -r requirements.yml --force
    - ansible --version

lint:
  extends: .ansible_base
  stage: lint
  script:
    - yamllint -c .yamllint.yml .
    - ansible-lint --profile production playbooks/ roles/
    - ansible-playbook -i inventories/test/hosts.yml site.yml --syntax-check

idempotency:
  extends: .ansible_base
  stage: test
  services:
    - docker:dind
  before_script:
    - !reference [.ansible_base, before_script]
    - pip install --quiet 'molecule>=25' 'molecule-plugins[docker]'
  script:
    - cd roles/nginx && molecule test

deploy-staging:
  extends: .ansible_base
  stage: deploy-staging
  script:
    - ansible-playbook -i inventories/test/hosts.yml site.yml --diff
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

deploy-prod:
  extends: .ansible_base
  stage: deploy-prod
  script:
    - bash ci/check_window.sh
    - ansible-playbook -i inventories/prod/hosts.yml site.yml --diff --limit "web:&prod"
  environment:
    name: production
  when: manual
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
      allow_failure: false
  variables:
    ANSIBLE_VAULT_PASSWORD_FILE: "$VAULT_PASS_FILE"
```

三个门禁点：

1. lint：ansible-lint --profile production 会检查变量未使用、任务命名、权限模式、become 使用等。第一次跑肯定一堆问题，逐步修。
2. idempotency：Molecule 在容器里验证幂等。
3. deploy-prod：when: manual + check_window.sh 双重门禁。

### 12.2 ansible-lint 必改的规则

`--profile production` 会启用较严格的规则。常见报警与修法：

| 规则 | 现象 | 修法 |
| --- | --- | --- |
| name[casing] | 任务名首字母小写 | 改大写 |
| name[missing] | 任务没有 name | 加 name |
| yaml[truthy] | 用了 yes/no | 改 true/false |
| risky-file-permissions | file/copy 没写 mode | 显式写 mode |
| no-changed-when | command/shell 没有 changed_when | 加 changed_when |
| command-instead-of-module | 用 shell 做模块能做的事 | 换模块 |
| var-naming[no-role-prefix] | 角色变量没前缀 | 加前缀 |
| fqcn[action-core] | 用了 service: 而不是 ansible.builtin.service: | 补全 FQCN |
| latest[git] | version: HEAD | 固定 commit |
| no-handler | 用 when: result.changed 代替 handler | 改成 notify |

`fqcn` 这条建议全量改，因为 `ansible-lint` 和未来版本都会要求。批量替换要注意 `block`/`when` 里的变量不受影响。

### 12.3 一次真实的灰度发布流程

```
1. 合并 MR → CI lint + idempotency 通过
2. 自动部署到 staging（3 台）
3. 观察 staging 15 分钟（人工确认或自动化冒烟）
4. 手工触发 deploy-prod，输入批次（默认 20%）
5. 每批完成后自动跑冒烟检查（curl 接口 + 日志无 ERROR）
6. 任一批失败 → 自动停止 → 告警 → 人工决策是否回滚
7. 全量完成后观察 30 分钟
```

第 5 步的冒烟脚本：

```bash
#!/usr/bin/env bash
# smoke.sh - 批量部署后的冒烟检查
set -uo pipefail

HOSTS_FILE="$1"
FAIL=0

while read -r host; do
    [ -z "${host}" ] && continue
    code=$(curl -o /dev/null -s -m 5 -w '%{http_code}' "http://${host}/healthz") || code="000"
    if [ "${code}" != "200" ]; then
        echo "FAIL ${host} healthz=${code}"
        FAIL=$((FAIL + 1))
    else
        echo "OK   ${host}"
    fi
done < "${HOSTS_FILE}"

if [ "${FAIL}" -gt 0 ]; then
    echo "smoke failed: ${FAIL} hosts"
    exit 1
fi
echo "smoke passed"
```

`set -uo pipefail` 不加 `-e`，因为要让循环跑完收集所有失败。退出码靠最后手动 `exit 1`。

## 十三、常见反模式清单

写 Playbook 三年，最容易积累的反模式：

| 反模式 | 后果 | 正确做法 |
| --- | --- | --- |
| 一个 3000 行的 site.yml | 改一行跑全量，风险大 | 拆角色 + tags |
| 用 shell 做模块能做的事 | 不幂等、不可读 | 用 copy/file/template/lineinfile |
| 变量名不带前缀 | 冲突（见 8.1） | <role>_<name> |
| -e 传业务参数 | 环境不一致 | 放 group_vars |
| when: false 调试完忘删 | 配置不生效 | 用 --limit / tags 调试 |
| 不写 changed_when | 幂等测试失败 | 所有 command/shell 都写 |
| become: true 全局 + 所有任务 | 权限过大 | 只在需要的任务上 become |
| handler 里写业务逻辑 | 行为不可预测 | handler 只做 reload/restart |
| 用 hosts: all 跑危险任务 | 波及面失控 | 明确组名 + --limit |
| 生产直接 -u root | 审计无法追溯 | 专用账号 + sudo |
| 忽略 --check 的局限 | 假通过（见 8.2） | 容器内真实执行做幂等测试 |
| serial 不配 | 500 台同时变更 | 按服务重要性配 10%~20% |
| 不设 max_fail_percentage | 失败继续跑完 | 按容忍度设置 |
| 明文密码写在 vars | 泄露 | Vault 或外部 KMS |
| register 不加 no_log | 敏感数据进日志 | 敏感任务加 no_log |

## 十四、总结

Ansible 的核心价值不是"批量执行命令"，而是"声明系统应该是什么样，然后收敛到那个样子"。这个价值只有在幂等性、可测试性、变量分层三件事都成立时才能兑现。

如果要我给出一个实施顺序，会是：

1. 先把 `ansible.cfg` 调对（pipelining、forks、log_path、host_key_checking、become）。
2. 建立目录结构和变量规范（前缀、defaults、环境分层）。
3. 把最痛的那个服务角色化，用它跑通"角色 → 测试 → 灰度"的完整链路。
4. 把 CI 门禁加上（lint + idempotency），让规范变成强制。
5. 再逐步迁移其他服务。

不要一次全上。一个能稳定运行、每周被放心执行的 5 个角色，价值远高于 50 个没人敢跑的角色。

最后提醒三件最容易出事的事：

- `--check` 不是安全网，它对包管理和命令执行是假的。真正的安全网是容器里的真实执行 + 幂等测试。
- 变量冲突是隐形杀手，变量名一律加角色前缀，部署脚本第一行加 assert。
- `serial` 和 `max_fail_percentage` 是爆炸半径的控制阀，生产变更必须配。

## 附：命令速查

```bash
# ---- 版本与配置 ----
ansible --version
ansible-config dump --only-changed
ansible-config list | grep -A5 'forks'

# ---- inventory ----
ansible-inventory -i inventories/prod/hosts.yml --graph
ansible-inventory -i inventories/prod/hosts.yml --host web01.example.com
ansible-inventory -i inventories/prod/hosts.yml --list --yaml | head -50

# ---- 连通性 ----
ansible -i inventories/prod/hosts.yml prod -m ping --one-line
ansible -i inventories/prod/hosts.yml web -m setup -a 'gather_subset=!all,os_family'

# ---- Playbook ----
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --list-tasks
ansible-playbook site.yml --list-tags
ansible-playbook site.yml --list-hosts
ansible-playbook site.yml --check --diff
ansible-playbook site.yml --limit 'web01:web02'
ansible-playbook site.yml --start-at-task "Deploy nginx.conf"
ansible-playbook site.yml --tags config --skip-tags dangerous
ansible-playbook site.yml -vvv 2>&1 | less

# ---- 角色 ----
ansible-galaxy init roles/newrole
ansible-galaxy role install -r requirements.yml -p roles
ansible-galaxy collection install -r requirements.yml --force
ansible-galaxy collection list

# ---- Vault ----
ansible-vault create inventories/prod/group_vars/all/vault.yml
ansible-vault edit inventories/prod/group_vars/all/vault.yml
ansible-vault view inventories/prod/group_vars/all/vault.yml
ansible-vault encrypt_string 'secret' --name 'vault_x'
ansible-vault rekey inventories/prod/group_vars/all/vault.yml

# ---- 测试 ----
molecule test -s default
molecule converge
molecule verify
molecule login -h rockylinux9
molecule destroy

# ---- lint ----
ansible-lint --profile production playbooks/ roles/
yamllint -c .yamllint.yml .
```

### 关键模块速查

| 场景 | 模块 | 关键参数 |
| --- | --- | --- |
| 装包 | dnf / apt / package | state=present / name 列表 |
| 服务 | service / systemd | state=started/reloaded、enabled |
| 文件 | file | state=directory/absent、mode |
| 复制 | copy | src/dest/mode/validate/backup |
| 模板 | template | 同 copy，加 src 为 .j2 |
| 改行 | lineinfile | regexp 必须写 |
| 改块 | blockinfile | marker 必须写 |
| 下载 | get_url | checksum 必须写 |
| 解包 | unarchive | creates 必须写 |
| 用户 | user | state、groups、append |
| 授权 | authorized_key | exclusive |
| 定时 | cron | name 必须写（否则无法删除） |
| 内核参数 | sysctl | state=present、reload |
| 防火墙 | ansible.posix.firewalld | permanent + immediate |
| 命令 | command / shell | changed_when、creates/removes |
| 断言 | assert | that、fail_msg |
| 调试 | debug | var/msg |
| 事实 | set_fact | cacheable=true 跨 play 保留 |
| 成功判断 | uri | status_code、retries/until |
