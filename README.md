# NanoPC T3 Plus Python 运行指南

> 在友善之臂 NanoPC T3 Plus（S5P6818, Cortex-A53）上配置 Python 环境并运行程序的完整教程。

---

## 目录

- [1. 设备信息](#1-设备信息)
- [2. 准备工作](#2-准备工作)
- [3. SSH 连接](#3-ssh-连接)
- [4. 系统环境配置](#4-系统环境配置)
- [5. Python 环境搭建](#5-python-环境搭建)
- [6. 运行 Python 程序](#6-运行-python-程序)
- [7. 创建开机自启服务](#7-创建开机自启服务)
- [8. 屏幕显示程序输出（可选）](#8-屏幕显示程序输出可选)
- [9. 常用命令](#9-常用命令)
- [10. 故障排除](#10-故障排除)

---

## 1. 设备信息

| 项目 | 规格 |
|---|---|
| **型号** | NanoPC T3 Plus |
| **SoC** | Samsung S5P6818 (Cortex-A53 octa-core) |
| **架构** | aarch64 (ARMv8) |
| **内存** | 2GB DDR3 |
| **系统** | Ubuntu 24.04 LTS / FriendlyCore |
| **内核** | 4.4.172-s5p6818 |
| **存储** | MicroSD 卡 / eMMC |
| **供电** | 5V/2A USB-Type C |

---

## 2. 准备工作

### 硬件清单

- NanoPC T3 Plus 主板
- 5V/2A 电源适配器
- MicroSD 卡（16GB 以上）
- 网线 或 已配置 WiFi
- （可选）HDMI 屏幕 + 键鼠

### 刷写系统（首次使用）

从 [友善之臂官方 Wiki](http://wiki.friendlyelec.com) 下载 Ubuntu 24.04 固件：

```bash
# 解压后写入 SD 卡（注意替换 sdX 为实际设备）
sudo dd if=ubuntu-24.04-desktop-arm64.img of=/dev/sdX bs=4M status=progress
sync
```

插入 SD 卡，连接网线，上电启动。

---

## 3. SSH 连接

### 3.1 查找 IP 地址

在路由器后台查看开发板的 IP，或在接屏幕的开发板上输入：

```bash
ip addr show | grep 'inet '
```

### 3.2 首次登录

```bash
ssh root@192.168.x.x
```

默认密码因固件版本而异，常见的有：`fa` / `root` / `pi` / `1234`。

### 3.3 设置时区

```bash
timedatectl set-timezone Asia/Shanghai
date
```

---

## 4. 系统环境配置

### 4.1 更新系统

```bash
apt-get update && apt-get upgrade -y
```

### 4.2 安装基础工具

```bash
apt-get install -y git curl vim screen tmux
```

---

## 5. Python 环境搭建

### 5.1 检查 Python 版本

Ubuntu 24.04 自带 Python 3.12：

```bash
python3 --version
# Python 3.12.3
```

### 5.2 安装 pip 和虚拟环境

```bash
apt-get install -y python3-pip python3.12-venv python3-requests
```

### 5.3 创建虚拟环境（推荐）

为你的项目创建独立的虚拟环境，避免包冲突：

```bash
# 进入项目目录
cd /opt
mkdir myproject && cd myproject

# 创建虚拟环境
python3 -m venv venv

# 激活虚拟环境
source venv/bin/activate

# 现在可以安装依赖了
pip install requests flask numpy
```

> 虚拟环境激活后，命令行前缀会变成 `(venv)`。

### 5.4 退出虚拟环境

```bash
deactivate
```

### 5.5 使用系统级安装（不推荐）

如果不想用虚拟环境：

```bash
# 注意：Ubuntu 24.04 限制了系统级 pip 安装
# 需要使用 --break-system-packages 参数
pip install requests --break-system-packages

# 或直接用 apt 安装 Python 包
apt-get install -y python3-requests python3-flask
```

---

## 6. 运行 Python 程序

### 6.1 前台运行

```bash
# 激活虚拟环境后直接运行
cd /opt/myproject
source venv/bin/activate
python3 main.py
```

特点：输出实时显示，但关掉 SSH 窗口程序就停了。适合调试。

### 6.2 后台运行（nohup）

```bash
# 不依赖 SSH 会话，关掉终端也不停止
nohup python3 main.py > output.log 2>&1 &

# 查看输出
tail -f output.log

# 停止程序
pkill -f "python3 main.py"
```

### 6.3 后台运行（tmux / screen）

推荐方式，可以随时查看和恢复运行界面：

```bash
# 创建 tmux 会话
tmux new-session -d -s mysession 'python3 main.py'

# 查看运行输出
tmux attach -t mysession

# 在 tmux 中：按 Ctrl+B 松手再按 D 可安全退出（程序继续运行）

# 其他 tmux 操作
tmux ls                          # 列出所有会话
tmux kill-session -t mysession   # 停止会话
```

---

## 7. 创建开机自启服务

### 7.1 创建 systemd 服务文件

```bash
cat > /etc/systemd/system/myapp.service << 'EOF'
[Unit]
Description=My Python App
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/myproject
ExecStart=/opt/myproject/venv/bin/python3 /opt/myproject/main.py
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
EOF
```

参数说明：

| 参数 | 说明 |
|---|---|
| `WorkingDirectory` | 程序工作目录 |
| `ExecStart` | 使用虚拟环境中的 Python 运行 |
| `Restart=always` | 崩溃后自动重启 |
| `RestartSec=5` | 重启等待 5 秒 |

### 7.2 启用并启动

```bash
systemctl daemon-reload
systemctl enable myapp.service
systemctl start myapp.service
```

### 7.3 管理服务

```bash
systemctl status myapp           # 查看状态
systemctl restart myapp          # 重启
systemctl stop myapp             # 停止
journalctl -u myapp -f           # 查看实时日志
```

### 7.4 搭配 tmux 的服务

如果你的程序需要保持在 tmux 中运行（方便查看输出）：

```bash
cat > /etc/systemd/system/myapp.service << 'EOF'
[Unit]
Description=My Python App (tmux)
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/myproject
ExecStart=/usr/bin/tmux new-session -d -s myapp -c /opt/myproject '/opt/myproject/venv/bin/python3 /opt/myproject/main.py'
ExecStop=/usr/bin/tmux kill-session -t myapp 2>/dev/null; pkill -f "python3.*main.py" 2>/dev/null
Restart=no

[Install]
WantedBy=multi-user.target
EOF
```

查看输出：`tmux attach -t myapp`

---

## 8. 屏幕显示程序输出（可选）

如果 NanoPC 连接了 HDMI 屏幕，可以让屏幕自动显示程序运行日志。

### 8.1 安装中文支持

```bash
apt-get install -y language-pack-zh-hans fbterm fonts-wqy-zenhei
locale-gen zh_CN.UTF-8
update-locale LANG=zh_CN.UTF-8
```

### 8.2 配置自动登录

```bash
mkdir -p /etc/systemd/system/getty@tty1.service.d
cat > /etc/systemd/system/getty@tty1.service.d/autologin.conf << 'EOF'
[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin root --noclear %I $TERM
EOF
systemctl daemon-reload
```

### 8.3 配置自动进入 tmux

```bash
cat >> /root/.profile << 'PROFILE'

# Auto-attach tmux on tty1
if [ "$(tty)" = "/dev/tty1" ]; then
    count=0
    while [ $count -lt 30 ]; do
        if tmux has-session -t myapp 2>/dev/null; then
            exec fbterm -- tmux attach-session -t myapp
        fi
        sleep 1
        count=$((count + 1))
    done
fi
PROFILE
```

> 将 `myapp` 替换为你的 tmux 会话名称。

### 8.4 配置 fbterm 字体

```bash
cat > /root/.fbtermrc << 'EOF'
font-names=Fira Code,Noto Sans Mono CJK SC,WenQuanYi Zen Hei Mono
font-size=20
color-foreground=192,192,192
color-background=0,0,0
EOF
```

### 8.5 重启 tty1

```bash
systemctl restart getty@tty1
```

### 8.6 屏幕操作指南

| 操作 | 按键/命令 |
|---|---|
| 退出 tmux（回到 shell） | `Ctrl+B` 松手再按 `D` |
| 回去看日志 | `tmux attach` |
| 切换其他 TTY | `Ctrl+Alt+F2` ~ F6 |
| 回到主 TTY | `Ctrl+Alt+F1` |
| 调整字体大小 | 编辑 `/root/.fbtermrc`，修改 `font-size` |

---

## 9. 常用命令

### 服务管理

```bash
systemctl status myapp          # 查看状态
systemctl restart myapp         # 重启
systemctl stop myapp            # 停止
journalctl -u myapp -f          # 实时日志
```

### 进程管理

```bash
ps aux | grep python            # 查看 Python 进程
top -u root                     # 查看资源占用
pkill -f "python3 main.py"      # 杀掉指定程序
```

### 文件传输

```bash
# 从电脑传到开发板
scp local_file root@192.168.x.x:/opt/myproject/

# 从开发板传到电脑
scp root@192.168.x.x:/opt/myproject/output.txt .
```

### 系统监控

```bash
htop                            # 进程监控
df -h                           # 磁盘空间
free -h                         # 内存使用
cat /sys/class/thermal/thermal_zone0/temp  # CPU 温度（除以1000得摄氏度）
```

---

## 10. 故障排除

### Python 报错 "externally-managed-environment"

Ubuntu 24.04 限制了系统级 pip 安装：

```bash
# 方案一：使用虚拟环境（推荐）
python3 -m venv venv
source venv/bin/activate
pip install 你需要的包

# 方案二：用 apt 装（如果有）
apt-get install -y python3-包名

# 方案三：强制安装（不推荐）
pip install 包名 --break-system-packages
```

### 中文显示乱码

```bash
# 检查 locale
locale | grep LANG

# 安装中文语言包
apt-get install -y language-pack-zh-hans
locale-gen zh_CN.UTF-8
update-locale LANG=zh_CN.UTF-8

# 重新登录生效
```

### apt 锁占用

```bash
# 如果 apt 报错无法获取锁
ps aux | grep apt
kill -9 进程PID
rm -f /var/lib/dpkg/lock-frontend
dpkg --configure -a
```

### 供电不稳定

NanoPC T3 Plus 需要 **稳定 5V/2A** 供电。出现以下情况请检查电源：
- 随机重启
- USB 设备掉线
- SD 卡读写错误

### 网络连不上 GitHub

```bash
# 使用国内镜像
git config --global url."https://ghproxy.com/".insteadOf "https://github.com/"

# 或切换 SSH
git remote set-url origin git@github.com:你的用户名/仓库名.git
```

---

## License

MIT
