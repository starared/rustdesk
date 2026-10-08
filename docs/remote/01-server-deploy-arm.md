# ARM 服务器部署 rustdesk-server 手册（Docker Compose）

> 核对时间：2026-10-08。依据 rustdesk/rustdesk-server 仓库 master 分支的 `README.md`、`docker-compose.yml`、`docs/environment-variables.md`，以及 Docker Hub `rustdesk/rustdesk-server` 的标签列表。

## 0. 结论先行

| 项目 | 取值 |
|---|---|
| 镜像 | `rustdesk/rustdesk-server:1.1.16`（Docker Hub 当前最新版本标签，2026-07-20 发布，多架构清单含 amd64 / arm64 / arm） |
| arm64 专用标签 | `rustdesk/rustdesk-server:1.1.16-arm64v8`（一般不需要，多架构标签会自动选 arm64） |
| 进程 | `hbbs`（ID/注册/打洞服务器）+ `hbbr`（中继服务器） |
| 必开端口 | TCP 21115、21116、21117；UDP 21116 |
| 可选端口 | TCP 21118、21119（仅网页版客户端需要，我们用不到，可不开） |
| 客户端需要填的 | ID 服务器 = 你的域名或公网 IP；Key = `data/id_ed25519.pub` 文件内容；中继服务器可留空 |

建议固定版本号 `1.1.16`，不要用 `latest`，升级时手动改版本号，避免服务器某天自动换版本导致客户端连不上。

注意：官方 README 里提到的 `BIND` 参数要求 1.1.17+，而 Docker Hub 上目前最新是 1.1.16，所以下面不使用 `BIND`。

## 1. 端口说明（来自官方 Port reference）

| 端口 | 协议 | 进程 | 用途 |
|---|---|---|---|
| 21115 | TCP | hbbs | NAT 类型测试 |
| 21116 | TCP + UDP | hbbs | ID 注册、会合、UDP 打洞 |
| 21117 | TCP | hbbr | 中继 |
| 21118 | TCP | hbbs | WebSocket 会合（仅网页客户端） |
| 21119 | TCP | hbbr | WebSocket 中继（仅网页客户端） |

## 2. 准备工作

```bash
# 确认是 arm64
uname -m          # 应输出 aarch64

# 安装 Docker（官方脚本支持 arm64 的 Ubuntu/Debian/CentOS 等）
curl -fsSL https://get.docker.com | sudo sh
sudo systemctl enable --now docker
docker compose version   # 确认 compose 插件可用
```

如果有域名，先把一条 A 记录（例如 `rd.example.com`）指向服务器公网 IP。没有域名也可以直接用公网 IP，后文 `rd.example.com` 全部替换成你的 IP 即可。

## 3. docker-compose.yml

```bash
sudo mkdir -p /opt/rustdesk && cd /opt/rustdesk
sudo nano docker-compose.yml
```

写入（把 `rd.example.com` 换成你的域名或公网 IP）：

```yaml
services:
  hbbs:
    container_name: hbbs
    image: rustdesk/rustdesk-server:1.1.16
    command: hbbs -r rd.example.com:21117
    volumes:
      - ./data:/root
    network_mode: host
    depends_on:
      - hbbr
    restart: unless-stopped

  hbbr:
    container_name: hbbr
    image: rustdesk/rustdesk-server:1.1.16
    command: hbbr -k _
    volumes:
      - ./data:/root
    network_mode: host
    restart: unless-stopped
```

几点说明：

- **`network_mode: host`**：官方示例用的是端口映射 + bridge 网络，这里改成 host 网络。原因是 hbbs 需要看到客户端的真实公网 IP 和端口来做 UDP 打洞，bridge 网络下 Docker 的 NAT 可能改写来源地址，导致直连成功率下降、更多流量走中继。用 host 网络后不需要写 `ports:`。
- **`./data:/root`**：官方镜像的工作目录是 `/root`，密钥对 `id_ed25519` / `id_ed25519.pub` 和数据库 `db_v2.sqlite3` 都会生成在这里，两个容器共用一个目录，所以共用同一套密钥。
- **`hbbs` 不加 `-k`**：官方文档写明 hbbs 默认就是 `-k -`，即首次启动自动生成密钥并强制校验。
- **`hbbr -k _`**：官方文档写明 hbbr 默认 Key 为空，即**不校验**，任何知道你 IP 的人都能白嫖你的中继带宽。加 `-k _` 让 hbbr 读取同目录的密钥并开启校验。出于隐私和防滥用考虑，建议保留。
- **`-r rd.example.com:21117`**：告诉客户端中继地址。官方说明中继和 hbbs 同一台机、端口是 21117 时可以省略，这里显式写上更稳。

## 4. 启动与获取公钥

```bash
cd /opt/rustdesk
sudo docker compose up -d
sudo docker compose ps          # 两个容器都应是 running
sudo docker logs hbbs | tail -20
sudo docker logs hbbr | tail -20

# 公钥（客户端里填的 Key）
sudo cat /opt/rustdesk/data/id_ed25519.pub
```

`id_ed25519.pub` 里是一行 44 字符左右的 base64 字符串，把它完整记下来。

**私钥 `id_ed25519` 绝对不要外传**，也不要提交到 GitHub。后面我们把服务器地址和公钥写进自编译客户端时，只用公钥。

## 5. 防火墙

需要放行两层：云厂商安全组 + 系统防火墙。

### 5.1 云厂商安全组 / 防火墙规则

在云控制台入站规则中放行：

- TCP 21115-21117
- UDP 21116

### 5.2 系统防火墙

Ubuntu / Debian（ufw）：

```bash
sudo ufw allow 21115:21117/tcp
sudo ufw allow 21116/udp
sudo ufw reload
```

CentOS / Rocky / Oracle Linux（firewalld）：

```bash
sudo firewall-cmd --permanent --add-port=21115-21117/tcp
sudo firewall-cmd --permanent --add-port=21116/udp
sudo firewall-cmd --reload
```

**甲骨文云（Oracle Cloud）ARM 实例特别注意**：官方 Ubuntu 镜像自带 iptables 规则，在 INPUT 链末尾有一条 REJECT，即使安全组放行了也会被拦。处理方式：

```bash
sudo iptables -I INPUT 6 -p tcp -m multiport --dports 21115:21117 -j ACCEPT
sudo iptables -I INPUT 6 -p udp --dport 21116 -j ACCEPT
sudo netfilter-persistent save
```

（`6` 是插入位置，需要在那条 REJECT 之前，可以先 `sudo iptables -L INPUT --line-numbers` 看一下。）

## 6. 开机自启

- `restart: unless-stopped` + `systemctl enable docker` 已经能保证服务器重启后自动拉起，一般不需要额外配置。
- 如果你希望用 systemd 统一管理（例如想 `systemctl status rustdesk`），可以加一个单元文件：

```ini
# /etc/systemd/system/rustdesk-server.service
[Unit]
Description=RustDesk server (hbbs + hbbr)
Requires=docker.service
After=docker.service network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/rustdesk
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now rustdesk-server
```

## 7. 验证

### 7.1 服务器本机

```bash
sudo ss -tulnp | grep -E '2111[5-7]'
# 应看到 hbbs 监听 21115/tcp、21116/tcp、21116/udp，hbbr 监听 21117/tcp
```

### 7.2 从外网（自己电脑）

Windows PowerShell：

```powershell
Test-NetConnection rd.example.com -Port 21116
Test-NetConnection rd.example.com -Port 21117
```

Linux / macOS：

```bash
nc -vz rd.example.com 21115
nc -vz rd.example.com 21116
nc -vz rd.example.com 21117
nc -vzu rd.example.com 21116   # UDP 用 nc 只能大致判断
```

### 7.3 用官方客户端端到端验证（在 Fork 编译完成之前就可以先做）

1. Windows 和 Android 都装官方 RustDesk 客户端。
2. 设置 → 网络 → ID/中继服务器：
   - ID 服务器：`rd.example.com`
   - 中继服务器：留空（或填 `rd.example.com`）
   - API 服务器：留空
   - Key：粘贴 `id_ed25519.pub` 的内容
3. 回到主界面，底部状态应显示“就绪”。
4. 用手机连电脑、电脑连手机各试一次。
5. 查看是直连还是中继：`sudo docker logs hbbr` 里出现新连接记录说明走了中继；如果没有记录且能连上，就是打洞直连成功。

这一步跑通，说明服务器侧没有问题，后面自编译客户端如果连不上，就只需要查客户端。

## 8. 常见问题

| 现象 | 原因与处理 |
|---|---|
| 客户端一直显示“正在连接网络” | 21116 TCP/UDP 不通。检查安全组、系统防火墙、Oracle iptables；用 7.2 的命令逐个测。 |
| 能显示就绪但连接时提示“Key 不匹配” | 客户端 Key 填错或带了空格/换行；或者删过 `data/` 目录导致重新生成了密钥。 |
| 连接成功但很卡 | 走了中继，速度受服务器带宽限制。可在 hbbr 环境变量调 `SINGLE_BANDWIDTH`（默认 128 Mb/s）/ `TOTAL_BANDWIDTH`（默认 1024 Mb/s），但真正瓶颈通常是服务器出口带宽。 |
| 想强制全部走中继（例如对方网络不允许直连） | 给 hbbs 加环境变量 `ALWAYS_USE_RELAY=Y`。 |
| 启动日志里 UDP 自检报错 | 某些 NAT/代理环境下自检会误报，可给 hbbs 设 `TEST_HBBS=no` 跳过。 |
| 需要调试日志 | 给容器加环境变量 `RUST_LOG=debug`（必须是进程环境变量，写在 `.env` 里无效）。 |
| 服务器迁移 | 把整个 `/opt/rustdesk/data` 目录（含 `id_ed25519`、`id_ed25519.pub`、`db_v2.sqlite3`）拷到新机器即可，客户端无需改 Key。 |
| 想换 latest 镜像 | 不建议。升级流程：改 compose 里的版本号 → `docker compose pull` → `docker compose up -d`，先备份 `data/`。 |

## 9. 备份

```bash
sudo tar czf ~/rustdesk-data-$(date +%F).tgz -C /opt/rustdesk data
```

密钥一旦丢失，所有客户端都要重新填 Key（自编译客户端则要重新编译），所以这个备份很重要。
