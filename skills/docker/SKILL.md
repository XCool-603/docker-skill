---
name: docker
description: 'Use when writing, reviewing, or fixing a Dockerfile, .dockerignore, or docker-compose file, or when docker build/run/compose fails with permission denied, EACCES, ownership, entrypoint, packaging, layer-cache, or image-size problems. Covers build-time and runtime permission traps, bind-mount and named-volume ownership, multi-stage packaging, and running docker commands under the DSH sandbox on Windows.'
---

# 写对 Docker：权限、打包与验证

AI 生成的 Docker 出问题，绝大多数不是"不会写语法"，而是**只考虑了镜像里的一个身份**。镜像里始终有两套身份：

- **构建期**：默认 `root`，直到 `USER` 出现；
- **运行期**：你声明的非 root 用户（或者偷懒用了 root）。

所有 `EACCES` / `permission denied` / 打包反复出 bug，几乎都能归到"这两套身份之间没有交接清楚"。**改 Dockerfile 之前先判断问题属于哪一类，不要靠试。**

## 最小正确模板

新写 Dockerfile 时从这个骨架开始，它已经规避了下面大部分坑：

```dockerfile
FROM node:22-slim AS builder
WORKDIR /src
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-slim AS runtime
# 1) 固定 UID/GID，便于和宿主对齐（见"挂载与卷属主"）
ARG UID=10001
ARG GID=10001
RUN groupadd -g "${GID}" app \
 && useradd -m -u "${UID}" -g app -s /usr/sbin/nologin app

WORKDIR /app
ENV NODE_ENV=production \
    HOME=/home/app

# 2) 先建目录并交给 app，再拷产物；--chown 让运行期用户可写
RUN install -d -o app -g app /app
COPY --from=builder --chown=app:app /src/dist ./dist
COPY --chown=app:app --chmod=755 docker/entrypoint.sh /usr/local/bin/entrypoint.sh

# 3) USER 只出现一次，且是最后一步
USER app
EXPOSE 8080
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["node", "dist/server.js"]
```

配套 `docker/entrypoint.sh`（结尾必须 `exec "$@"`）：

```sh
#!/bin/sh
set -eu
exec "$@"
```

---

## 0. 先判断：是 Docker 的问题，还是 DSH 沙箱的问题

**这一步必须先做。** 在受限（sandboxed）会话里，docker 命令会因为打不开命名管道而失败——那不是 Dockerfile 写错，改 Dockerfile 是白改，这正是"反复改"的一个主要来源。

DSH 在 Windows 上的受限模式使用受限令牌，并向子进程管道的默认 DACL 合并一个 RESTRICTING SID，因此**受限进程无法打开命名管道**。Docker Desktop 的 CLI 正是通过 `\\.\pipe\dockerDesktopLinuxEngine`（或 `\\.\pipe\docker_engine`）连接 daemon 的，所以：

| 现象 | 结论 |
|---|---|
| `error during connect: ... open //./pipe/dockerDesktopLinuxEngine: Access is denied` | 沙箱边界，**不是** Dockerfile 问题 |
| `open //./pipe/docker_engine: Access is denied` / `EPERM` | 同上 |
| `[sandbox: file access denied under <mode> mode]` | 文件策略拒绝，换写法无用 |
| 连工作区列目录 / 写入都失败 | 属于文件权限问题，先用 `diagnose-windows-sandbox-acl` skill 修复，再谈 docker |

处理方式：

- **不要在受限模式下反复换命令写法重试**——边界不会因为换个调用方式而消失。
- 为**这一条**具体命令申请一次完全权限（full access），跑完就够；不要为此把整个会话放开。
- 构建上下文必须位于工作区内；`-v` 挂载工作区之外的路径同样需要完全权限。
- 受限模式下 `docker build` 的输出捕获也可能触发管道 `EPERM`。此时把输出重定向到文件（`docker build ... > build.log 2>&1`）或让 stdio 继承，而不是去改 Dockerfile。

判断依据是**报错特征**，不要靠猜当前模式。

---

## 1. 构建期权限（Dockerfile 内部）

**1.1 `USER` 只能出现一次，且放在最后。**
`USER app` 之后再 `RUN apt-get install ...` 必然 `EACCES`。所有需要 root 的步骤（装包、建用户、建目录、chmod）都排在 `USER` 之前。

**1.2 `COPY` 出来的文件默认属主是 `root:root`。**
运行期用户要写这些文件，就必须在 COPY 时改属主：

```dockerfile
COPY --chown=app:app . /app
```

不要用 `RUN chown -R app:app /app`——它多生成一整个数据层（镜像变大、构建变慢），而且顺序写错（chown 之后再 COPY）就完全失效。`--chown` 只管 COPY 进来的内容，所以**在镜像里新建的目录要单独 `RUN install -d -o app -g app /app`**。

**1.3 建用户：Debian/Ubuntu 与 Alpine 参数不同，别混用。**

```dockerfile
# Debian / Ubuntu
RUN groupadd -g 10001 app \
 && useradd -m -u 10001 -g app -s /usr/sbin/nologin app

# Alpine（busybox adduser，参数不一样）
RUN addgroup -g 10001 -S app \
 && adduser -u 10001 -S -G app -h /app -s /sbin/nologin app
```

`-m` / `-h` 必须给：**用户没有 HOME 时，很多工具写 `~/.cache`、`~/.npm`、`~/.config` 会直接失败**，报错位置离根因很远。若不想建 home，就显式 `ENV HOME=/app XDG_CACHE_HOME=/tmp/.cache` 并保证该目录可写。

**1.4 包管理器在非 root 下会失败。**
- `npm ci` / `npm i -g` 需要 root 或可写 prefix → 在 root 阶段装完再切 `USER`。
- Python 用 venv，并修正 PATH，否则切用户后找不到解释器：
  ```dockerfile
  RUN python -m venv /opt/venv
  ENV PATH="/opt/venv/bin:$PATH"
  ```
- BuildKit 缓存挂载给非 root 用时必须给 uid/gid，否则缓存目录属主是 root，装包失败：
  ```dockerfile
  RUN --mount=type=cache,target=/root/.npm,uid=10001,gid=10001 npm ci
  ```

**1.5 apt 必须非交互且清缓存。**

```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates tzdata \
 && rm -rf /var/lib/apt/lists/*
```

不加 `DEBIAN_FRONTEND=noninteractive`（或 `-y` 配合）时，`tzdata` 之类的交互提示会让构建挂住或失败。

**1.6 入口脚本的执行位——Windows 宿主上最常见的坑。**
在 Windows 上 git 常常不保留可执行位，`COPY` 进镜像后脚本没有 `+x`，容器一启动就 `permission denied`。用 `--chmod` 直接固化（BuildKit，Docker 23+ 默认启用）：

```dockerfile
COPY --chown=app:app --chmod=755 entrypoint.sh /usr/local/bin/entrypoint.sh
```

老版本 Docker 用 `RUN chmod +x /usr/local/bin/entrypoint.sh`（放在 `USER` 之前）。

**1.7 非 root 不能监听 1024 以下端口。**
容器内监听 `8080`，对外映射 `-p 80:8080`。别用 `setcap` 绕，那会削弱镜像安全性。

---

## 2. 运行期权限

**2.1 入口脚本用 exec 形式并 `exec "$@"`。**

```dockerfile
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["node", "dist/server.js"]
```

脚本结尾必须是 `exec "$@"`，否则 PID 1 是 shell，`SIGTERM` 到不了应用，`docker stop` 要等超时才杀，表现为"打包没问题但停不干净"。需要回收僵尸进程时加 `docker run --init`。

**2.2 只读根文件系统。**
`--read-only --tmpfs /tmp` 能挡掉"应用偷偷写根目录"的问题。启用前先确认应用只写显式挂载点；启动报 `Read-only file system` 时，加 tmpfs 或卷，**不要**退回可写根目录了事。

**2.3 不要用 `chmod 777` 或退回 root 来"解决"权限。**
那只是把症状藏起来，属主问题依然存在，而且会让镜像审计不通过。需要共享写入时，用固定 GID 配合 `group_add`。

---

## 3. 挂载与卷的属主（反复改的重灾区）

**绑定挂载会盖掉镜像里的属主。** 这是最容易让人误判成"Dockerfile 写错了"的一类：

```bash
docker run -v "$PWD:/app" app:verify
```

挂载后 `/app` 的属主变成宿主目录的 UID，镜像里 `COPY --chown=app:app` 的效果**全部失效** → 非 root 进程写不进去 → `EACCES`。此时改 Dockerfile 是没用的。三种正确解法，按推荐顺序：

**解法 1（首选）：用命名卷。**
命名卷**首次创建且为空**时，Docker 会把镜像中该路径下的内容**连同属主和权限**一起复制进卷。所以只要该路径在镜像里存在且属主正确，非 root 用户就能正常写：

```yaml
services:
  app:
    volumes:
      - appdata:/app/data
volumes:
  appdata: {}
```

⚠️ 前提是**挂载点在镜像里已经存在且属主正确**。如果挂载的是镜像中不存在的路径，卷会是 `root:root`，属主不会自动对齐——先在 Dockerfile 里 `RUN install -d -o app -g app /app/data`。

**解法 2：对齐 UID/GID。**

```bash
docker run --user "$(id -u):$(id -g)" app:verify
```

```yaml
services:
  app:
    user: "${UID:-1000}:${GID:-1000}"
```

前提是镜像里该 UID 对工作目录有读写权限。

**解法 3：构建期参数化 UID**，让镜像用户和宿主一致（模板里的 `ARG UID` / `ARG GID` 就是这个用途）。

**3.1 compose 默认以 root 运行**，在宿主机上生成的文件会变成 root 属主，之后宿主上的普通用户就改不动了。要么显式 `user:`，要么明确接受这个后果。

**3.2 挂 docker socket 时组 ID 常不一致**，容器内 `docker` 组的 GID 与宿主不同 → 用 `group_add: ["${DOCKER_GID}"]`。

---

## 4. 多阶段构建与打包

**4.1 `COPY --from=<stage>` 的源路径相对的是那个阶段的根，不是它的 `WORKDIR`。**

```dockerfile
FROM node:22-slim AS builder
WORKDIR /src
RUN npm ci && npm run build
FROM node:22-slim AS runtime
WORKDIR /app
COPY --from=builder /src/dist ./dist   # 绝对路径，别写 ./dist
```

**4.2 最终阶段只拷产物**，不要把编译器、devDependencies、包管理器缓存带进去（体积和攻击面都会涨）。

**4.3 多架构：`TARGETARCH` 必须在每个使用它的阶段内重新 `ARG` 声明。**

```dockerfile
ARG TARGETARCH
RUN case "$TARGETARCH" in amd64) ARCH=x64 ;; arm64) ARCH=arm64 ;; esac \
 && curl -fsSLo /tmp/bin "https://example.com/tool-${ARCH}"
```

让 builder 跑在原生架构（避免 QEMU 拖慢几十倍）：`FROM --platform=$BUILDPLATFORM node:22-slim AS builder`。

**4.4 distroless / scratch 里没有 shell。**
不能 `RUN chmod`、不能用 shell 形式 `ENTRYPOINT`、`HEALTHCHECK` 也不能用 `sh -c`。选它就必须用 exec 形式直接跑二进制，并在 builder 阶段把所有准备工作做完。

**4.5 可复现性**：提交锁文件并强制使用（`npm ci`、`pnpm install --frozen-lockfile`、`poetry install --no-root`），基础镜像尽量钉 digest，避免"昨天能构建今天不能"。

**4.6 密钥绝不要用 `ARG` / `ENV`**——会留在镜像历史和层里。用 BuildKit secret：

```dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```

---

## 5. `.dockerignore` 与层缓存

**5.1 漏排 `node_modules` / `.git` 是双重灾难**：上下文巨大、构建极慢；更糟的是宿主的 `node_modules` 会覆盖容器内的，导致原生模块 ABI 不匹配的诡异报错。

**5.2 但别过度排除。** 构建真正需要的文件（`.env`、生成的代码、`schema.prisma`、`Makefile`）被排除后，报错是"file not found"，很容易被误判成代码问题。排除 `.git` 还会让用 `git describe` 打版本号的步骤失败。

**5.3 依赖先装、源码后拷，才能命中层缓存。**

```dockerfile
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
```

反例：`COPY . .` 写在 `RUN npm ci` 之前，改任何一行源码都会重装依赖——这是"每次构建都特别慢"的根因。

**5.4 排查缓存**：`docker build --progress=plain` 看是哪一层 `CACHED` 没命中。注意 `.dockerignore` 自身的改动也会让缓存失效。

---

## 6. Windows 宿主特有

- **执行位 / CRLF**：见 1.6。另外在 `.gitattributes` 里加 `*.sh text eol=lf`，否则 CRLF 会让 shebang 变成 `/bin/sh^M`，报 `bad interpreter: No such file or directory`。
- **路径**：Dockerfile 内一律用 `/`，不要写反斜杠。
- **绑定挂载到 Windows 文件系统**（WSL2 下的 `/mnt/c/...`）时文件权限是模拟的，`chmod` 可能不生效。把构建上下文放在 WSL 自己的文件系统里，既快得多，权限行为也一致得多。

---

## 7. 交付前必须验证（打破"反复改"）

**写完 Dockerfile 不算完成。** 必须真的构建并运行过，否则就是在拿用户当编译器。按顺序跑：

```bash
docker build --progress=plain -t app:verify .
docker run --rm app:verify id -u                                   # 期望：非 0
docker run --rm app:verify sh -c 'touch "$PWD/.w" && rm "$PWD/.w"' # 期望：成功
docker run --rm app:verify <真实启动命令>                            # 期望：正常起来
```

带挂载的，再补一次挂载场景（这是唯一能验证属主是否对齐的方式）：

```bash
docker run --rm -v appdata:/app/data app:verify sh -c 'touch /app/data/.w && rm /app/data/.w'
```

compose 项目还要：

```bash
docker compose config          # 校验语法与变量插值，不启动容器
docker compose up -d
docker compose logs --tail=50
docker compose down
```

**受限模式下这些命令会被命名管道挡住** → 回到第 0 节：申请一次完全权限把验证跑完，而不是把 Dockerfile 改得"看起来更安全"。

如果环境里没有 docker（例如纯 Windows 主机未装 Docker Desktop），**明确说明"未经构建验证"**，不要声称已验证通过。

---

## 8. 症状 → 根因速查

| 报错 / 现象 | 根因 | 修法 |
|---|---|---|
| `RUN` 步骤里 `Permission denied` | `USER` 切换过早 | `USER` 移到最后 |
| 容器启动即 `permission denied: ./entrypoint.sh` | 脚本缺执行位（Windows 宿主常见） | `COPY --chmod=755` 或 `RUN chmod +x` |
| `/bin/sh^M: bad interpreter` | CRLF 行尾 | `.gitattributes` 设 `*.sh text eol=lf` |
| 运行期 `EACCES: ... open '/app/...'` | COPY 未 `--chown`，或绑定挂载盖掉属主 | `COPY --chown` / 换命名卷 / 对齐 UID |
| 写 `~/.cache`、`$HOME` 失败 | 用户没有 HOME 或 HOME 不可写 | `useradd -m` 或 `ENV HOME=...` |
| `bind: permission denied` 监听 80 | 非 root 不能绑 <1024 | 监听 8080 + `-p 80:8080` |
| 卷内文件属主是 `root` | 挂载点在镜像中不存在 | 镜像里 `install -d -o app -g app` |
| 容器里出现宿主的 `node_modules` | `.dockerignore` 漏排 | 加 `node_modules`、`.git` |
| 改一行源码就重装依赖 | COPY 顺序错 | 先拷依赖清单再 `COPY . .` |
| 构建上下文几百 MB / 极慢 | 缺 `.dockerignore` | 补 `.dockerignore` |
| `exec format error` | 架构不匹配 | 补 `--platform` / `TARGETARCH` |
| `docker stop` 要等超时 | PID 1 是 shell | 入口脚本 `exec "$@"`、`--init` |
| `open //./pipe/docker*: Access is denied` | DSH 受限模式（**非 Dockerfile 问题**） | 见第 0 节，申请完全权限 |

---

## 改 Dockerfile 之前先问自己

1. 这个报错是**构建期**还是**运行期**的？两套身份不一样。
2. 是**镜像内**的问题，还是**挂载**把属主盖掉了？
3. 是 Docker 的问题，还是**沙箱**不让 docker 说话？（第 0 节）
4. 我这次改动**验证过**了吗？没验证就不要说"修好了"。
