# Gitea Docker Compose

这个仓库使用 Docker Compose 部署 Gitea 和 Gitea Actions runner。runner 使用 Docker-in-Docker（DinD）模式执行 Actions 任务。

## 组件

| 服务 | 作用 | 镜像 |
| --- | --- | --- |
| `server` | Gitea Web 服务、Git 仓库和 Actions 服务端 | `docker.gitea.com/gitea:28.2.0` |
| `runner` | 执行 Gitea Actions 工作流 | `docker.io/gitea/runner:5-dind` |

当前 Compose 文件发布 Gitea HTTP 端口 `3000:3000`。runner 使用 `privileged: true` 和 `runner-docker` named volume 启动内部 Docker daemon。

## 前置条件

- 已安装并运行 Docker Desktop/Docker Engine；
- Docker Compose v2；
- Gitea runner 注册 token；
- 宿主机防火墙允许访问 TCP `3000`；
- 如果使用 DinD，runner 任务容器必须能够访问 `GITEA_INSTANCE_URL`。

## 配置

复制示例环境文件：

```powershell
Copy-Item .env.example .env
```

然后编辑 `.env`：

```dotenv
GITEA_RUNNER_REGISTRATION_TOKEN=替换为实际的runner注册token
GITEA_INSTANCE_URL=http://192.168.0.46:3000/
```

`192.168.0.46` 应替换为运行 Docker 的宿主机局域网地址。不要在 DinD 模式下使用 `http://server:3000/`、`http://localhost:3000/` 或 `http://127.0.0.1:3000/`。

`server` 只能在 Compose 网络中的容器内解析，而 Actions 任务是在 DinD 创建的内层容器中执行，内层容器通常无法解析 `server`。

Gitea 的 `ROOT_URL` 也建议配置为同一个任务容器能够访问的地址，例如：

```ini
ROOT_URL = http://192.168.0.46:3000/
```

不要把真实 token 提交到 Git。`.env` 已经被 `.gitignore` 忽略，`.env.example` 只保留变量名和空值。

## 启动和更新

检查 Compose 配置：

```powershell
docker compose config
```

启动服务：

```powershell
docker compose up -d
```

修改 Compose 或环境变量后，强制重建 runner：

```powershell
docker compose up -d --force-recreate runner
```

查看状态和日志：

```powershell
docker compose ps
docker compose logs -f server
docker compose logs -f runner
```

浏览器访问：`http://localhost:3000/`

## 数据和迁移

| 路径/卷 | 内容 | 是否必须迁移 |
| --- | --- | --- |
| `./data` | Gitea 数据库、仓库、附件、配置和 SSH 密钥 | 是 |
| `./data-runner` | runner 注册信息和配置 | 想保留原 runner 身份时需要 |
| `runner-docker` | DinD 的镜像、容器和构建缓存 | 非必须，可重新创建 |

`runner-docker` 主要保存缓存。迁移到新主机时可以不迁移它，首次运行会重新下载 Actions 所需镜像；不要使用 `docker compose down -v`，否则会删除 Compose 管理的 named volume。

如果把 runner 的数据目录改成其他路径，例如 `./runner-data:/data`，需要一并迁移原 `data-runner` 内容。建议让 runner 数据目录与 Gitea 的 `./data` 保持并列，避免把 runner 数据嵌套在 Gitea 数据目录中。

## 常见问题

### `Could not resolve host: server`

这表示 Actions checkout 使用了 `http://server:3000/`，但内层任务容器无法解析 Compose 服务名。检查 `.env` 中的 `GITEA_INSTANCE_URL`，并确认日志中的仓库地址已经变成宿主机可访问的地址，例如：

```text
http://192.168.0.46:3000/组织名/仓库名/
```

修改后重建 runner。如果日志仍显示 `server`，旧 runner 注册配置可能保存在 `data-runner/.runner` 中，需要备份该目录后重新注册 runner。

### `git failed with exit code 128`

如果日志中同时出现 `/usr/bin/git`，说明 Git 已安装。应继续查看同一段日志中的网络、DNS、认证或仓库地址错误；单独的 exit code 128 不是“找不到 Git”的证据。

### runner 无法注册

检查以下项目：

1. `.env` 中的 `GITEA_RUNNER_REGISTRATION_TOKEN` 不为空且仍然有效；
2. `GITEA_INSTANCE_URL` 能从 runner 任务容器访问；
3. Gitea Actions 已启用；
4. Windows 防火墙没有阻止 TCP `3000`。

## 安全建议

- 不要提交 `.env`、runner 注册 token 或 `data` 目录；
- 如果 token 曾经出现在公开日志、截图或提交记录中，应在 Gitea 中重新生成；
- `privileged: true` 是 DinD 方案常见要求，只应在可信的 Docker 主机上运行；
- 定期备份 `data` 和 runner 配置，并测试恢复流程。
