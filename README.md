# docker-frp

把 [frp](https://github.com/fatedier/frp) 的官方发布包直接打包成容器镜像，**镜像标签与 frp 官方版本号完全一致**。

| 镜像 | 说明 |
| --- | --- |
| `ghcr.io/jiangood/frps` | 服务端 |
| `ghcr.io/jiangood/frpc` | 客户端 |

标签：`0.71.0`、`v0.71.0`、`latest`（`latest` 只在构建上游最新版本时才打）。
平台：`linux/amd64`（默认只构建该架构，见文末）。

## 快速开始

### frps 服务端

不带任何参数启动时使用镜像内置配置（`bindPort = 7000`）：

```bash
docker run -d --name frps --restart always \
  -p 7000:7000 -p 7500:7500 \
  ghcr.io/jiangood/frps
```

生产环境请使用自己的配置，推荐 `network_mode: host`（frps 常需要监听 80/443 等端口，并且需要拿到真实客户端 IP）：

```bash
docker run -d --name frps --restart always --network host \
  -v $PWD/frps.toml:/etc/frp/frps.toml:ro \
  ghcr.io/jiangood/frps
```

配置放到别的路径时，把参数写在镜像名后面即可：

```bash
docker run -d --name frps --restart always --network host \
  -v $PWD/frps.toml:/data/frps.toml:ro \
  ghcr.io/jiangood/frps -c /data/frps.toml
```

### frpc 客户端

```bash
docker run -d --name frpc --restart always \
  -v $PWD/frpc.toml:/etc/frp/frpc.toml:ro \
  ghcr.io/jiangood/frpc
```

### docker compose

```yaml
# frps
services:
  frps:
    image: ghcr.io/jiangood/frps
    container_name: frps
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./frps.toml:/etc/frp/frps.toml:ro
```

```yaml
# frpc
services:
  frpc:
    image: ghcr.io/jiangood/frpc
    container_name: frpc
    restart: unless-stopped
    volumes:
      - ./frpc.toml:/etc/frp/frpc.toml:ro
```

### 常用命令

```bash
# 查看版本
docker run --rm ghcr.io/jiangood/frps --version

# 校验配置
docker run --rm -v $PWD/frps.toml:/data/frps.toml:ro \
  ghcr.io/jiangood/frps verify -c /data/frps.toml

# 进容器排查问题
docker run --rm -it --entrypoint sh ghcr.io/jiangood/frps

# 更新到最新版本
docker compose pull && docker compose up -d
```

## 镜像说明

- 基础镜像 `alpine:3.24`，已包含 `ca-certificates`。
- 二进制是官方 release 包（Dockerfile 用 `ADD` 直接下载官方 `tar.gz` 并解压），未做二次编译。
- 内置配置路径：`frps` → `/etc/frp/frps.toml`，`frpc` → `/etc/frp/frpc.toml`。
- 入口为 `ENTRYPOINT ["frps"]` + `CMD ["-c", "/etc/frp/frps.toml"]`：不加参数时用内置配置；写在镜像名后面的参数会原样传给 `frps`/`frpc`。
- 默认以 root 运行，因为 frps 经常需要绑定 80/443 等特权端口；只用高位端口时可以加 `--user 65534:65534`。
- 镜像内不含时区数据，日志时间为 UTC。

## 自动构建与发布

工作流：[`.github/workflows/build.yml`](.github/workflows/build.yml)

触发方式：

| 触发 | 行为 |
| --- | --- |
| push 到 `main` | 构建并推送 |
| push `v*.*.*` 标签 | 构建并推送该版本 |
| Pull Request | 只构建和测试，不推送 |
| 每周一 03:00 UTC（北京时间 11:00） | 检查 frp 是否有新版本，**有新版本才构建** |
| 手动触发 | 可指定版本、平台、是否强制重建 |

版本解析顺序：手动输入的版本 → 本次推送的 tag → frp 官方最新 release。
`latest` 标签只有在构建的版本等于官方最新版本时才会生成。

每周的定时任务会先查询 GHCR 中是否已存在该版本号的 manifest，已存在就直接跳过，不会重复构建。

每次构建都会执行以下校验，全部通过才会推送：

1. 镜像内 `frps --version` / `frpc --version` 输出与目标版本一致；
2. 内置配置文件存在且 `verify` 通过；
3. 真实联调：起一个 frps 和一个 frpc，用 frpc 把 frps 的 dashboard 端口（7500）转发到 16000，再从另一个容器穿过隧道抓取 `/api/serverinfo`，确认整条链路可用。

> 注意：GitHub 会在仓库连续 60 天没有任何提交活动后暂停定时工作流，届时在 Actions 页面重新启用即可。

## 本地构建

```bash
docker build --build-arg FRP_VERSION=0.71.0 -t frps:0.71.0 frps
docker build --build-arg FRP_VERSION=0.71.0 -t frpc:0.71.0 frpc
```

## 关于镜像可见性

GHCR 上的包首次推送后，可以在 GitHub → Packages → 选择对应的包 → **Package settings → Change visibility → Public** 设为公开，这样任何人都能直接 `docker pull`。

## 其他架构

默认只构建 `linux/amd64`。需要别的架构时手动触发工作流，把 `platforms` 改成例如 `linux/amd64,linux/arm64` 即可，
`frps/Dockerfile`、`frpc/Dockerfile` 里的映射关系为：

| 平台 | 官方发布包 |
| --- | --- |
| `linux/amd64` | `frp_<ver>_linux_amd64.tar.gz` |
| `linux/arm64` | `frp_<ver>_linux_arm64.tar.gz` |
| `linux/arm/v6`、`linux/arm/v7` | `frp_<ver>_linux_arm.tar.gz`（GOARM=5，v5/v6/v7 都能跑） |

armv7 如果想要更好的性能，可以改用官方硬浮点包 `frp_<ver>_linux_arm_hf.tar.gz`（GOARM=7）。

## 许可

镜像内的 frp 二进制由 [fatedier/frp](https://github.com/fatedier/frp) 发布，遵循 Apache-2.0 许可。
