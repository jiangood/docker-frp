# docker-frp

把 [frp](https://github.com/fatedier/frp) 的官方发布包打包成容器镜像，镜像标签与 frp 版本号保持一致。

| 镜像 | 说明 |
| --- | --- |
| `ghcr.io/jiangood/frps` | 服务端 |
| `ghcr.io/jiangood/frpc` | 客户端 |

标签即 frp 版本号（`X.Y.Z` 与 `vX.Y.Z`），另有 `latest`（只在构建上游最新版本时才打）；已构建的版本见 [versions.txt](versions.txt)。平台：`linux/amd64`。

## 使用

frps（服务端），不带参数时使用内置配置（`bindPort = 7000`）：

```bash
docker run -d --name frps --restart always --network host \
  -v $PWD/frps.toml:/etc/frp/frps.toml \
  ghcr.io/jiangood/frps
```

建议加 `--network host`：frps 需要监听 7000，以及 80/443 等 vhost 端口，同时能拿到真实客户端 IP。只用内置配置时去掉 `-v` 即可。

frpc（客户端）：

```bash
docker run -d --name frpc --restart always \
  -v $PWD/frpc.toml:/etc/frp/frpc.toml \
  ghcr.io/jiangood/frpc
```

docker compose：

```yaml
services:
  frps:
    image: ghcr.io/jiangood/frps
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./frps.toml:/etc/frp/frps.toml
```

```yaml
services:
  frpc:
    image: ghcr.io/jiangood/frpc
    restart: unless-stopped
    volumes:
      - ./frpc.toml:/etc/frp/frpc.toml
```

常用命令：

```bash
docker run --rm ghcr.io/jiangood/frps --version              # 查看版本
docker run --rm -v $PWD/frps.toml:/etc/frp/frps.toml \
  ghcr.io/jiangood/frps verify -c /etc/frp/frps.toml        # 校验配置
docker run --rm -it --entrypoint sh ghcr.io/jiangood/frps    # 进容器排查
docker compose pull && docker compose up -d                  # 更新到最新版
```

## 镜像说明

- 基础镜像 `alpine:3.24`，二进制取自官方 release，未二次编译。
- 内置配置：`/etc/frp/frps.toml`、`/etc/frp/frpc.toml`，挂载同名文件即可覆盖。
- 入口为 `ENTRYPOINT ["frps"]` + `CMD ["-c", "/etc/frp/frps.toml"]`，写在镜像名后面的参数会原样传给二进制。
- 默认以 root 运行（frps 经常需要绑定 80/443 等特权端口），只用高位端口时可加 `--user 65534:65534`。
- 镜像内不含时区数据，日志时间为 UTC。

## 自动构建

工作流：[`.github/workflows/build.yml`](.github/workflows/build.yml)

| 触发 | 行为 |
| --- | --- |
| push 到 `main` 或 `v*.*.*` tag | 构建并推送 |
| Pull Request | 只构建测试，不推送 |
| 每周一 03:00 UTC（北京时间 11:00） | 上游有新版本时才构建 |
| 手动触发 | 可指定版本、平台、是否强制重建 |

版本解析顺序：手动输入 → 本次推送的 tag → frp 官方最新 release。每周的定时任务会先查 GHCR 里是否已存在该版本，已存在就直接跳过。构建前会校验镜像内版本号、内置配置 `verify` 以及 frps 默认配置启动是否成功，全部通过才会推送。推送成功后，工作流会把该版本号写入 [versions.txt](versions.txt) 并提交回默认分支。

> GitHub 会在仓库连续 60 天没有提交活动后暂停定时工作流，届时在 Actions 页面重新启用即可。

## 本地构建

```bash
VERSION=<版本号>   # 例如上游最新 release
docker build --build-arg FRP_VERSION="$VERSION" -t frps:"$VERSION" frps
docker build --build-arg FRP_VERSION="$VERSION" -t frpc:"$VERSION" frpc
```

默认只构建 `linux/amd64`，需要其他架构时手动触发工作流修改 `platforms`。Dockerfile 里的映射关系：

| 平台 | 官方发布包 |
| --- | --- |
| `linux/amd64` | `frp_<ver>_linux_amd64.tar.gz` |
| `linux/arm64` | `frp_<ver>_linux_arm64.tar.gz` |
| `linux/arm/v6`、`linux/arm/v7` | `frp_<ver>_linux_arm.tar.gz`（GOARM=5，v5/v6/v7 通用） |

> 镜像内的 frp 二进制由 [fatedier/frp](https://github.com/fatedier/frp) 发布，遵循 Apache-2.0 许可。
