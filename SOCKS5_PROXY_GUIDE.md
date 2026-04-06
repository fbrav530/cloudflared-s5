# SOCKS5 代理支持 - 完整使用指南

## 📋 功能概述

cloudflared 现已支持通过 SOCKS5 代理连接到 Cloudflare 边缘节点，并且具有**智能降级**功能：
- ✅ 代理可用时使用代理连接
- ✅ 代理失败时自动降级到直连
- ✅ 无需担心代理服务器故障影响服务可用性

## 🚀 快速开始

**⚠️ 重要：参数位置**
- `--edge-proxy-url` 是 `tunnel` 命令级别的参数
- 必须放在 `tunnel` 和 `run` 之间
- 格式：`cloudflared tunnel --edge-proxy-url <URL> run <tunnel-name>`

### 方式 1: 命令行参数

```bash
# 不带认证的代理
cloudflared tunnel --edge-proxy-url socks5://127.0.0.1:1080 run mytunnel

# 带用户名密码认证的代理
cloudflared tunnel --edge-proxy-url socks5://user:pass@proxy.example.com:1080 run mytunnel

# 使用默认端口 1080
cloudflared tunnel --edge-proxy-url socks5://proxy.example.com run mytunnel
```

### 方式 2: 配置文件

在您的 `config.yml` 中添加:

```yaml
tunnel: mytunnel
credentials-file: /path/to/credentials.json

# SOCKS5 代理配置
edge-proxy-url: socks5://127.0.0.1:1080

# 或带认证
# edge-proxy-url: socks5://username:password@proxy.server.com:1080

ingress:
  - hostname: example.com
    service: http://localhost:8080
  - service: http_status:404
```

然后运行:
```bash
cloudflared tunnel run mytunnel
```

### 方式 3: 环境变量

```bash
export TUNNEL_EDGE_PROXY_URL="socks5://127.0.0.1:1080"
cloudflared tunnel run mytunnel
```

## 📖 详细说明

### 代理 URL 格式

完整格式: `socks5://[username:password@]host[:port]`

**示例:**
- `socks5://127.0.0.1:1080` - 本地代理，无认证
- `socks5://proxy.example.com` - 远程代理，使用默认端口 1080
- `socks5://user:pass@192.168.1.100:8080` - 带认证的代理
- `socks5://admin:s3cr3t@proxy.corp.com:1080` - 企业代理

### 工作原理

```
用户启动 cloudflared
        │
        ▼
   配置了代理?
    ┌───┴───┐
   是│       │否
    │       │
    ▼       ▼
尝试代理  直接连接
    │       │
代理成功?   │
 ┌──┴──┐   │
是│    否│   │
 │     │   │
 │  自动降级  │
 │   直连   │
 │     │   │
 └──┬──┘   │
    │     │
    └──┬──┘
       ▼
   连接成功
```

### 自动降级机制

**场景 1: 代理服务器不可达**
```bash
# 即使代理服务器 10.0.0.100:1080 宕机，连接也会自动降级到直连
cloudflared tunnel --edge-proxy-url socks5://10.0.0.100:1080 run mytunnel
# → 代理失败 → 自动直连 → 隧道正常运行 ✓
```

**场景 2: 代理认证失败**
```bash
# 认证信息错误时，自动降级到直连
cloudflared tunnel --edge-proxy-url socks5://wrong:creds@proxy:1080 run mytunnel
# → 认证失败 → 自动直连 → 隧道正常运行 ✓
```

**场景 3: 代理响应超时**
```bash
# 代理服务器响应慢或挂起，自动降级
cloudflared tunnel --edge-proxy-url socks5://slow-proxy:1080 run mytunnel
# → 超时 → 自动直连 → 隧道正常运行 ✓
```

## 🔧 使用场景

### 场景 1: 企业防火墙环境

在有严格出站限制的企业环境中:

```yaml
# config.yml
tunnel: corporate-tunnel
credentials-file: /etc/cloudflared/credentials.json

# 通过企业 SOCKS5 代理出网
edge-proxy-url: socks5://username:password@corporate-proxy.internal:1080

ingress:
  - hostname: internal-app.company.com
    service: http://internal-server:8080
  - service: http_status:404
```

### 场景 2: 国际网络优化

通过代理服务器优化到 Cloudflare 的连接路由:

```bash
# 使用特定地区的代理服务器
cloudflared tunnel --edge-proxy-url socks5://hk-proxy.example.com:1080 run mytunnel
```

### 场景 3: 多层网络架构

在需要通过跳板机连接的环境:

```bash
# 通过跳板机的 SOCKS5 代理连接
cloudflared tunnel --edge-proxy-url socks5://jump-host:1080 run mytunnel
```

### 场景 4: 开发测试环境

测试环境使用代理，生产环境直连:

```yaml
# dev-config.yml
edge-proxy-url: socks5://dev-proxy:1080

# prod-config.yml
# 不配置 edge-proxy-url，使用直连
```

## 🔒 安全最佳实践

### 1. 避免在命令行中暴露密码

❌ **不推荐** (密码会出现在进程列表中):
```bash
cloudflared tunnel --edge-proxy-url socks5://user:password123@proxy:1080 run mytunnel
```

✅ **推荐** (使用配置文件):
```yaml
# config.yml (设置权限 600)
edge-proxy-url: socks5://user:password123@proxy:1080
```

```bash
chmod 600 config.yml
cloudflared tunnel run mytunnel
```

### 2. 使用环境变量

```bash
# 从环境变量读取代理配置
export TUNNEL_EDGE_PROXY_URL="socks5://user:${PROXY_PASSWORD}@proxy:1080"
cloudflared tunnel run mytunnel
```

### 3. 代理服务器安全

- 确保 SOCKS5 代理服务器启用认证
- 使用强密码
- 限制代理服务器的访问 IP 白名单
- 定期轮换代理认证凭据

## 🐛 故障排查

### 问题 1: 无法连接到边缘节点

**检查步骤:**
1. 验证代理服务器是否可达:
```bash
nc -zv proxy-host 1080
```

2. 测试代理功能:
```bash
curl --socks5 user:pass@proxy-host:1080 https://www.cloudflare.com
```

3. 检查 cloudflared 日志:
```bash
cloudflared tunnel --loglevel debug run mytunnel
```

### 问题 2: 代理认证失败

检查认证信息是否正确:
```bash
# 测试代理认证
curl --socks5-user user:pass --socks5 proxy-host:1080 https://www.cloudflare.com
```

### 问题 3: 不确定是否使用了代理

查看连接日志，代理失败时会自动降级:
```bash
cloudflared tunnel --loglevel debug --edge-proxy-url socks5://proxy:1080 run mytunnel
```

## 📊 性能考虑

### 代理的性能影响

使用代理会增加额外的网络跳数:

```
直连模式:
  cloudflared → Cloudflare Edge
  延迟: ~50ms

代理模式:
  cloudflared → SOCKS5 Proxy → Cloudflare Edge
  延迟: ~50ms + 代理延迟
```

**建议:**
- 选择地理位置接近的代理服务器
- 监控代理服务器的性能和稳定性
- 在非必要时不使用代理（依赖自动降级）

## 🧪 测试

### 功能测试

```bash
# 测试基本功能
cd edgediscovery
go test -v -run TestDialEdgeWithProxy

# 测试代理降级
go test -v -run TestDialEdgeWithProxy_FallbackToDirect
```

### 集成测试

创建测试脚本 `test_proxy.sh`:

```bash
#!/bin/bash

# 启动本地 SOCKS5 代理用于测试
# 需要安装: pip install pysocks

# 测试 1: 正常代理
cloudflared tunnel --edge-proxy-url socks5://127.0.0.1:1080 run test-tunnel &
PID=$!
sleep 10
kill $PID

# 测试 2: 无效代理 (应自动降级)
cloudflared tunnel --edge-proxy-url socks5://127.0.0.1:9999 run test-tunnel &
PID=$!
sleep 10
kill $PID

echo "Tests completed"
```

## 📝 技术细节

### 实现文件

- `edgediscovery/dial.go` - 核心拨号逻辑，支持代理和降级
- `supervisor/tunnel.go` - 隧道配置，包含 `EdgeProxyURL` 字段
- `cmd/cloudflared/flags/flags.go` - 命令行标志定义
- `cmd/cloudflared/tunnel/cmd.go` - CLI 参数注册
- `cmd/cloudflared/tunnel/configuration.go` - 配置解析和传递

### 支持的协议

目前 SOCKS5 代理支持对以下协议生效:
- ✅ HTTP/2 连接
- ⚠️  QUIC 连接 (QUIC 使用 UDP，SOCKS5 主要用于 TCP，需要特殊处理)

### 代理 vs LocalIP

**注意:** 使用代理时，`--edge-bind-address` (LocalIP) 参数可能不生效，因为:
- 实际的出口 IP 是代理服务器的 IP
- LocalIP 仅影响到代理服务器的连接，不影响代理到 Cloudflare Edge 的连接

## 🔄 与其他功能的兼容性

| 功能 | 是否兼容 | 说明 |
|------|---------|------|
| High Availability (HA) | ✅ | 每个 HA 连接都会使用代理 |
| Post-Quantum | ✅ | 代理不影响加密协议 |
| Edge IP Version | ✅ | IPv4/IPv6 选择仍然生效 |
| Edge Bind Address | ⚠️ | 可能不生效（见上文） |
| Protocol Fallback | ✅ | 协议降级机制正常工作 |

## 🆘 常见问题

**Q: 代理失败后是否会有性能影响?**
A: 会有短暂的重试延迟（约几秒），之后会正常直连。

**Q: 可以强制只使用代理，不降级吗?**
A: 当前实现会自动降级。如需强制代理，可以通过防火墙规则限制直连。

**Q: 支持其他类型的代理吗（HTTP、HTTPS）?**
A: 目前仅支持 SOCKS5。未来可以扩展支持 HTTP CONNECT 代理。

**Q: 代理服务器需要支持什么?**
A: 标准的 SOCKS5 协议（RFC 1928）即可，可选支持用户名/密码认证（RFC 1929）。

**Q: 参数为什么要放在 tunnel 和 run 之间?**
A: `--edge-proxy-url` 是 tunnel 命令级别的全局参数，适用于所有子命令（run、cleanup 等）。

## 📚 相关资源

- [SOCKS5 协议规范 (RFC 1928)](https://www.rfc-editor.org/rfc/rfc1928)
- [SOCKS5 用户名/密码认证 (RFC 1929)](https://www.rfc-editor.org/rfc/rfc1929)
- [cloudflared 文档](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)

## 📄 更新日志

- 2025-10-27: 添加 SOCKS5 代理支持和自动降级功能
  - 新增 `--edge-proxy-url` 命令行参数
  - 新增 `DialEdgeWithProxy` 函数
  - 新增自动降级到直连的容错机制
- 2026-04-06: 验证功能实现并更新文档
  - 确认参数位置（tunnel 命令级别）
  - 更新所有示例命令格式
  - 补充常见问题解答
