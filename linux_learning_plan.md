# Linux 深入学习路线

---

## 一、前置条件

- 能熟练使用基本的 Linux 命令行（cd / ls / mkdir / rm / mv / cp / grep / find）
- 理解 ssh 远程登录
- 已掌握 Docker 基础（会用容器，理解容器是进程隔离的产物）
- 有一台可用的 Linux 机器（虚拟机或云服务器均可）

---

## 二、学习路线总览

```
┌───────────────┐    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ 第1周         │───▶│ 第2周         │───▶│ 第3周         │───▶│ 第4周         │
│ 用户空间深入  │    │ 系统管理      │    │ 网络          │    │ 性能与排障    │
└───────────────┘    └───────────────┘    └───────────────┘    └───────────────┘
```

---

## 三、阶段详细规划

### 第1周：用户空间深入

**目标：理解文件系统本质、进程模型、Shell 编程进阶**

#### Day 1：文件系统底层

| 主题 | 内容 |
|------|------|
| inode | 文件名与数据分离：inode 存储元数据（权限、大小、时间戳、数据块位置） |
| `stat` / `ls -i` | 查看文件的 inode 号和信息 |
| 硬链接 | `ln src dst` — 同一个 inode 的多个名字，**跨文件系统无效** |
| 软链接 | `ln -s src dst` — 指向路径的快捷方式，**可以跨文件系统** |
| inode 耗尽 | 磁盘有空间但无法创建文件 → 可能是 inode 用完了 |

**练习：**
```bash
# 创建文件，观察 inode
echo "hello" > a.txt
ls -li a.txt          # 第1列是 inode 号
stat a.txt

# 硬链接 vs 软链接
ln a.txt hard.txt     # 硬链接，inode 相同
ln -s a.txt soft.txt  # 软链接，inode 不同
ls -li a.txt hard.txt soft.txt

# 删除原文件后分别读取
rm a.txt
cat hard.txt  # 还能读（数据还在）
cat soft.txt  # No such file（链接断了）
```

#### Day 2：文件权限与 ACL

| 主题 | 说明 |
|------|------|
| rwx 含义 | 文件：读/写/执行；目录：列出/创建文件/进入 |
| setuid / setgid | 执行时以文件属主/属组权限运行（`passwd` 命令的原理） |
| sticky bit | 目录下只有文件所有者能删除（`/tmp` 目录） |
| umask | 创建文件和目录时的默认权限掩码 |
| ACL | `setfacl` / `getfacl` — 比 ugo 更细粒度的权限 |

**练习：**
```bash
# 理解 setuid
ls -l $(which passwd)      # -rwsr-xr-x，s 表示 setuid
ls -l $(which sudo)        # ---s--x--x

# 理解 sticky bit
ls -ld /tmp                # drwxrwxrwt，t 表示 sticky bit

# umask 实验
umask 022                  # 新建文件 644，目录 755
touch f1 && ls -l f1
umask 077                  # 新建文件 600，目录 700
touch f2 && ls -l f2

# ACL
setfacl -m u:alice:rw myfile
getfacl myfile
```

#### Day 3：进程深入

| 主题 | 说明 |
|------|------|
| `/proc/<pid>/` | 进程的运行时信息：cmdline、environ、fd、maps、status |
| fork / exec | fork 复制进程，exec 替换程序镜像（Docker entrypoint 的执行过程） |
| 信号 | `kill -l` 列出所有信号；SIGTERM(15) vs SIGKILL(9) vs SIGHUP(1) |
| 孤儿进程 vs 僵尸进程 | 父进程先死 → init 收养；子进程死了父进程没 wait → 僵尸 |
| daemon | 守护进程的特征：脱离终端、setsid、重定向 fd |

**练习：**
```bash
# 探索 /proc
ls /proc/self/fd           # 当前进程的文件描述符
cat /proc/$$/status        # 当前 shell 的进程状态
cat /proc/$$/limits        # 资源限制

# 信号实验
sleep 100 &
kill -TERM %1   # 优雅停止（可被捕获）
sleep 100 &
kill -KILL %1   # 强制杀掉（不可捕获）

# 制造一个僵尸进程观察
python3 << 'EOF'
import os, time
pid = os.fork()
if pid == 0:
    print(f"child {os.getpid()} exiting")
    os._exit(0)
else:
    print(f"parent {os.getpid()} sleeping, child {pid} will be zombie")
    time.sleep(60)  # 这期间子进程是僵尸
EOF
# 另开终端：ps aux | grep Z
```

#### Day 4：管道、重定向、文件描述符

| 主题 | 说明 |
|------|------|
| 3 个标准 fd | 0(stdin) / 1(stdout) / 2(stderr) |
| 重定向运算符 | `>` (覆盖)、`>>` (追加)、`2>` (stderr)、`&>` (stdout+stderr) |
| Here Document | `<< 'EOF' ... EOF` |
| 管道 | `\|` 连接 stdout→stdin，重点：每个管道符启动一个子进程 |
| 命名管道 (FIFO) | `mkfifo mypipe` — 不相关的进程也能通过路径通信 |
| `/dev/null` | 比特黑洞，丢弃输出 |

**练习：**
```bash
# 同时输出到终端和文件
echo "hello" | tee output.txt

# 交换 stdout 和 stderr
some_cmd 3>&1 1>&2 2>&3

# 命名管道实验
mkfifo mypipe
# Terminal A:
cat mypipe
# Terminal B:
echo "Hello from B" > mypipe

# Process substitution（bash 特性）
diff <(ls dir1) <(ls dir2)
```

#### Day 5：Shell 编程进阶

| 主题 | 说明 |
|------|------|
| 变量 | `${var:-default}` / `${var:+alt}` / `${var:?error}` / `${var#pattern}` |
| 函数 | `func() { ... }`，局部变量需要 `local` |
| 错误处理 | `set -e` (遇错即停) / `set -u` (未定义变量报错) / `set -o pipefail` (管道的任何命令失败都算失败) |
| trap | `trap 'cleanup' EXIT` —— 脚本退出时执行清理 |
| 数组 | `arr=(a b c)` / `${arr[@]}` / `${#arr[@]}` |
| 调试 | `bash -x script.sh` / `set -x` —— 打印每条执行命令 |

**练习：** 写一个健壮的备份脚本：

```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="${BACKUP_DIR:-/tmp/backups}"
RETENTION_DAYS="${RETENTION_DAYS:-7}"

cleanup() {
    echo "Cleaning up temp files..."
    rm -f /tmp/backup_*.tmp
}
trap cleanup EXIT

backup_dir() {
    local src="$1"
    local name="$(basename "$src")"
    local target="${BACKUP_DIR}/${name}_$(date +%Y%m%d_%H%M%S).tar.gz"
    echo "Backing up $src -> $target"
    tar -czf "$target" -C "$(dirname "$src")" "$name"
}

# 主逻辑
mkdir -p "$BACKUP_DIR"
backup_dir "/home/$USER"
find "$BACKUP_DIR" -name "*.tar.gz" -mtime "+$RETENTION_DAYS" -delete
echo "Done. $(ls "$BACKUP_DIR" | wc -l) backups stored."
```

#### Day 6：文本处理工具链

| 工具 | 典型用途 |
|------|----------|
| `sed` | 流编辑器：替换 `s/old/new/g`、删除行 `Nd`、提取范围 `/start/,/end/p` |
| `awk` | 字段处理：`awk '{print $1, $NF}'`、条件过滤、聚合 `sum+=$3` |
| `sort` / `uniq` | 排序去重 `sort \| uniq -c \| sort -rn` |
| `xargs` | 将 stdin 转为命令行参数 |
| `jq` | 处理 JSON 数据 |

**练习：** 一条命令完成：分析 Nginx access.log，找出 Top 10 访问 IP

```bash
awk '{print $1}' /var/log/nginx/access.log \
  | sort | uniq -c | sort -rn | head -10
```

用 jq 分析 Docker 容器信息：
```bash
docker inspect $(docker ps -q) | jq '.[] | {Name: .Name, IP: .NetworkSettings.IPAddress}'
```

#### Day 7：第一周综合练习

写一个系统巡检脚本，收集并格式化输出以下信息：

```
┌─ 系统信息 ─────────────────┐
│ 主机名：xxx                │
│ 内核版本：xxx              │
│ 运行时间：xxx              │
├─ 资源使用 ─────────────────┤
│ CPU 负载：xxx              │
│ 内存：已用/总量 (xx%)      │
│ 磁盘：挂载点 使用率        │
│ inode 使用率最高的分区     │
├─ 进程 Top 5 ───────────────┤
│ CPU 最高：...              │
│ 内存最高：...              │
├─ 网络 ─────────────────────┤
│ 监听端口：...              │
│ 活跃连接数：...            │
└─ 安全 ─────────────────────┘
│ 失败登录：xxx              │
└────────────────────────────┘
```

---

### 第2周：系统管理

**目标：理解 systemd、用户管理、存储管理**

#### Day 8：systemd 原理与 Unit 文件

| 主题 | 说明 |
|------|------|
| systemd 是什么 | PID 1，管理系统启动和服务的 init 系统 |
| Unit 类型 | service / socket / timer / mount / target / slice |
| Unit 文件位置 | `/usr/lib/systemd/system/` (包管理) vs `/etc/systemd/system/` (管理员) |
| 关键指令 | `ExecStart` / `ExecStop` / `Restart` / `User` / `Environment` |

**练习：** 为一个 Python Web 应用编写 systemd unit：

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Python Web App
After=network.target

[Service]
Type=simple
User=www
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/venv/bin/python app.py
Restart=on-failure
RestartSec=5
Environment="APP_ENV=production"
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
sudo systemctl status myapp
journalctl -u myapp -f
```

#### Day 9：systemd Timer 与日志

| 主题 | 说明 |
|------|------|
| Timer | systemd 替代 cron：`OnCalendar=daily` / `OnUnitActiveSec=1h` |
| journalctl | `-u <unit>` / `-b` (本次启动) / `--since "1 hour ago"` / `-f` |
| 日志持久化 | `/etc/systemd/journald.conf` → `Storage=persistent` |
| 日志清理 | `journalctl --vacuum-time=7d` / `--vacuum-size=500M` |

**练习：** 用 systemd timer 替代 cron，定时运行 Day 7 的巡检脚本：

```ini
# /etc/systemd/system/syscheck.service
[Unit]
Description=System Health Check

[Service]
Type=oneshot
ExecStart=/usr/local/bin/syscheck.sh

# /etc/systemd/system/syscheck.timer
[Unit]
Description=Run system check hourly

[Timer]
OnCalendar=hourly
Persistent=true   # 如果上次错过了，启动后立即补跑

[Install]
WantedBy=timers.target
```

#### Day 10：用户、组与 sudo

| 主题 | 说明 |
|------|------|
| `/etc/passwd` | 用户账号信息（login、uid、gid、home、shell） |
| `/etc/shadow` | 加密密码和过期策略 |
| `/etc/group` | 组信息 |
| `useradd` / `usermod` / `userdel` | 用户生命周期 |
| sudo 配置 | `/etc/sudoers` (用 `visudo` 编辑！) |
| wheel 组 | 管理员的传统组名 |

**练习：**
```bash
# 创建应用专用用户（不分配 shell，不能登录）
sudo useradd -r -s /usr/sbin/nologin -d /opt/myapp myapp

# 为应用用户配置 sudo 权限（只允许重启特定服务）
sudo visudo -f /etc/sudoers.d/myapp
# 内容：
# %deployers ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp
```

#### Day 11：PAM 认证

| 主题 | 说明 |
|------|------|
| PAM 是什么 | Pluggable Authentication Modules，认证的统一接口 |
| 配置文件 | `/etc/pam.d/` —— 每个服务一个文件（sshd、login、sudo） |
| 控制类型 | required / requisite / sufficient / optional |
| 常用模块 | `pam_unix.so` (密码) / `pam_tally2.so` (失败锁定) / `pam_limits.so` (资源限制) |
| `/etc/security/limits.conf` | 通过 PAM 控制进程资源限制 |

**练习：** 配置 SSH 登录失败 5 次后锁定 10 分钟：

```
# /etc/pam.d/sshd 中添加：
auth required pam_tally2.so deny=5 unlock_time=600 onerr=fail
```

#### Day 12：存储管理

| 主题 | 说明 |
|------|------|
| 分区类型 | MBR (传统，最大 2TB) vs GPT (现代，有备份分区表) |
| 文件系统 | ext4 (稳定首选) / XFS (大文件好) / Btrfs / ZFS |
| `fdisk` / `parted` | 分区工具 |
| `mkfs.ext4` | 创建文件系统 |
| `/etc/fstab` | 开机自动挂载 |
| `lsblk` / `blkid` | 查看块设备和 UUID |
| LVM | PV → VG → LV，灵活的卷管理：在线扩容、快照 |

**练习：** 创建并挂载一个文件系统（用空文件模拟）：

```bash
# 创建一个 1GB 的虚拟磁盘文件
dd if=/dev/zero of=/tmp/vdisk.img bs=1M count=1024
# 格式化并挂载
sudo losetup -f /tmp/vdisk.img       # 绑定到 loop 设备
sudo losetup -a                       # 查看
sudo mkfs.ext4 /dev/loop0
sudo mount /dev/loop0 /mnt/vdisk
df -h /mnt/vdisk

# LVM 练习（如果有空磁盘）
sudo pvcreate /dev/sdb
sudo vgcreate vg_data /dev/sdb
sudo lvcreate -L 5G -n lv_mysql vg_data
sudo mkfs.ext4 /dev/vg_data/lv_mysql
sudo mount /dev/vg_data/lv_mysql /var/lib/mysql
```

#### Day 13：包管理深入

| 主题 | 说明 |
|------|------|
| dpkg / rpm | 底层包管理：安装 .deb/.rpm 文件 |
| apt / yum | 高层：自动处理依赖和仓库 |
| 仓库配置 | `/etc/apt/sources.list` 和 `/etc/apt/sources.list.d/` |
| 固定版本 | `apt-mark hold <pkg>` 阻止升级 |
| 编译安装 | `./configure && make && make install` — 从源码安装 |

**练习：**
```bash
# 分析包的依赖树
apt-cache depends nginx
apt-cache rdepends libssl3    # 反向：谁依赖它

# 从源码编译安装一个小工具
git clone https://github.com/sharkdp/bat.git
cd bat
# 阅读 INSTALL.md 或 README.md
cargo build --release   # 或用 make
sudo cp target/release/bat /usr/local/bin/
```

#### Day 14：第二周综合练习

用虚拟机或云服务器完成以下操作：

```
1. 创建三个用户：deployer（可 sudo systemctl restart）、app（运行应用，不可登录）、db（运行数据库，不可登录）
2. 配置 PAM：deployer 通过 SSH 登录，失败 5 次锁定 15 分钟
3. 创建一个 LVM 卷，挂载到 /data，确保开机自动挂载
4. 为应用写一个 systemd service：依赖于 /data 挂载点和网络
5. 配置 systemd timer，每天凌晨 2 点运行数据库备份脚本
6. 配置 journald 日志永久存储，限制 1GB
```

---

### 第3周：网络

**目标：理解 TCP/IP 协议栈、能用 tcpdump/iptables 调试网络问题**

#### Day 15：TCP/IP 协议栈

| 层 | 协议 | 理解 |
|----|------|------|
| 应用层 | HTTP / DNS / SSH | 你写的程序在这里 |
| 传输层 | TCP / UDP | 端口、连接、可靠传输 vs 尽力而为 |
| 网络层 | IP | 路由、寻址、分片 |
| 链路层 | Ethernet / ARP | MAC 地址、帧、同一网段内通信 |

| 概念 | 说明 |
|------|------|
| TCP 三次握手 | SYN → SYN-ACK → ACK |
| TCP 四次挥手 | FIN → ACK ← FIN → ACK |
| TIME_WAIT | 主动关闭方等待 2MSL，确保最后 ACK 被收到 |
| 拥塞控制 | 慢启动、拥塞避免、快速重传 |

**练习：** 用 `ss` 观察 TCP 状态：

```bash
ss -tan    # TCP 所有 socket，显示状态
ss -tan state time-wait   # 只看 TIME_WAIT
ss -tlp    # 正在监听的 socket + 进程名
ss -s      # 统计摘要
```

#### Day 16：ss、tcpdump 调试工具

| 工具 | 用途 |
|------|------|
| `ss` | 查看 socket 状态（现代替代 netstat） |
| `tcpdump` | 抓包：`-i eth0 port 80` / `-w file.pcap` / `-r file.pcap` |
| 过滤表达式 | `host 10.0.0.1` / `port 443` / `tcp[tcpflags] & tcp-syn != 0` |

**练习：**
```bash
# 抓取到某个 IP 的所有 HTTP 流量
sudo tcpdump -i any -A 'host example.com and port 80'

# 抓取 TCP SYN 包（观察三次握手）
sudo tcpdump -i any 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0'

# 启动一个临时 HTTP 服务，抓取访问
python3 -m http.server 8888 &
sudo tcpdump -i lo -A port 8888
curl localhost:8888
```

#### Day 17：DNS 解析链路

| 工具 | 用途 |
|------|------|
| `dig` | DNS 查询神器：`dig @8.8.8.8 example.com` |
| `/etc/resolv.conf` | DNS 解析器配置 |
| `/etc/hosts` | 本机静态解析（优先级最高，由 `/etc/nsswitch.conf` 控制） |
| 查询追踪 | `dig +trace example.com` —— 从根到权威服务器 |

**练习：**
```bash
# 追踪完整 DNS 解析链路
dig +trace www.baidu.com

# 对比不同 DNS 服务器
dig @8.8.8.8 example.com    # Google DNS
dig @1.1.1.1 example.com    # Cloudflare DNS
dig @114.114.114.114 example.com  # 国内公共 DNS

# 测试 /etc/hosts 优先级
echo "127.0.0.1 test.local" | sudo tee -a /etc/hosts
ping test.local  # 解析到 127.0.0.1
```

#### Day 18：iptables / nftables 防火墙

| 概念 | 说明 |
|------|------|
| 表与链 | filter(INPUT/OUTPUT/FORWARD) / nat(PREROUTING/POSTROUTING) / mangle |
| 规则匹配 | `-s`(源IP) / `-d`(目的IP) / `-p`(协议) / `--dport`(目的端口) / `-m state` |
| 策略 | `ACCEPT` / `DROP` / `REJECT` / `LOG` / `MASQUERADE` |
| 持久化 | `iptables-save > /etc/iptables/rules.v4` |

**练习：**
```bash
# 查看所有规则
sudo iptables -L -n -v

# 允许已建立/相关的连接（防火墙基础规则）
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 只允许特定 IP 访问 22 端口
sudo iptables -A INPUT -p tcp --dport 22 -s 192.168.1.0/24 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j DROP

# NAT：将本机 8080 端口转发到容器的 80 端口（理解 Docker port mapping 原理）
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
sudo iptables -t nat -A POSTROUTING -p tcp -d 172.17.0.2 --dport 80 -j MASQUERADE
```

#### Day 19：容器网络原理

| 技术 | 说明 | 在 Docker/K8s 中的作用 |
|------|------|------------------------|
| Network Namespace | 网络栈隔离（独立网卡、路由表、iptables） | 每个容器有自己的网络命名空间 |
| veth pair | 虚拟网线：一头在容器（eth0），一头在主机（vethXXXXX） | 容器和主机通信的基础 |
| Linux Bridge | 虚拟交换机，连接多个 veth | `docker0` 默认网桥 |
| iptables NAT | DNAT（入站端口映射）/ SNAT（出站访问外网） | `docker run -p` 的实现 |

**练习：** 手动创建网络隔离环境，理解 Docker 网络原理：

```bash
# 1. 创建两个 network namespace
sudo ip netns add ns1
sudo ip netns add ns2
ip netns list

# 2. 创建 veth pair 连接两个 ns
sudo ip link add veth1 type veth peer name veth2
sudo ip link set veth1 netns ns1
sudo ip link set veth2 netns ns2

# 3. 配置 IP 并启用
sudo ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth1
sudo ip netns exec ns1 ip link set veth1 up
sudo ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth2
sudo ip netns exec ns2 ip link set veth2 up

# 4. 测试连通
sudo ip netns exec ns1 ping 10.0.0.2

# 5. 与 Docker 网络对比
docker network inspect bridge   # 观察 "veth" 相关配置
ip link show | grep veth        # 查看主机上的 veth 接口
```

#### Day 20：防火墙与 Docker 的坑

| 问题 | 原因 | 解决 |
|------|------|------|
| 手动 iptables 规则被 Docker 绕过 | Docker 的 iptables 规则在 filter FORWARD 链中 | 在 DOCKER-USER 链添加规则 |
| `-p 8080:80` 外部能直接访问 | Docker 自动在 iptables 添加 DNAT，绕过 INPUT 链 | 监听 127.0.0.1：`-p 127.0.0.1:8080:80` |
| UFW 无效 | UFW 管理的 iptables 规则与 Docker 的规则在不同链 | 使用 `--iptables=false` 或配置 DOCKER-USER |

**练习：** 验证 Docker 端口映射的 iptables 规则：

```bash
# 启动一个容器
docker run -d --name web -p 8080:80 nginx:alpine

# 观察 Docker 添加的 iptables 规则
sudo iptables -t nat -L -n | grep 8080

# 测试：只监听 127.0.0.1
docker run -d --name web-private -p 127.0.0.1:8081:80 nginx:alpine
curl localhost:8081         # 通
curl <public_ip>:8081       # 不通（确认外网无法访问）
```

#### Day 21：第三周综合练习

安装并配置一个带防火墙的 Docker 主机：

```
1. 默认策略：INPUT DROP / FORWARD DROP / OUTPUT ACCEPT
2. 允许规则：
   - 来自内网 (192.168.0.0/24) 的 SSH (22)
   - 本机回环 (lo)
   - 已建立/相关的连接
   - 80 和 443 端口对外
3. 启动 Nginx 容器映射到 80 端口，确保外部能访问
4. 使用 tcpdump 抓取一次完整的 HTTP 请求-响应
5. 用 nsenter 直接进入容器的 network namespace 查看路由表
6. 用 dig 追踪你域名的 DNS 解析路径
```

---

### 第4周：性能与排障

**目标：掌握性能分析工具链，能独立排查系统级问题**

#### Day 22：CPU 性能分析

| 工具 | 用途 |
|------|------|
| `top` / `htop` | 实时进程监控 |
| `mpstat` | 每个 CPU 核心的使用率 |
| `pidstat` | 按进程的 CPU 统计 |
| `perf top` | 实时函数级性能热力图 |
| `perf record` + `perf report` | 采样分析 |
| 火焰图 | 从 `perf` 数据生成，直观展示调用栈热点 |

**练习：**
```bash
# 造一个 CPU 密集的程序
python3 -c "import hashlib; [hashlib.sha256(b'x'*100000).hexdigest() for _ in range(1000000)]" &

# 用多种工具定位
top -p $(pgrep -f sha256)
pidstat -p $(pgrep -f sha256) 1
sudo perf top -p $(pgrep -f sha256)
```

#### Day 23：内存分析

| 主题 | 说明 |
|------|------|
| VIRT vs RES vs SHR | VIRT=虚拟内存(含 mmap) / RES=物理内存 / SHR=共享内存 |
| OOM Killer | 内存不足时内核选择进程杀掉 |
| `dmesg` | 查看 OOM 日志：`dmesg \| grep -i oom` |
| `/proc/<pid>/smaps` | 进程的详细内存映射 |
| `pmap` | 进程内存映射快照 |
| 内存泄漏排查 | 观察 RES 持续增长，用 `valgrind` 或 `heaptrack` |

**练习：**
```bash
# 观察 OOM 过程
# 创建一个会耗尽内存的 cgroup
sudo mkdir /sys/fs/cgroup/test
echo "50M" | sudo tee /sys/fs/cgroup/test/memory.max
echo $$ | sudo tee /sys/fs/cgroup/test/cgroup.procs

# 在 shell 中尝试分配大量内存
python3 -c "x = bytearray(100 * 1024 * 1024)"  # 会触发 OOM

# 退出 cgroup
echo $$ | sudo tee /sys/fs/cgroup/cgroup.procs
```

#### Day 24：磁盘 I/O 分析

| 工具 | 用途 |
|------|------|
| `iostat -x 1` | 磁盘 I/O 统计：`%util`(忙碌度)、`await`(平均等待时间)、`r/s`/`w/s` |
| `iotop` | 按进程的 I/O 实时排名 |
| `fio` | I/O 基准测试工具 |
| `du -sh` / `ncdu` | 目录大小分析 |

**练习：**
```bash
# 用 dd 制造 I/O 负载，用 iostat 观察
# Terminal A:
dd if=/dev/zero of=/tmp/bigfile bs=1M count=1024 oflag=direct
# Terminal B:
iostat -x 1

# 用 iotop 找出哪个进程在疯狂读写
sudo iotop -o

# 磁盘基准测试
fio --name=random-rw --ioengine=libaio --rw=randrw \
    --bs=4k --size=1G --numjobs=4 --runtime=30 \
    --filename=/tmp/fio_test
```

#### Day 25：strace — 系统调用调试

| 主题 | 说明 |
|------|------|
| strace 原理 | 使用 ptrace 追踪进程发出的每个系统调用 |
| 常用用法 | `strace -p <pid>` / `strace -c <cmd>` (统计) / `strace -e trace=network <cmd>` |
| 经典场景 | 程序启动失败、配置文件找不到、权限不够 |
| `-f` | 追踪子进程 |
| `-e trace=file` | 只追踪文件相关调用 |

**练习：**
```bash
# 看清楚一个程序到底做了什么
strace curl -s http://example.com 2>&1 | head -50

# 统计系统调用耗时
strace -c python3 -c "import os; os.listdir('/')"

# 排查：程序为什么没找到配置文件
strace -e trace=file myapp 2>&1 | grep "\.conf"

# 查看 Docker 容器启动流程
strace -f docker run --rm alpine echo hello 2>&1 | grep -E "clone|exec"
```

#### Day 26：lsof — 一切皆文件

| 场景 | 命令 |
|------|------|
| 哪个进程打开了这个文件 | `lsof /var/log/syslog` |
| 这个进程打开了哪些文件 | `lsof -p <pid>` |
| 哪个进程在监听 80 端口 | `lsof -i :80` |
| 哪个进程删除了文件但还在用（磁盘空间不释放） | `lsof \| grep deleted` |
| 查看进程的工作目录 | `lsof -p <pid> \| grep cwd` |

**练习：**
```bash
# 重现"磁盘满了但 du 找不到大文件"的经典问题
# Terminal A:
exec 3> /tmp/big_hidden
dd if=/dev/zero of=/tmp/big_hidden bs=1M count=500
rm /tmp/big_hidden   # 删除文件，但 fd 3 还开着！
df -h /tmp           # 磁盘空间没恢复
du -sh /tmp          # 找不到大文件
lsof | grep deleted  # 找到了！/tmp/big_hidden (deleted)

# Terminal A 释放：
exec 3>&-            # 关闭 fd，空间恢复
```

#### Day 27：cgroup v2 资源限制

| 控制器 | 限制内容 |
|--------|----------|
| cpu | `cpu.max`：`$PERIOD $QUOTA`（如 `100000 50000`=0.5核） |
| memory | `memory.max`：硬限制；`memory.high`：软限制（限速） |
| io | `io.max`：`$DEV $R_W $IOPS` |
| pids | `pids.max`：进程数上限 |

**练习：** 用 cgroup v2 实现类似 Docker 的资源限制：

```bash
# 创建一个 cgroup（如果没有权限，用用户 slice）
CGROUP=/sys/fs/cgroup/myapp.slice
sudo mkdir -p $CGROUP

# 限制内存 256MB
echo "268435456" | sudo tee $CGROUP/memory.max

# 限制 CPU 1 核
echo "100000 100000" | sudo tee $CGROUP/cpu.max

# 限制 50 个进程
echo "50" | sudo tee $CGROUP/pids.max

# 将当前 shell 移入 cgroup
echo $$ | sudo tee $CGROUP/cgroup.procs

# 测试内存限制
python3 -c "x = bytearray(300 * 1024 * 1024)"  # 应被 OOM

# 退出 cgroup
echo $$ | sudo tee /sys/fs/cgroup/cgroup.procs
```

与 Docker 限制对比：
```bash
# Docker 的 --memory 和 --cpus 就是在容器 cgroup 里设置这些值
docker run --memory=256m --cpus=1 --name limited alpine sleep 3600
# 查看 Docker 为这个容器创建的 cgroup
ls /sys/fs/cgroup/system.slice/docker-$(docker inspect limited -f '{{.Id}}').scope/
```

#### Day 28：第四周综合 — 故障排查演练

模拟以下 6 个真实故障场景，每个独立排查并修复：

```
场景 1：服务起不来
   → 端口被占用？权限不够？依赖文件不存在？
   → 工具链：systemctl status → journalctl → ss → strace

场景 2：磁盘空间告急但 du 找不到大文件
   → 被删文件仍被进程打开（lsof | grep deleted）
   → 工具链：df -h → du -sh * → lsof

场景 3：OOM Killer 频繁杀进程
   → 哪个进程被杀了？谁在拼命吃内存？
   → 工具链：dmesg | grep oom → ps aux --sort=-%mem → /proc/<pid>/smaps

场景 4：网络连接超时
   → DNS 解析慢了？端口被封了？路由走错了？
   → 工具链：dig → ss → traceroute → tcpdump → iptables -L

场景 5：CPU 100% 但不知道在做什么
   → 哪个进程？哪个函数？
   → 工具链：top/htop → pidstat → perf top → 火焰图

场景 6：容器内应用异常慢，但宿主机资源充足
   → cgroup 限制？文件系统 I/O？网络延迟？
   → 工具链：docker stats → iostat → 检查 cgroup 配置 → strace -T
```

写一份排障笔记，记录每个场景的现象→排查过程→根因→修复方案。

---

## 四、里程碑检查点

```
Week 1 结束：✓ 理解 inode/硬链接/软链接/权限模型，能写健壮的 Shell 脚本
Week 2 结束：✓ 能编写 systemd unit/timer，配置用户与权限，管理存储和包
Week 3 结束：✓ 能理解 TCP/IP 协议栈，用 tcpdump/iptables 调试，理解容器网络原理
Week 4 结束：✓ 能独立排查 CPU/内存/磁盘/网络性能问题，理解 cgroup 资源限制
```

---

## 五、推荐资源

| 类型 | 资源 |
|------|------|
| 在线教程 | [linuxjourney.com](https://linuxjourney.com/) — 交互式 Linux 学习路径 |
| 命令速查 | [cheat.sh](https://cheat.sh/) — 命令行速查表，如 `curl cheat.sh/tar` |
| 动手实验 | [sadservers.com](https://sadservers.com/) — Linux 故障排除挑战（强推！） |
| 书籍 | 《UNIX/Linux 系统管理技术手册》(UNIX and Linux System Administration Handbook) — 系统管理圣经 |
| 书籍 | 《性能之巅》(Systems Performance, Brendan Gregg) — 性能分析的终极参考 |
| 参考 | [brendangregg.com](https://www.brendangregg.com/) — Brendan Gregg 的性能文章和火焰图 |
| 工具 | `tldr` 命令（`npm install -g tldr`）— 比 man 更友好 |
| 社区 | [serverfault.com](https://serverfault.com/) — Server Fault，系统管理的 Stack Overflow |
