使用 Docker/Podman 在服务器与本地 Linux 上部署 FRP，实现 SSH 内网穿透。

适用版本：frp 0.60+（TOML 配置）。

目标：在公网服务器运行 frps，在本地 Linux 运行 frpc，将本地 SSH（22）映射到服务器端口 6000。

```
ssh -p 6000 <username>@<公网服务器IP>
```

---

# 1. 架构拓扑

```
[公网服务器]
    ↑ (TCP 7000)
    │
    │       --- SSH --> 本地 22
    │
[frps] <----------------------- [frpc]
    ↓
Remote Port: 6000
```

---

# 2. 公网服务器：部署 frps（Docker）

## 2.1 配置文件

路径：/opt/frp/frps.toml

```
bindPort = 7000

[webServer]
addr = "0.0.0.0"
port = 7500
user = "admin"
password = "admin"

[auth]
token = "my_secret_token_123"
```

## 2.2 运行 frps（Docker）

```
docker rm -f frps 2>/dev/null || true

docker run -d --name frps \
  -v /opt/frp/frps.toml:/etc/frp/frps.toml \
  -p 7000:7000 \
  -p 7500:7500 \
  -p 6000:6000 \
  --restart always \
  snowdreamtech/frps
```

## 2.3 检查端口监听

```
sudo ss -tlnp | grep :6000
```

期望输出包含 `LISTEN ... 0.0.0.0:6000 ...`。

## 2.4 云安全组

放行 7000/tcp、7500/tcp、6000/tcp。

---

# 3. 本地 Linux：部署 frpc（Podman）

## 3.1 配置文件

路径示例：/home/<user>/frpc/frpc.toml

```
serverAddr = "你的服务器IP"
serverPort = 7000

[auth]
token = "my_secret_token_123"

[[proxies]]
name = "ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6000
```

## 3.2 文件权限

```
chmod 644 frpc.toml
```

## 3.3 运行 frpc（Podman）

使用 host 网络并添加 SELinux 标签：

```
podman rm -f frpc 2>/dev/null || true

podman run -d --name frpc \
  -v /home/<user>/frpc/frpc.toml:/etc/frp/frpc.toml:Z \
  --network host \
  --restart always \
  docker.io/snowdreamtech/frpc
```

若缺少 `:Z`，SELinux 会导致 `open /etc/frp/frpc.toml : permission denied`。

---

# 4. 验证

## 4.1 Dashboard

访问 `http://<服务器IP>:7500`，确认：

- ClientCounts = 1
- Proxies 中存在 ssh，状态 online
- Port = 6000

## 4.2 SSH 测试

```
ssh -p 6000 hanlin@<服务器IP>
```

---

# 5. 常见问题

## 5.1 SSH 提示 Connection closed/refused

- Docker 未映射 6000，重新运行时加上 `-p 6000:6000`
- 宿主机未监听 6000，使用 `sudo ss -tlnp | grep :6000` 检查

## 5.2 frpc 日志 permission denied

- 配置权限不足：`chmod 644 frpc.toml`
- Podman 挂载未加 `:Z`，重新运行容器

## 5.3 Dashboard 无法访问或 frps 启动失败

查看日志：

```
docker logs frps
```

若出现 `json: unknown field "dashboardPort"`，说明使用了旧配置，需改用 `[webServer]`。

---

# 6. 关键要点

- 使用 frp 0.60+ TOML 配置
- 服务器端显式映射 remotePort 6000
- frpc 使用 host 网络并添加 SELinux 标签 `:Z`
- 本地 SSH 服务正常监听 22
- 云安全组放行 7000/7500/6000
