# NanoPC T3 Plus 部署 HydroAI 教程

> 在友善之臂 NanoPC T3 Plus（S5P6818, Cortex-A53）上部署 HydroAI OJ 自动回复机器人的完整指南。

---

## 目录

- [1. 设备信息](#1-设备信息)
- [2. 准备工作](#2-准备工作)
- [3. SSH 连接](#3-ssh-连接)
- [4. 系统环境配置](#4-系统环境配置)
- [5. 安装 HydroAI](#5-安装-hydroai)
- [6. 配置](#6-配置)
- [7. 创建系统服务（开机自启）](#7-创建系统服务开机自启)
- [8. 屏幕显示日志（可选）](#8-屏幕显示日志可选)
- [9. Web 管理面板](#9-web-管理面板)
- [10. 管理命令速查](#10-管理命令速查)
- [11. 故障排除](#11-故障排除)

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
- MicroSD 卡（16GB 以上，已刷 Ubuntu 24.04）
- 网线 或 已配置 WiFi
- （可选）HDMI 屏幕 + 键盘鼠标

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

在路由器后台查看开发板的 IP，或在接到屏幕的开发板上输入：

```bash
ip addr show | grep 'inet '
```

常见的 IP 段为 `192.168.x.x`。

### 3.2 首次登录

```bash
ssh root@192.168.x.x
```

默认密码因固件版本而异，常见的有：`fa` / `root` / `pi` / `1234`。

### 3.3 设置时区

```bash
timedatectl set-timezone Asia/Shanghai
date
# 输出示例：2026年 06月 14日 星期一 22:00:00 CST
```

---

## 4. 系统环境配置

### 4.1 更新系统

```bash
apt-get update && apt-get upgrade -y
```

### 4.2 安装 Python 和依赖

Ubuntu 24.04 自带 Python 3.12：

```bash
apt-get install -y python3-pip python3.12-venv python3-requests
```

### 4.3 安装常用工具

```bash
apt-get install -y git screen tmux curl
```

---

## 5. 安装 HydroAI

### 5.1 克隆项目

```bash
cd /opt
git clone https://github.com/jianzongX/HydroAI.git
cd HydroAI
```

### 5.2 创建虚拟环境并安装依赖

```bash
python3 -m venv venv
source venv/bin/activate
pip install requests
```

### 5.3 项目文件结构

```
/opt/HydroAI/
├── main.py              入口文件
├── bot.py               机器人主逻辑（消息轮询、AI 回复）
├── config.py            配置管理
├── ai_client.py         DeepSeek API 客户端（流式）
├── oj_client.py         OJ 站内信 API
├── command_parser.py    自然语言命令解析
├── console_ui.py        控制台 UI
├── web_tools.py         工具函数
├── web_server.py        Web 面板后端
├── index.html           Web 面板前端
├── venv/                Python 虚拟环境
├── settings.json        配置文件（用户填写）
└── bot.log              运行日志
```

---

## 6. 配置

### 6.1 创建配置文件

```bash
nano /opt/HydroAI/settings.json
```

填写你的 OJ 账号和 API Key：

```json
{
    "_comment": "HydroAI 配置文件",
    "oj": {
        "username": "你的 OJ 用户名",
        "password": "你的 OJ 密码",
        "base_url": "https://hydro.ac"
    },
    "ai": {
        "api_key": "sk-你的 DeepSeek API Key",
        "api_url": "https://api.deepseek.com/v1/chat/completions",
        "model": "deepseek-v4-flash",
        "system_prompt": "你是 DeepSeek，回复简洁易懂，不用 Markdown。",
        "max_tokens": 4096,
        "temperature": 0.7,
        "timeout": 120
    },
    "bot": {
        "admin_id": 你的管理员用户ID,
        "allowed_user_ids": [],
        "poll_interval_seconds": 3,
        "blocked_words": []
    }
}
```

### 6.2 获取 DeepSeek API Key

1. 注册 [DeepSeek 平台](https://platform.deepseek.com)
2. 进入 API Keys 页面，创建新 Key
3. 复制以 `sk-` 开头的密钥

### 6.3 获取管理员 ID

在 OJ 站点查看个人信息页面的 URL，`/user/214` 中的数字即为用户 ID。

---

## 7. 创建系统服务（开机自启）

### 7.1 创建服务文件

```bash
cat > /etc/systemd/system/hydroai.service << 'EOF'
[Unit]
Description=HydroAI - OJ AI Auto Reply Bot
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/HydroAI
ExecStart=/usr/bin/tmux new-session -d -s hydroai -c /opt/HydroAI '/opt/HydroAI/venv/bin/python3 /opt/HydroAI/main.py'
ExecStop=/usr/bin/tmux kill-session -t hydroai 2>/dev/null; pkill -f "python3.*main.py" 2>/dev/null; sleep 1
ExecReload=/usr/bin/tmux kill-session -t hydroai 2>/dev/null; sleep 1; /usr/bin/tmux new-session -d -s hydroai -c /opt/HydroAI '/opt/HydroAI/venv/bin/python3 /opt/HydroAI/main.py'
Restart=no

[Install]
WantedBy=multi-user.target
EOF
```

### 7.2 启用并启动

```bash
systemctl daemon-reload
systemctl enable hydroai.service
systemctl start hydroai.service
```

### 7.3 验证

```bash
systemctl status hydroai.service --no-pager -l
```

输出应包含 `enabled` 和 `active (exited)`。

### 7.4 查看运行日志

```bash
journalctl -u hydroai -f
```

---

## 8. 屏幕显示日志（可选）

如果 NanoPC 连接了 HDMI 屏幕，可以配置自动显示 Bot 运行日志。

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

# HydroAI: auto-attach tmux on tty1
if [ "$(tty)" = "/dev/tty1" ]; then
    count=0
    while [ $count -lt 30 ]; do
        if tmux has-session -t hydroai 2>/dev/null; then
            exec fbterm -- tmux attach-session -t hydroai
        fi
        sleep 1
        count=$((count + 1))
    done
fi
PROFILE
```

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

### 8.6 屏幕操作

| 操作 | 按键/命令 |
|---|---|
| 退出 tmux（回到 shell） | `Ctrl+B` 松手再按 `D` |
| 回去看日志 | `tmux attach` |
| 切换其他 TTY | `Ctrl+Alt+F2` |
| 回到日志 TTY | `Ctrl+Alt+F1` |
| 调整字体大小 | 编辑 `/root/.fbtermrc`，改 `font-size` 值 |

---

## 9. Web 管理面板

启动后，浏览器打开：

```
http://开发板IP:8080
```

功能：
- 查看运行状态（启动/停止、处理消息数）
- 启动 / 停止 Bot
- 管理白名单（添加、删除、备注）
- 管理屏蔽词
- 修改 AI 配置（API Key、模型、提示词）
- 修改 OJ 配置（账号、密码）
- 实时日志

---

## 10. 管理命令速查

### 服务管理

```bash
systemctl status hydroai      # 查看状态
systemctl restart hydroai     # 重启
systemctl stop hydroai        # 停止
journalctl -u hydroai -f      # 实时日志
```

### 查看 Bot 输出

```bash
tmux attach -t hydroai        # SSH 远程查看运行界面
tail -f /var/log/hydroai/bot.log   # 查看日志文件
```

### 更新代码

```bash
cd /opt/HydroAI
git pull
source venv/bin/activate
pip install requests            # 更新依赖
systemctl restart hydroai
```

### 修改配置（热加载）

编辑 `settings.json` 后 **无需重启**，Bot 会在 30 秒内自动加载新配置。

### 查看系统状态

```bash
htop           # CPU/内存
df -h          # 磁盘
free -h        # 内存
cat /sys/class/thermal/thermal_zone0/temp  # CPU 温度（需除以1000）
```

---

## 11. 故障排除

### Bot 启动失败

```bash
# 查看详细错误
journalctl -u hydroai -n 50 --no-pager

# 手动调试运行
cd /opt/HydroAI && source venv/bin/activate
python main.py
```

### 屏幕中文乱码

```bash
# 检查 locale
locale | grep LANG

# 手动设置
export LANG=zh_CN.UTF-8

# 重新配置 locale
dpkg-reconfigure locales
```

### 网络连不上 GitHub

```bash
# 使用国内镜像
git config --global url."https://ghproxy.com/".insteadOf "https://github.com/"

# 或切换 SSH
git remote set-url origin git@github.com:jianzongX/HydroAI.git
```

### 供电不稳定

NanoPC T3 Plus 需要 **稳定 5V/2A** 供电。出现以下情况请检查电源：
- 随机重启
- USB 设备掉线
- SD 卡读写错误

建议使用原装电源适配器，不要通过 USB HUB 供电。

---

## License

MIT
