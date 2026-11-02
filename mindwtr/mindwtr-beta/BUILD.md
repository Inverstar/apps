# 构建并推送 mindwtr-merged 镜像

> 这套构建文件只保留在 **Inverstar/apps fork** 里，不随 chart 一起提交到上游 beclab/apps。
> 上游 PR 只包含 `mindwtr/`（Chart + OlaresManifest + values + templates）的改动。

## 前置条件

- Docker Desktop（支持 buildx 多架构构建）
- 已登录 GHCR：`docker login ghcr.io -u inverstar`（密码用 GitHub PAT，需要 `write:packages` 权限）

## 已修复的两个启动阻断（2026-11-02）

| 症状 | 根因 | 修复位置 |
|------|------|----------|
| 容器起来几毫秒就 `error: ENOENT` 退出 | 最终阶段 `FROM app AS merged` 基于 nginx 基底，没有 `WORKDIR`，cwd 是 `/`；`bun run --filter` 找不到 `/app/package.json` | Dockerfile 加 `WORKDIR /app`；start.sh 再 `cd /app` 兜底 |
| Cloud 导入期抛 `MINDWTR_CLOUD_CORS_ORIGIN must be set in production.` | 镜像 `ENV NODE_ENV=production`，而 `apps/cloud/src/server-config.ts` 在 production 下强制要求显式 origin | start.sh 用 `: "${MINDWTR_CLOUD_CORS_ORIGIN:=http://localhost:5173}"` 兜底；chart 侧会传真实应用域名 |

## 一键构建推送（推荐）

```bash
cd mindwtr/mindwtr-beta/mindwtr-merged

docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --build-arg CLOUD_IMAGE=ghcr.io/dongdongbh/mindwtr-cloud:1.3.7 \
  --build-arg APP_IMAGE=ghcr.io/dongdongbh/mindwtr-app:1.3.7 \
  --build-arg MCP_VERSION=1.1.13 \
  -t ghcr.io/inverstar/mindwtr-merged:1.3.7-r2 \
  -t ghcr.io/inverstar/mindwtr-merged:latest \
  --push \
  .
```

> **必须换新 tag**（这里是 `1.3.7-r2`）。Deployment 的 `imagePullPolicy: IfNotPresent`，
> 沿用 `1.3.7` 的话节点不会重新拉取，修好的镜像根本不会生效。
> 推完之后把 `mindwtr/values.yaml` 里的 `image.tag` 改成新 tag，重新打包 chart。

## 构建成功的标志

日志中必须出现以下两行，否则说明 patch 失败：

```
BUILD: Cloud /mcp passthrough patch applied.
[patch-cloud] PATCH OK: 已在 .../server.ts 插入 /mcp 与 /healthz 转发分支。
```

## 本地冒烟测试（推送前可选）

```bash
# 先只构建 amd64 本地测试，不推送
docker buildx build \
  --platform linux/amd64 \
  --load \
  --build-arg CLOUD_IMAGE=ghcr.io/dongdongbh/mindwtr-cloud:1.3.7 \
  --build-arg APP_IMAGE=ghcr.io/dongdongbh/mindwtr-app:1.3.7 \
  --build-arg MCP_VERSION=1.1.13 \
  -t mindwtr-merged:test \
  .

# 生成两个测试 token
CLOUD_TOKEN=$(openssl rand -hex 32)
MCP_TOKEN=$(openssl rand -hex 32)

docker run -d --name mindwtr-test -p 8080:8080 \
  -e MINDWTR_CLOUD_AUTH_TOKENS="$CLOUD_TOKEN" \
  -e MINDWTR_MCP_HTTP_TOKEN="$MCP_TOKEN" \
  -e MINDWTR_MCP_CLOUD_TOKEN="$CLOUD_TOKEN" \
  mindwtr-merged:test

# 等约 15s 后验证
sleep 15
curl -s http://localhost:8080/health   # 应返回 ok 或 {"status":"ok"}
curl -s http://localhost:8080/healthz  # 应返回 ok
curl -s http://localhost:8080/ready    # 应返回 ok

# 确认工作目录修好了（必须输出 /app，否则 bun --filter 会 ENOENT）
docker exec mindwtr-test sh -c 'pwd; ls -l /app/package.json'

# 确认三个进程都在
docker logs mindwtr-test 2>&1 | tail -20   # 应看到「全部就绪，对外入口端口 8080」

# 清理
docker rm -f mindwtr-test
```

## 镜像内容说明

| 来源 | 内容 | 进程 | 端口（容器内） |
|------|------|------|----------------|
| `mindwtr-app:1.3.7` | nginx + PWA 静态文件 | nginx | 0.0.0.0:**8080**（唯一对外） |
| `mindwtr-cloud:1.3.7` | bun 运行时 + Cloud 源码 | bun | 127.0.0.1:8787 |
| `mindwtr-mcp@1.1.13`（npm） | MCP helper | node | 127.0.0.1:8723 |
| `patch-cloud.py` | 构建期给 server.ts 注入 /mcp 转发 | — | — |

与官方镜像的区别：
- 官方提供 `mindwtr-app` 和 `mindwtr-cloud` 两个独立镜像，没有提供合并版
- `mindwtr-merged` 是我们把这两个镜像 + npm MCP helper 拼装在一起的结果
- 额外在 Cloud 的 server.ts 中注入了 /mcp 路由转发（`patch-cloud.py`）
- 用 `start.sh` 管理三个进程的有序启动和监督

