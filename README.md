# Caddy 自定义镜像自动构建

基于 GitHub Actions 自动构建带插件的 Caddy Docker 镜像（`linux/amd64`），自动追踪 Caddy 最新稳定版，并在推送前完成**冒烟测试**与**Trivy 漏洞扫描**。构建产物推送到 GitHub Container Registry（GHCR）。

## 📁 文件结构

```
.
├── Dockerfile
└── .github/
    └── workflows/
        └── build-caddy.yml
```

- **`Dockerfile`**：多阶段构建。构建阶段用 `xcaddy` 编译插件，运行阶段仅保留定制二进制，并附带基于 `wget` 的健康检查。
- **`build-caddy.yml`**：解析 Caddy 最新 release → 检查镜像是否已存在 → 本地构建 → 冒烟测试 → Trivy 扫描 → 推送 GHCR。

## 🚀 快速开始

### 1. Fork 或创建仓库

将上述两个文件放入你的 GitHub 仓库。私有仓库同样可用。

### 2. 按需修改插件列表

编辑 `.github/workflows/build-caddy.yml` 中的 `Define Caddy plugins` 步骤：

```yaml
PLUGINS=""
PLUGINS="$PLUGINS --with github.com/caddy-dns/cloudflare"
PLUGINS="$PLUGINS --with github.com/mholt/caddy-ratelimit"
PLUGINS="$PLUGINS --with github.com/greenpau/caddy-security"
# 按需增删，每行一个 --with
```

插件模块路径可参考项目内的插件清单文档。删除不需要的行、添加新行即可，无需改动 `Dockerfile`。

### 3. 确认镜像名称

Workflow 中默认镜像名为：

```yaml
IMAGE_NAME: ghcr.io/${{ github.repository_owner }}/caddy-pass
```

如需改为与仓库名一致，可改为 `ghcr.io/${{ github.repository }}`。首次推送后，到仓库 **Packages** 设置中确认包的可见性（公开或私有）。

### 4. 触发构建

- **手动触发**：进入仓库 **Actions** → 选择 `Build Caddy with Latest Stable Version` → **Run workflow**。
- **定时触发**：Workflow 默认每天 06:00 UTC 检查一次 Caddy 新版本。若解析到的版本已存在于 GHCR，则跳过构建；有新版本时才执行完整流水线。

### 5. 拉取并使用镜像

构建成功后：

```bash
docker pull ghcr.io/<你的用户名>/caddy-pass:latest
```

镜像标签包括：

- `latest`
- 具体 Caddy 版本号（如 `2.11.8`）
- 提交 SHA（仅最终推送步骤使用版本号与 `latest`，SHA 标签可按需添加）

运行示例：

```bash
docker run -d \
  --name caddy \
  -p 80:80 -p 443:443 -p 443:443/udp \
  -v $PWD/Caddyfile:/etc/caddy/Caddyfile:ro \
  -v caddy_data:/data \
  -v caddy_config:/config \
  ghcr.io/<你的用户名>/caddy-pass:latest
```

## 🔍 验证插件是否生效

```bash
docker run --rm ghcr.io/<你的用户名>/caddy-pass:latest caddy list-modules | grep -E 'dns.providers|ratelimit|security'
```

应能看到对应模块名，例如 `dns.providers.cloudflare`、`http.handlers.ratelimit`、`security` 等。

## 🛡️ 构建流水线的质量闸门

每次构建都会依次执行：

1. **解析最新稳定版**：通过 `gh release view` 获取 Caddy 最新 release tag，剥离 `v` 前缀作为版本号。
2. **存在性检查**：若该版本镜像已存在于 GHCR，跳过后续步骤，避免重复构建。
3. **本地构建**：使用同一版本号驱动 `FROM` 基础镜像与 `xcaddy build`，杜绝版本漂移。
4. **冒烟测试**：用最小 `Caddyfile` 启动容器，`curl` 验证 HTTP 200 响应；失败则打印容器日志并中止。
5. **Trivy 扫描**：对 `CRITICAL` 级别漏洞阻断推送（`ignore-unfixed: true` 忽略暂无修复的漏洞）。
6. **推送 GHCR**：仅当以上全部通过后，推送 `latest` 与版本号标签。

## 🔄 升级 Caddy 版本

无需手动改版本号。定时任务会自动发现新版本并重建。若希望**固定**某个版本，可在 `Dockerfile` 中为 `ARG CADDY_VERSION` 设置默认值，或在 Workflow 中改为手动指定版本并移除自动解析步骤。

## ⚠️ 注意事项

- **GHCR 权限**：Workflow 使用内置 `GITHUB_TOKEN` 推送镜像，无需额外 PAT。权限已限定为 `contents: read` 与 `packages: write`。
- **健康检查**：镜像内置 `HEALTHCHECK`，探测 `http://localhost:2019/config/`（Caddy 管理 API）。生产环境请保持管理 API 启用，或按需调整健康检查端点。
- **插件顺序**：部分插件（如 `webdav`、`coraza_waf`、`cache`）需要在 Caddyfile 全局选项中调整 `order`，否则不生效。配置前请查阅对应插件文档。
- **构建缓存**：使用 GitHub Actions 缓存（`type=gha,mode=max`），插件列表不变时可显著缩短构建时间。缓存有容量限制，插件极多时如遇缓存失效可改用 registry 缓存后端。
