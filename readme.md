# Shiroi Dokploy Deploy

使用 GitHub Actions 构建 Shiroi Docker 镜像并通过 Dokploy 部署。

## 为什么？

Shiroi 是 [Shiro](https://github.com/Innei/Shiro) 的闭源开发版本。

由于 Next.js 构建需要大量内存，很多服务器无法承受这样的开销。此项目利用 GitHub Actions 构建 Docker 镜像，推送到 GitHub Container Registry (GHCR) 私有仓库，然后通过 Dokploy API 触发部署。

## 工作流程

```
Shiroi 仓库 push
       ↓
GitHub Actions 构建镜像
       ↓
推送到 GHCR (私有)
       ↓
调用 Dokploy API 触发部署
       ↓
Dokploy 从 GHCR 拉取镜像并部署
```

## 配置步骤

### 1. Dokploy 配置

#### 添加 GHCR Registry

在 Dokploy Dashboard → Settings → Registry → Add Registry：

| 字段 | 值 |
|------|-----|
| Name | `ghcr` |
| Registry URL | `ghcr.io` |
| Username | 你的 GitHub 用户名 |
| Password | GitHub PAT (需要 `read:packages` 权限) |

#### 创建 Application

1. 创建新的 Application，选择 **Docker** 类型
2. 配置镜像：
   - Registry: 选择刚才添加的 `ghcr`
   - Image: `ghcr.io/<your-username>/shiroi`
   - Tag: `latest`
3. 配置环境变量（参考 Shiroi 文档）
4. 配置端口映射、数据卷等

#### 获取 Application ID

从 Dokploy URL 中提取，例如：
```
https://dokploy.example.com/dashboard/project/.../services/application/hdoihUG0FmYC8GdoFEc
                                                                      └─────────────────┘
                                                                        这就是 App ID
```

#### 生成 API Token

Dokploy Dashboard → Profile → Generate API Key

### 2. GitHub Secrets 配置

在此仓库的 Settings → Secrets and variables → Actions 中添加：

| Secret | 说明 |
|--------|------|
| `GH_PAT` | 可访问 Shiroi 私有仓库的 GitHub Token (需要 `repo` 权限) |
| `DOKPLOY_URL` | Dokploy 实例地址，如 `https://dokploy.example.com` (不带尾部斜杠) |
| `DOKPLOY_API_TOKEN` | Dokploy 生成的 API Key |
| `DOKPLOY_APP_ID` | Dokploy Application ID |
| `AFTER_DEPLOY_SCRIPT` | (可选) 部署后执行的脚本 |
| `BASE_URL` | Docker **build-arg**，站点根 URL（建议无尾部斜杠）；与私有 `Dockerfile` 中 `ARG BASE_URL` 一致 |

#### `BASE_URL`、ISR 与 `NEXT_PUBLIC_*`

当前 Shiroi `Dockerfile` 典型写法为：`ARG BASE_URL`，并在 builder 阶段令 `NEXT_PUBLIC_GATEWAY_URL=${BASE_URL}`、`NEXT_PUBLIC_API_URL=${BASE_URL}/api/v2`。CI 只需传入 **`BASE_URL`** 作为 build-arg，`next build` 即可得到一致的公开端点。

若站点启用 **ISR**，构建期仍会按上述 URL 访问后端；仅依赖运行时的容器环境变量无法覆盖已写入 bundle 的构建期取值，因此 **`BASE_URL` 必须在镜像构建时传入**，并与 Dokploy 中运行时配置的站点根 URL 一致。

构建期密钥类变量（如 `S3_*`、`TMDB_API_KEY`、`GH_TOKEN`、`WEBHOOK_SECRET`）在 `Dockerfile` 中多为独立 `ARG`，若构建命令需要它们，请在 **同一 `build-args` 块**中增加对应 Secrets（勿写入仓库明文）。

### 3. 创建 GitHub PAT

1. 进入 [GitHub Settings → Tokens](https://github.com/settings/tokens)
2. Generate new token (classic)
3. 选择权限：
   - `repo` - 访问私有仓库
   - `read:packages` - 读取 packages (Dokploy 拉取镜像用)
   - `write:packages` - 写入 packages (CI 推送镜像用，会自动获得)

## 触发部署

- **自动触发**：Push 到 `main` 分支
- **手动触发**：使用 `repository_dispatch` 事件

## 镜像管理

镜像存储在 GHCR 私有仓库：`ghcr.io/<username>/shiroi`

每次构建会生成两个 tag：
- `latest` - 最新版本
- `<commit-sha>` - 对应的 commit hash

## 故障排除

### 镜像推送失败

1. 检查 `GITHUB_TOKEN` 权限是否包含 `packages:write`
2. 确认仓库 Settings → Actions → General → Workflow permissions 设置为 "Read and write permissions"

### Dokploy 部署失败

1. 检查 `DOKPLOY_URL` 是否正确（不带尾部斜杠）
2. 确认 `DOKPLOY_API_TOKEN` 有效
3. 确认 `DOKPLOY_APP_ID` 正确
4. 检查 Dokploy 中的 Registry 凭证是否正确

### Dokploy 拉取镜像失败

1. 确认 Dokploy 中配置的 Registry 凭证正确
2. 确认 PAT 有 `read:packages` 权限
3. 检查镜像地址是否正确：`ghcr.io/<username>/shiroi:latest`
