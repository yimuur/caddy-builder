# Caddy 常用主流插件模块参考清单

## 📋 总览表

| 插件名称 | 模块路径 | 用途分类 | 仓库链接 |
| :--- | :--- | :--- | :--- |
| **Cloudflare DNS** | `github.com/caddy-dns/cloudflare` | DNS 与证书签发 | <https://github.com/caddy-dns/cloudflare> |
| **阿里云 DNS** | `github.com/caddy-dns/alidns` | DNS 与证书签发 | <https://github.com/caddy-dns/alidns> |
| **腾讯云 DNS** | `github.com/caddy-dns/tencentcloud` | DNS 与证书签发 | <https://github.com/caddy-dns/tencentcloud> |
| **caddy-security** | `github.com/greenpau/caddy-security` | 安全与访问控制 | <https://github.com/greenpau/caddy-security> |
| **caddy-defender** | `github.com/JasonLovesDoggo/caddy-defender` | 安全与访问控制 | <https://github.com/JasonLovesDoggo/caddy-defender> |
| **Coraza WAF** | `github.com/corazawaf/coraza-caddy/v2` | 安全与访问控制 | <https://github.com/corazawaf/coraza-caddy> |
| **caddy-ratelimit** | `github.com/mholt/caddy-ratelimit` | 流量控制与性能 | <https://github.com/mholt/caddy-ratelimit> |
| **cache-handler** | `github.com/caddyserver/cache-handler` | 流量控制与性能 | <https://github.com/caddyserver/cache-handler> |
| **WebDAV** | `github.com/mholt/caddy-webdav` | 文件与网络服务 | <https://github.com/mholt/caddy-webdav> |
| **caddy-l4** | `github.com/mholt/caddy-l4` | 文件与网络服务 | <https://github.com/mholt/caddy-l4> |
| **Dynamic DNS** | `github.com/mholt/caddy-dynamicdns` | 文件与网络服务 | <https://github.com/mholt/caddy-dynamicdns> |

> 表中模块路径以实际编译时 `xcaddy build --with` 使用的为准。个别插件（如 Coraza WAF）的模块路径带有 `/v2` 后缀，编译时需完整填写。

## 📦 DNS 与证书签发

这类插件的核心作用是启用 **DNS-01 质询**，从而签发泛域名证书（`*.example.com`）或内网域名证书，**无需在源站开放 80/443 端口**。

### Cloudflare DNS

**模块路径**：`github.com/caddy-dns/cloudflare`
**Caddy 模块名**：`dns.providers.cloudflare`
**仓库链接**：<https://github.com/caddy-dns/cloudflare>

**用途**：通过 Cloudflare API 完成 DNS-01 质询。需要创建具有 `Zone.Zone:Read` 和 `Zone.DNS:Edit` 权限的 API Token（推荐使用单一 Token，同时包含两项权限）。

**Caddyfile 用法**：

```caddyfile
example.com, *.example.com {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
    }
    reverse_proxy localhost:32400
}
```

如果环境中 DNS 传播较慢，可以追加 resolver 指定公共 DNS 加速验证：

```caddyfile
tls {
    dns cloudflare {env.CF_API_TOKEN}
    resolvers 1.1.1.1
}
```

### 阿里云 DNS

**模块路径**：`github.com/caddy-dns/alidns`
**仓库链接**：<https://github.com/caddy-dns/alidns>

**用途**：通过阿里云 DNS（AliDNS）API 完成 DNS-01 质询，自动签发和续期证书。如果使用 STS 临时授权，还可以追加 `security_token` 参数。

**Caddyfile 用法**：

```caddyfile
tls {
    dns alidns {
        access_key_id {env.ALIYUN_ACCESS_KEY_ID}
        access_key_secret {env.ALIYUN_ACCESS_KEY_SECRET}
    }
}
```

### 腾讯云 DNS

**模块路径**：`github.com/caddy-dns/tencentcloud`
**仓库链接**：<https://github.com/caddy-dns/tencentcloud>

**用途**：通过腾讯云 DNSPod API 完成 DNS-01 质询。

**Caddyfile 用法**：

```caddyfile
tls {
    dns tencentcloud {
        secret_id {env.TENCENTCLOUD_SECRET_ID}
        secret_key {env.TENCENTCLOUD_SECRET_KEY}
    }
}
```

## 🛡️ 安全与访问控制

### caddy-security

**模块路径**：`github.com/greenpau/caddy-security`
**仓库链接**：<https://github.com/greenpau/caddy-security>

**用途**：为 Caddy 路由添加**身份认证与授权**，支持 Form-Based、Basic、Local、LDAP、OpenID Connect、OAuth 2.0（GitHub、Google、Facebook、Okta 等）、SAML 认证，以及基于 JWT/PASETO 的授权。它提供两个核心 HTTP 指令：`authenticate`（登录门户）和 `authorize`（保护资源路由）。

**Caddyfile 用法**（最小骨架）：

```caddyfile
example.com {
    @portal path /auth /auth/*
    route @portal {
        authenticate with myportal
    }
    route /app* {
        authorize with app_policy
        reverse_proxy 127.0.0.1:8080
    }
}
```

其中 `myportal` 和 `app_policy` 需要在顶层的 `security { ... }` 应用块中提前定义。门户路由必须放在受保护路由**之前**。

### caddy-defender

**模块路径**：`github.com/JasonLovesDoggo/caddy-defender`
**仓库链接**：<https://github.com/JasonLovesDoggo/caddy-defender>

**用途**：基于客户端 IP 识别并**拦截或操纵**来自 AI 爬虫、云服务商（AWS、GCP、Azure、OpenAI、DeepSeek、GitHub Copilot 等）的请求。支持多种响应策略：`block`（403）、`custom`（自定义消息）、`drop`（断开连接）、`garbage`（返回垃圾数据干扰 AI 训练）、`redirect`（308 重定向）、`tarpit`（拖延连接）等。

**Caddyfile 用法**：

```caddyfile
example.com {
    defender block {
        ranges openai aws gcp
        whitelist 203.0.113.50
    }
    reverse_proxy localhost:3000
}
```

### Coraza WAF

**模块路径**：`github.com/corazawaf/coraza-caddy/v2`
**仓库链接**：<https://github.com/corazawaf/coraza-caddy>

**用途**：集成 **OWASP Coraza WAF**，100% 兼容 OWASP Core Rule Set (CRS) 和 ModSecurity 语法，提供 SQL 注入、XSS 等 Web 攻击防护。

**Caddyfile 用法**：Coraza 对指令顺序有硬性要求，**必须在全局选项中添加 `order coraza_waf first`**，否则模块不会生效。

```caddyfile
{
    order coraza_waf first
}

example.com {
    coraza_waf {
        load_owasp_crs
        directives `
            Include @coraza.conf-recommended
            Include @crs-setup.conf.example
            Include @owasp_crs/*.conf
        `
    }
    reverse_proxy localhost:8080
}
```

`load_owasp_crs` 启用的 `@` 路径规则**已编译在二进制内**，无需额外下载或挂载规则文件。

## ⚡ 流量控制与性能

### caddy-ratelimit

**模块路径**：`github.com/mholt/caddy-ratelimit`
**仓库链接**：<https://github.com/mholt/caddy-ratelimit>

**用途**：基于**滑动窗口算法**对 HTTP 请求进行限流。以“key + 速率”形式定义 zone，超限返回 `429 Too Many Requests`，并自动设置 `Retry-After` 响应头。支持分布式限流（集群内多实例共享状态）。

**Caddyfile 用法**：

```caddyfile
example.com {
    rate_limit {
        zone dynamic {
            key {remote_host}
            events 100
            window 1m
        }
    }
    reverse_proxy localhost:8080
}
```

该配置表示：对每个远程 IP，每分钟最多 100 次请求。`key` 可使用 `{remote_host}`、`{query.id}` 等占位符实现按 IP、按查询参数等维度的精细化限流。

### cache-handler

**模块路径**：`github.com/caddyserver/cache-handler`
**仓库链接**：<https://github.com/caddyserver/cache-handler>

**用途**：基于 Souin 的**分布式 HTTP 缓存**模块，符合 RFC 7234，设置 `Cache-Status` 响应头，提供 REST API 用于清除缓存和列出资源，并支持 ESI 标签处理。**存储后端需要单独引入**（如 Otter、Redis、Badger 等）。

**Caddyfile 用法**（使用 Otter 内存存储）：

```caddyfile
{
    order cache before rewrite
    cache {
        otter {
            configuration {
                size 1000
            }
        }
    }
}

example.com {
    cache
    reverse_proxy localhost:8080
}
```

编译时需要同时引入 storage 模块：`xcaddy build --with github.com/caddyserver/cache-handler --with github.com/darkweak/storages/otter/caddy`。

## 🗂️ 文件与网络服务

### WebDAV

**模块路径**：`github.com/mholt/caddy-webdav`
**仓库链接**：<https://github.com/mholt/caddy-webdav>

**用途**：为指定目录提供 **WebDAV 文件共享**服务，支持标准 WebDAV 客户端挂载和读写文件。

**Caddyfile 用法**：WebDAV 不是标准指令，必须手动调整顺序，**放在 `file_server` 之前**：

```caddyfile
{
    order webdav before file_server
}

media.example.com {
    root * /var/media
    webdav
}
```

注意：执行 WebDAV **PUT** 操作时，Caddy 进程需要对目标存储目录有写权限。

### Layer 4 (caddy-l4)

**模块路径**：`github.com/mholt/caddy-l4`
**仓库链接**：<https://github.com/mholt/caddy-l4>

**用途**：为 Caddy 增加**四层（TCP/UDP）代理**能力（项目名 “Project Conncept”），基于连接属性或流开头进行可组合的原始 TCP/UDP 处理。典型场景包括 TLS 终止后转发原始 TCP、按 TLS SNI 或 HTTP Host 分流、SSH 代理、连接限速等。该模块仍处于实验阶段，可能有破坏性变更。

**Caddyfile 用法**（在 `layer4` 顶级块中定义）：

```caddyfile
{
    layer4 {
        :993 {
            @tls tls
            route @tls {
                proxy {
                    to localhost:9993
                }
            }
        }
    }
}
```

### Dynamic DNS

**模块路径**：`github.com/mholt/caddy-dynamicdns`
**仓库链接**：<https://github.com/mholt/caddy-dynamicdns>

**用途**：自动将本机的**公网 IPv4/IPv6 地址**更新到 DNS 服务商的 A/AAAA 记录中，适用于动态公网 IP 环境。支持多种 IP 来源（UPnP、HTTP 查询、网卡接口等）和冗余 IP 源。需要配合 `caddy-dns` 系列的 DNS provider 插件使用。

**Caddyfile 用法**（在**全局选项**中配置）：

```caddyfile
{
    dynamic_dns {
        provider cloudflare {env.CLOUDFLARE_API_TOKEN}
        domains {
            example.com @
            plex.example.com
        }
    }
}
```

`@` 代表 zone 本身（即根域名 `example.com`）。

## 💡 通用注意事项

**① 插件通过 `xcaddy` 编译进二进制**。以上所有插件都不是 Caddy 标准模块，必须通过 `xcaddy build --with <模块路径>` 在构建时嵌入，或使用预集成了插件的二进制/镜像。运行时执行 `caddy list-modules | grep <模块名>` 可确认插件是否已正确加载。

**② 非标准指令需要调整 `order`**。Caddy 对标准指令有固定排序，插件新增的指令（如 `webdav`、`coraza_waf`、`cache`）如果不放入合适的 `order` 位置，可能不生效或产生意外行为。配置前务必查看对应插件的 README 确认是否需要 `order`。

**③ 证书与 DNS 插件的环境变量**。DNS 类插件通常从环境变量读取 API 凭据（如 `{env.CF_API_TOKEN}`），生产环境中应通过 Docker `environment` 或 systemd `EnvironmentFile` 注入，避免将明文 Token 写入 Caddyfile。

**④ 注意维护状态差异**。部分插件（如 `caddy-l4`）明确标注为实验性/开发中，可能有破坏性变更；`caddy-webdav` 也声明非 Caddy 官方组织仓库。在生产环境使用前，建议评估对应插件的活跃度和版本稳定性。
