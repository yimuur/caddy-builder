# syntax=docker/dockerfile:1

# 全局 ARG：用于 FROM 行，无默认值，必须由 CI 传入
ARG CADDY_VERSION

# ---------- 构建阶段：使用官方 builder，用 xcaddy 编译插件 ----------
FROM caddy:${CADDY_VERSION}-builder-alpine AS builder

# 在阶段内重新声明，才能在 RUN 中使用
ARG CADDY_VERSION
ARG CADDY_PLUGINS

# 显式传入版本：v${CADDY_VERSION}，确保与基础镜像版本一致
RUN xcaddy build v${CADDY_VERSION} ${CADDY_PLUGINS}

# ---------- 运行阶段：只保留编译好的二进制和最小运行时 ----------
FROM caddy:${CADDY_VERSION}-alpine

COPY --from=builder /usr/bin/caddy /usr/bin/caddy

# alpine 镜像只有 busybox wget，没有 curl；探测管理 API 判断进程存活
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget -qO- http://localhost:2019/config/ || exit 1

EXPOSE 80 443 443/udp 2019

CMD ["caddy", "run", "--config", "/etc/caddy/Caddyfile", "--adapter", "caddyfile"]
