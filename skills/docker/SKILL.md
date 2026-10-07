---
name: docker
description: 'Use when writing, reviewing, or fixing a Dockerfile, .dockerignore, or docker-compose file; when docker build/run/compose fails with permission denied, EACCES, ownership, packaging, layer-cache, container-name conflict, or a container cannot reach a host service; or when a deploy wrapper script (e.g. `sh scripts/docker.sh deploy`) is proposed or exists — this skill forbids wrapper scripts. Also covers bind-mount and named-volume ownership, root-to-non-root upgrades, compose naming and host.docker.internal, and running docker under the DSH sandbox on Windows.'
---

# 写对 Docker：权限、打包与验证

AI 生成的 Docker 出问题，绝大多数不是"不会写语法"，而是**只考虑了镜像里的一个身份**。镜像里始终有两套身份：

- **构建期**：默认 `root`，直到 `USER` 出现；
- **运行期**：你声明的非 root 用户（或者偷懒用了 root）。

所有 `EACCES` / `permission denied` / 打包反复出 bug，几乎都能归到"这两套身份之间没有交接清楚"。**改 Dockerfile 之前先判断问题属于哪一类，不要靠试。**

## 开工前必做：确认基镜像是哪个发行版、以及它**已经有哪些用户**

**建用户/建组的命令是发行版专属的，用错一定构建失败。** 这一步优先于后面所有规则——先做它，再写别的。

```bash
docker run --rm <你的基础镜像> sh -c 'command -v useradd groupadd adduser addgroup; echo "---"; head -2 /etc/os-release; echo "--- 已存在的用户/组:"; id; getent passwd | tail -3; echo "APP_UID=$APP_UID"'
```

**后半段（已存在的用户）和前半段同样重要**：很多官方镜像已经内置了非 root 用户，
你却去建一个**同名**用户 —— 报错是 `groupadd` 退出码 9（组名已存在），
而 Docker 只打印一行 `did not complete successfully: exit code: 9`，看不出任何原因。

已知的内置非 root 用户：

| 基础镜像 | 内置用户 | 备注 |
|---|---|---|
| `node:*` | `node`（1000:1000） | |
| `nginx:*` | `nginx` | |
| `postgres:*` | `postgres` | |
| **`mcr.microsoft.com/dotnet/aspnet` / `runtime`（8.0+）** | **`app`（1654:1654）** | 名字就叫 `app`，与本文档模板里的用户名**正好撞车**；是 `--no-create-home` 建的，HOME 要自己建 |

| 输出 | 基镜像 | 用哪套 |
|---|---|---|
| 有 `useradd`、`groupadd` | Debian / Ubuntu | 下方模板（GNU 版） |
| 只有 `adduser`、`addgroup` | Alpine | 1.3 的 Alpine 版 |
| 两个都没有 | distroless / scratch | 根本不能建用户，见 4.4 |

**这一步没做时，报错长这样**——出现任何一条，都说明你跳过了它，而问题不在权限配置上：

| 报错 | 真实原因 |
|---|---|
| `groupadd: not found` / `useradd: not found` | 基镜像是 Alpine。`useradd` 由 `shadow` 包提供，Alpine 基础镜像**默认没装** |
| `adduser: Unknown option` / `addgroup: invalid option` | 基镜像是 Debian，却在用 busybox 参数 |
| `adduser: uid '1000' is in use` | 该 UID 已被基镜像占用（`node`、`ubuntu` 镜像都用 1000:1000） |
| `groupadd: Permission denied` | 前面的步骤已经切过 `USER`，当前不是 root |
| `did not complete successfully: exit code: 9` | `groupadd` 的「**组名已存在**」——你要建的组名撞上了镜像内置的组（如 .NET 镜像的 `app`） |

> 💡 **最省事的做法：别自己建用户。** 官方镜像通常已经备好非 root 用户——
> `node` 镜像有 `node`（UID 1000）、`nginx` 有 `nginx`、`postgres` 有 `postgres`、
> **.NET 镜像有 `app`（UID 1654）**。直接 `USER node` / `USER app` 就能一次性绕开整套方言与撞号问题。
> 本文档下面的模板是"**没有**内置用户时"的写法，用之前先确认目标镜像里确实没有同名用户。

> 🔍 **`did not complete successfully: exit code: N` 本身不告诉你任何事。**
> 把那条 `RUN` 的命令**原样**在容器里跑一遍就能看到真实报错：
>
> ```bash
> docker run --rm <基础镜像> sh -c 'groupadd -g 10001 app; echo "退出码=$?"'
> ```
>
> 建用户/建组类命令的退出码是标准化的（`groupadd`：3=参数非法、4=GID 重复、9=组名重复；
> `useradd`：4=UID 重复、9=用户名重复、12=建 home 失败）。**看到码先查语义，别急着改 Dockerfile。**

---

## 最小正确模板（**Debian / Ubuntu 基镜像**）

> ⚠️ 这个骨架假定 `FROM` 是 Debian 系。**如果你的 `FROM` 是 Alpine，下面创建用户那两行必须换成 1.3 的 Alpine 版**——照抄会直接报 `groupadd: not found`。
> 这就是"每次都在同一步失败"最常见的原因：模板的发行版和实际 `FROM` 对不上。

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
#    ⚠️ 下面两行是 Debian 版。Alpine 基镜像必须换成 1.3 的 Alpine 版。
#    用 10001 而不是 1000：node/ubuntu 镜像已占用 1000，代入 1000 必报 uid in use（见 1.3.1）
#    ⚠️ 若这两个号是 --build-arg 从宿主传进来的（CI/部署流程常这么干，传的就是 1000），
#       上面这个写法在用户的服务器上必挂 —— 换成 1.3.3 的容忍写法。
#    ⚠️ 名字必须叫 APP_UID/APP_GID，不能叫 UID/GID：RUN 里变量由 shell 展开，
#       而 UID 在 bash 里是只读内置变量（root 下为 0），会盖掉 ARG（见 1.8）。
ARG APP_UID=10001
ARG APP_GID=10001
RUN groupadd -g "${APP_GID}" app \
 && useradd -m -u "${APP_UID}" -g app -s /usr/sbin/nologin app

WORKDIR /app
ENV NODE_ENV=production \
    HOME=/home/app

# 2) 先建目录并交给 app，再拷产物；--chown 让运行期用户可写
#    用 mkdir + chown，不用 install -d -o/-g —— 前者在 Debian 和 Alpine 上行为一致
RUN mkdir -p /app && chown app:app /app
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

## 执行 docker：用原生命令，不要包装脚本

**禁止用 `sh scripts/docker.sh deploy` 这类包装脚本。** 直接调 `docker` / `docker compose`：

```bash
docker compose up -d --build
docker compose logs -f --tail=100
docker compose down
```

四条理由，每条都对应一次实际的排查成本：

- **报错被藏了一层**。脚本失败时你看到的是脚本自己的行号和退出码，而不是 docker 的原始报错；
  每轮排查都要先"穿过"脚本，这是"反复改"的直接来源之一。
- **脚本本身是新的故障面**。行尾、执行位、参数转义、`set -e` 行为，每一样都能单独炸
  （第 6 节整节都是脚本在 Windows 上翻车的场景）。
- **脚本会固化错误的值**。典型是把宿主 UID `1000` 硬编码进 `--build-arg`，
  在别人的服务器上必然撞号（见 1.3.1、1.3.3）。
- **它不提供任何 docker 没有的能力**。绝大多数包装脚本只是把几条命令拼起来。

需要固定流程时，用 docker 自己的机制，而不是自己再写一层：

| 想固定什么 | 用什么 |
|---|---|
| 启动顺序 | `depends_on` + `healthcheck` |
| 构建参数 | `docker-compose.yml` 的 `build.args` |
| 环境变量 | `env_file` / `environment` |
| 重建策略 | `docker compose up -d --build` |
| 定时更新 | 宿主 crontab 直接写 `docker compose` 命令，而不是调用脚本 |

**已经存在的包装脚本**：删掉，把里面真正需要的命令收敛进 `docker-compose.yml`。
不要"留着但不用"——留着就会被下一个 AI 或下一个人接着用。

> ⚠️ 不用脚本**不等于**撞号问题消失：`build.args` 里照样可能传进 1000。
> 所以 1.3.3 的"Dockerfile 自己容忍撞号"依然必须做。

---

## 1. 构建期权限（Dockerfile 内部）

**1.1 `USER` 只能出现一次，且放在最后。**
`USER app` 之后再 `RUN apt-get install ...` 必然 `EACCES`。所有需要 root 的步骤（装包、建用户、建目录、chmod）都排在 `USER` 之前。

**1.2 `COPY` 出来的文件默认属主是 `root:root`。**
运行期用户要写这些文件，就必须在 COPY 时改属主：

```dockerfile
COPY --chown=app:app . /app
```

不要用 `RUN chown -R app:app /app`——它多生成一整个数据层（镜像变大、构建变慢），而且顺序写错（chown 之后再 COPY）就完全失效。`--chown` 只管 COPY 进来的内容，所以**在镜像里新建的目录要单独 `RUN mkdir -p /app && chown app:app /app`**。

**1.3 建用户：Debian/Ubuntu 与 Alpine 是两套互不兼容的方言，混用必定在 `[2/N]` 这一步构建失败。**

> ⚠️ 这是整份 skill 最容易踩坏的一步。两段代码长得像，参数却完全不通用：
> 把 `-D` / `-G` / `-h` 这类 busybox 参数用在 Debian 上，或把 `-m` 用在 Alpine 上，
> 都会直接报 `Unknown option` / `invalid option`。
> **先看 `FROM` 是哪个发行版，再选对应的一段，不要凭印象拼。**

```dockerfile
# ── Debian / Ubuntu（node:22-slim、python:3.12-slim、ubuntu…）──
# 注意：Debian 的 adduser 是 perl 脚本，参数与 busybox 完全不同，别用 adduser
RUN groupadd -g "${APP_GID}" app \
 && useradd -m -u "${APP_UID}" -g app -s /usr/sbin/nologin app

# ── Alpine（node:22-alpine、nginx:alpine…）──
# busybox 版：-D 跳过设密码，-G 指定主组，-h 指定 home
RUN addgroup -g "${APP_GID}" app \
 && adduser -u "${APP_UID}" -D -G app -h /app -s /sbin/nologin app
```

方言对照表（**左列参数不能出现在右列命令里，反之亦然**）：

| 用途 | Debian / Ubuntu | Alpine（busybox） |
|---|---|---|
| 建组 | `groupadd -g GID app` | `addgroup -g GID app` |
| 建用户 | `useradd -m -u UID -g app app` | `adduser -u UID -D -G app app` |
| 建 home | `-m` | `-h DIR` |
| 不设密码 | 默认不设 | 加 `-D` 跳过设密码；不加 `-S` 时若也不给 `-D`，会尝试交互式设密码 |
| nologin 路径 | `/usr/sbin/nologin` | `/sbin/nologin` |

**1.3.1 选 UID/GID —— "每次都卡在同一步"的头号原因。**

出现下面任一条，原因只有一个：**基镜像已经占了这个号**。

```
adduser: uid '1000' in use
useradd: UID 1000 is not unique
```

`node` 系列镜像内置 `node` 用户（**1000:1000**），`ubuntu` 镜像内置 `ubuntu`（**1000:1000**）。
而宿主的 `id -u` 通常也是 1000 —— 所以"把镜像用户对齐成宿主 UID"这个直觉，在 node 镜像上**必然失败**。

**按这个顺序决定，不要跳步：**

| 场景 | 怎么做 |
|---|---|
| 官方镜像（`node` / `nginx` / `postgres`…） | **直接用镜像自带的用户**：`USER node`。不建用户就不可能撞号 |
| 没有绑定挂载宿主目录 | **用 10001 这类高位号**，不要用 1000。命名卷不需要 UID 对齐 |
| 确实要绑定挂载，且需要 UID 对齐 | 先查占用，再决定（见下） |

**对齐前必须查占用：**

```bash
docker run --rm <你的基础镜像> sh -c 'grep -E ":(1000|10001):" /etc/passwd /etc/group'
```

（`grep -E` 在 busybox 和 Debian 上都有，可跨发行版使用。）

**如果宿主的 `id -u` 就是 1000，而镜像里已被占用，只有三条出路：**

1. **改用命名卷**，放弃绑定挂载——多数情况下这才是正解；
2. **换一个空闲 UID** 建用户，容器内以该 UID 运行（代价：不再与宿主一致）；
3. **换基础镜像**：`node:22-slim` 没有预置用户，`node:22-alpine` 有 `node`。

⚠️ **不要在撞号时改用 `useradd -o` 之类的强制选项**——那会造出两个同 UID 的用户，权限判定将不可预测。

**1.3.2 HOME 必须存在且可写。**
`-m` / `-h` 必须给：**用户没有 HOME 时，很多工具写 `~/.cache`、`~/.npm`、`~/.config` 会直接失败**，报错位置离根因很远。若不想建 home，就显式 `ENV HOME=/app XDG_CACHE_HOME=/tmp/.cache` 并保证该目录可写。

**1.3.3 UID/GID 是构建参数传进来的时候，Dockerfile 必须自己容忍撞号。**

1.3.1 的结论是"别撞号"，但那要求**你**能决定这个号。一旦号是别人传进来的，撞号就是必然而不是意外：

```bash
# CI / 部署流程常这么写：把宿主 UID 直接传进构建，然后在别人的服务器上炸
docker compose build --build-arg APP_UID=$(id -u) --build-arg APP_GID=$(id -g)
```

Linux 宿主的 `id -u` 就是 1000，而 `node` / `ubuntu` 镜像正占着 1000:1000 —— 于是在**开发机上永远构建成功**
（那里通常用默认的 10001），**只在用户的服务器上失败**，报错还停在 `[2/N]` 那一行。

这时正确做法不是"在文档里提醒别传 1000"（没人会读，而且值是自动传进来的），而是**让 Dockerfile 自己容忍**：

```dockerfile
ARG APP_UID=10001
ARG APP_GID=10001

RUN set -eux; \
    # 组：GID 空闲才建，否则复用现有组名（下面 adduser -G 需要名字）
    if ! grep -qE ":${APP_GID}:" /etc/group; then addgroup -g "${APP_GID}" app; fi; \
    # 用户：UID 空闲才建；已被占用（例如 node:1000）就跳过，直接复用那个身份
    if ! grep -qE ":${APP_UID}:" /etc/passwd; then \
      GROUP_NAME="$(awk -F: -v gid="${APP_GID}" '$3 == gid { print $1 }' /etc/group)"; \
      adduser -u "${APP_UID}" -G "${GROUP_NAME}" -h /home/app -s /sbin/nologin -D app; \
    fi; \
    mkdir -p /downloads /home/app/.cache; \
    chown "${APP_UID}:${APP_GID}" /downloads /home/app/.cache

COPY --chown=${APP_UID}:${APP_GID} . /app
USER ${APP_UID}:${APP_GID}
```

两个容易漏的细节：

- **全程用数字 ID**（`--chown=${APP_UID}:${APP_GID}`、`USER ${APP_UID}:${APP_GID}`）。复用已有用户时它的名字不叫 `app`，
  任何写死名字的地方都会失败；
- **用 `chown <uid>:<gid>`，不要用 `install -d -o <用户名>`**：后者要解析用户名，复用场景下直接报
  `install: unknown user`。（`grep`、`awk`、`mkdir`、`chown` 在 busybox 与 Debian 上都有，可跨发行版。）

这样"新建"与"复用"两条路径都成立，同一个 Dockerfile 在干净镜像和宿主对齐两种场景下都能构建。
**代价**：UID=0 也照样接受，所以如果确实要禁止 root，得自己加一道 `test "${APP_UID}" != "0"`。

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

**1.8 别把 build ARG 命名为 `UID` / `GID` / `HOME` / `PATH` —— 在 `RUN` 里会被 shell 的值盖掉。**

`RUN` 里的变量展开发生在 **shell** 里，不是 Docker 里。而 `UID`、`EUID`、`HOME`、`PATH` 这些
是 **shell 自己会设置的名字**（bash 里 `UID`/`EUID` 还是只读的）。于是：

```dockerfile
ARG UID=10001
RUN useradd -m -u "${UID}" app        # ❌ 若 shell 定义了 UID，这里展开成 -u 0
```

```
useradd: UID 0 is not unique
```

报错信息完全看不出根因——你明明写的是 10001。更麻烦的是它**依赖基础镜像的 shell**：
同样的 Dockerfile 换个基镜像就可能从"正常"变成"构建失败"。

```dockerfile
ARG APP_UID=10001
RUN useradd -m -u "${APP_UID}" app    # ✅ 换个不会撞名的名字
```

同理，任何"看起来像环境变量"的 ARG 名（`HOME`、`PATH`、`LANG`、`USER`）都可能被基镜像或 shell
的预设值盖掉。**判断标准：这个名字如果是 shell 或基础镜像会自己设的，就别用作 ARG 名。**
本文档早先的模板就写作 `ARG UID` / `ARG GID`（已全部改名），照抄旧版本会踩这个坑。

**附带一条**：需要校验属主时，**按用户名比较**，别用 `${UID}`：

```dockerfile
# ✅ 构建期自检：权限配错在构建时就炸，而不是等容器起来报 EACCES
RUN test -x /usr/local/bin/entrypoint.sh \
 && test "$(stat -c '%U:%G' /data)" = "app:app"
```

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

⚠️ 前提是**挂载点在镜像里已经存在且属主正确**。如果挂载的是镜像中不存在的路径，卷会是 `root:root`，属主不会自动对齐——先在 Dockerfile 里 `RUN mkdir -p /app/data && chown app:app /app/data`。

**解法 2：对齐 UID/GID。**

```bash
docker run --user "$(id -u):$(id -g)" app:verify
```

```yaml
services:
  app:
    # 必须与镜像内用户的 UID 一致。不要用 1000 作默认值 —— 见 1.3.1
    user: "${UID}:${GID}"
```

⚠️ 这里的 `${UID}` 是 **Compose 从环境变量插值**的，不是 shell 变量展开 ——
而 shell 的 `UID`/`GID` **默认不导出**，所以直接 `docker compose up` 多半取到空值
（插值成 `user: ":"`，容器起不来或行为怪异）。要么在 `.env` 里写死，要么显式传入：

```bash
UID=$(id -u) GID=$(id -g) docker compose up -d
```

前提是镜像里该 UID 对工作目录有读写权限。
⚠️ **`:-1000` 这种默认值本身就是陷阱**：宿主 `id -u` 通常是 1000，而 `node`、`ubuntu` 镜像
**已经占用了 1000**，代入即报 `adduser: uid '1000' in use`。要么显式设值，要么干脆别用绑定挂载。

**解法 3：构建期参数化 UID**，让镜像用户和宿主一致（模板里的 `ARG APP_UID` / `ARG APP_GID` 就是这个用途）。

> ⚠️ **只在真的需要绑定挂载时才这么做。** 这是本 skill 里最容易引发构建失败的一条建议：
> 宿主 `id -u` 通常是 1000，而 `node`、`ubuntu` 等镜像**已占用 1000**，直接代入必然报
> `adduser: uid '1000' in use`。
> **命名卷不需要 UID 对齐**——没有绑定挂载就别参数化，用 10001 这类高位号即可。决定前先读 1.3.1。

**3.1 compose 默认以 root 运行**，在宿主机上生成的文件会变成 root 属主，之后宿主上的普通用户就改不动了。要么显式 `user:`，要么明确接受这个后果。

**3.2 挂 docker socket 时组 ID 常不一致**，容器内 `docker` 组的 GID 与宿主不同 → 用 `group_add: ["${DOCKER_GID}"]`。

**3.3 把"一直以 root 跑"的容器改成非 root 时，已有的数据卷会挡住你。**

这是 3.1 的反向陷阱，也是最容易被低估的一次改动：命名卷只在**首次创建且为空**时继承镜像里的属主
（解法 1）。**已经存在的卷不会**——旧版本容器以 root 写过数据，卷里的文件就是 `root:root`。
于是你加上 `USER app`、重新部署，容器起来了却写不进数据库，报 `EACCES` / `unable to open database file`。
**老用户升级时必然踩到，而你自己在干净机器上测试永远复现不了。**

三种处理方式：

1. **入口脚本以 root 起步，chown 之后再降权**（推荐：升级对用户零操作）。
   需要镜像里有 `gosu`（Debian 仓库有，约 2 MB 的静态二进制，无依赖）：

   ```dockerfile
   ARG DEBIAN_FRONTEND=noninteractive
   RUN apt-get update \
    && apt-get install -y --no-install-recommends gosu \
    && rm -rf /var/lib/apt/lists/*
   COPY --chmod=755 docker/entrypoint.sh /usr/local/bin/entrypoint.sh
   ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
   CMD ["node", "dist/server.js"]
   ```

   ```sh
   #!/bin/sh
   set -eu
   if [ "$(id -u)" = "0" ]; then
       [ -d /data ] && { chown -R app:app /data 2>/dev/null \
           || echo "警告：/data 属主修正失败，将以 root 继续运行" >&2; }
       command -v gosu >/dev/null 2>&1 && exec gosu app "$@"
       echo "警告：未找到 gosu，无法降权，以 root 继续运行" >&2
   fi
   exec "$@"
   ```

   ⚠️ **关键设计：失败要退回旧行为（root），而不是让容器起不来。**
   一个"更严格"的入口脚本（`set -e` + 直接 `exec gosu`）在 gosu 缺失或 chown 失败时会让容器
   直接挂掉——用可用性换隔离度，对自托管部署是亏的。告警 + 继续跑，比一个起不来的容器好。
   注意入口脚本必须是 LF 且可执行（见 1.6、第 6 节）。

2. **文档里给一条一次性命令**，让用户自己修卷属主：

   ```bash
   docker run --rm -v appdata:/data alpine chown -R 10001:10001 /data
   ```

   代价：用户不照做就升级失败，而且报错信息（`EACCES`）不容易联想到这里。

3. **干脆不改**，继续以 root 跑，并在 README 里写明这是有意的取舍。
   （如果这个容器只跑自家代码、不处理不可信输入，这个选择是站得住的——别为了"看起来安全"引入上面那些复杂度。）

无论选哪种，**改完必须专门验证一次"老卷升级"路径**：先用旧镜像跑出一个有数据的卷，再切到新镜像启动，
确认仍能写入。只测"干净卷"等于没测（见第 7 节）。

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

**5.5 `.dockerignore` 的行尾必须是 LF**（Windows 宿主上很容易踩）。

Docker 是**逐行**解析 `.dockerignore` 的。在 `core.autocrlf=true` 的机器上，文件被检出成 CRLF 后，
每行末尾都带一个 `\r`，于是 `node_modules/` 实际变成了 `node_modules/\r` —— 规则**静默失配**：
不报错、不警告，只是不再排除，构建上下文重新变大、宿主的 `node_modules` 又混进镜像。

```gitattributes
.dockerignore text eol=lf
```

同理，`.env` 这类"值必须精确"的文件也建议钉成 LF：运维命令常用 `cut -d= -f2-` 取值，
行尾的 `\r` 会悄悄进到口令、密钥里（表现为"密码明明对却登录不上"）。

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

**UID/GID 来自构建参数时，必须专门用"会撞号的那个号"再构建一次。** 这条路径在开发机上永远走不到
（默认值是安全的高位号），只在用户服务器上炸，所以只能靠这里补：

```bash
# 1000 是 node/ubuntu 镜像自带的号，也正是宿主机上 id -u 的常见值
docker build --build-arg APP_UID=1000 --build-arg APP_GID=1000 -t app:uid1000 .
docker run --rm app:uid1000 id -u                                   # 期望：1000（不是 0，也不该构建失败）
docker run --rm app:uid1000 sh -c 'touch "$PWD/.w" && rm "$PWD/.w"'
docker run --rm -v appdata1000:/app/data app:uid1000 sh -c 'touch /app/data/.w && rm /app/data/.w'
```

宿主上 `id -u` 与 `id -g` 不相等时（如 `1000:1001`）**也要各测一次**——"UID 被占、GID 空闲"与
"两个都被占"走的是不同分支，只测一种会漏。

compose 项目还要：

```bash
docker compose config          # 校验语法与变量插值，不启动容器
docker compose up -d
docker compose logs --tail=50
docker compose down
```

**受限模式下这些命令会被命名管道挡住** → 回到第 0 节：申请一次完全权限把验证跑完，而不是把 Dockerfile 改得"看起来更安全"。

**环境里没有 docker 时（例如纯 Windows 主机未装 Docker Desktop）**：**明确说明"未经构建验证"**，
不要声称已验证通过。同时把**能离线验的先验掉**，并把"要你在有 docker 的机器上跑这几条"写清楚：

| 离线能验 | 怎么验 |
|---|---|
| 行尾 / 执行位相关文件 | 读字节数 CR 是否为 0（`.sh`、`Dockerfile`、`.dockerignore`、`.env.example`） |
| `.gitattributes` 是否覆盖到位 | `git check-attr text eol -- <文件>...`，确认目标文件都是 `eol: lf` |
| compose 层级是否正确 | 按缩进逐行核对：`extra_hosts`/`environment`/`volumes` 必须同级，列表项比键多两格 |
| 容器名/卷名是否会撞 | 通读 `container_name` 与顶层 `volumes:`，确认都带项目前缀 |
| 构建上下文是否混入本地产物 | 检查 `.dockerignore` 是否排掉 `node_modules`、`dist`、`wwwroot`、`*.db` |
| 镜像内路径的属主一致性 | 核对 `COPY --chown` 与 `RUN mkdir/chown` 的目标路径是否覆盖所有需要写的目录 |

这些**都不能替代构建验证**。交付时要给出可在别处执行的验证清单（照上面第 7 节那几条），
并说明哪几条你已经验过、哪几条没有——**含糊其辞比承认没验更糟**。

---

## 8. compose 编排：容器名与跨容器访问

这一节的两个问题都不在 Dockerfile 里，所以最容易改错地方。

**8.1 `Conflict. The container name "/X" is already in use` —— 先确认那是不是你的容器。**

固定 `container_name` 会**关掉 Compose 的项目名前缀**。后果：

- 同一台机器上从两个目录部署同一份 compose → 第二次必然撞名（报的是 `container_name` 那个名字）；
- 报错里的名字可能属于**完全无关的另一个项目**——它恰好也用了这个名字。

**别照着报错去改自己的 compose。** 先问 Docker 那个容器是谁的：

```bash
docker ps -a --filter name=^/redis$ --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
docker inspect redis --format '{{ index .Config.Labels "com.docker.compose.project" }} ← {{ index .Config.Labels "com.docker.compose.project.working_dir" }}'
docker compose config | grep -n container_name      # 你当前目录这份到底声明了什么名字
```

第二条输出里的 `working_dir` 直接告诉你它属于哪个目录的项目。**如果指向别的项目，就不是你的问题。**

**修法（非破坏性优先）**：

```bash
docker rename redis redis-old     # 改名即可解封：不删数据、不影响别的服务
```

⚠️ **不要顺手 `docker rm -f redis`**：如果它是另一个应用正在用的 Redis，删了会打断那个应用；
而且匿名卷会一起丢。要删也先 `docker rename`，确认没用之后再删。

**预防**：除非确实需要固定名字（例如别处已经写死了 `docker exec shortdrama ...`），
**不要设 `container_name`** —— 让 Compose 按 `<项目名>-<服务名>-1` 自动命名，天然不会跨项目冲突。

**8.2 容器里的 `127.0.0.1` 是容器自己，不是宿主机。**

把"本机服务"的地址写进配置时，这一点会让**源码运行正常、容器里永远连不上**：

```yaml
services:
  app:
    environment:
      # ❌ 容器里的 127.0.0.1 指容器自己，连不到宿主机上的服务
      SERVICE_URL: "http://127.0.0.1:8787"
      # ✅ 用 host.docker.internal
      SERVICE_URL: "http://host.docker.internal:8787"
    # Linux 需要这行才能解析 host.docker.internal；Docker Desktop（Win/Mac）原生支持
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

- **容器之间**通信用服务名（`http://redis:6379`），前提是同一个 compose 网络；
- 宿主上的服务也可以用宿主的**局域网 IP**，但那会随网络变化，不如 `host.docker.internal` 稳；
- 只监听 `127.0.0.1` 的宿主服务**能被容器访问到**吗？能——`host.docker.internal` 走的是宿主网关，
  目标仍是宿主上的 `127.0.0.1:端口`。所以宿主的服务不必改成 `0.0.0.0`（改成 `0.0.0.0` 反而会暴露到局域网）。

**顺带**：镜像里 `appsettings` / `.env.example` 的默认值若是 `127.0.0.1`，
一定要在注释里写明"Docker 部署必须改"，否则下一个部署的人会照抄默认值，然后对着
"连不上本机服务"的报错查半天。**验证方式**：

```bash
docker compose exec app curl -fsS http://host.docker.internal:8787/api/health
```

**8.3 面板/编排工具的「更新容器」会把动态 IP 固化成静态 IP。**

报错长这样：

```
Error response from daemon: invalid config for network <项目>_default: invalid endpoint settings:
user specified IP address is supported only when connecting to networks with user configured subnets
```

根因：宝塔 / 1Panel 这类面板的「更新容器」**不是在跑 compose**，而是把面板记录的容器参数
搬到新容器上。它从 `docker inspect` 读到当前分配的 IP，回填成 `ipv4_address`；
而 compose 自动创建的网络（`<项目>_default`）**没有用户配置的子网**（Docker 自动分配），
于是静态 IP 不合法 → 新容器创建失败（面板通常还会"恢复原容器"）。

**先确认是不是这个原因**：

```bash
# 网络有没有自定义子网？（输出为空/无 Subnet 就是没有）
docker network inspect <项目>_default --format '{{json .IPAM.Config}}'
# 容器有没有被请求静态 IP？（IPAMConfig 非 null 就是有）
docker inspect <容器名> --format '{{json .NetworkSettings.Networks}}'
```

**修法：让 compose 管这个容器，别用面板的「更新容器」。**

```bash
docker compose down && docker compose up -d
```

compose 知道期望状态，重建时会用动态 IP；面板则会把"当前状态"当"期望状态"再应用一遍 ——
所以**改了 compose 之后点面板更新，应用的仍是旧参数**（配置漂移），这是同一根源的另一面。

如果必须用面板管理，就在面板里把该容器的**静态 IP 留空**。
不推荐"给网络写死一个子网"来迁就它：写死的子网可能和宿主已有网段冲突，导致网络直接创建失败。

**通用教训**：编排工具的"更新/重建"按钮与 `docker compose` 是两套期望状态来源。
选一个用，别混用 —— 混用的症状就是"配置明明改了却不生效"或"重建就报网络错误"。

**8.4 「端口打不开」：先分清对外端口、绑定地址、容器内端口三件事。**

`ports` 的写法是 **`<对外>:<容器内>`**，写反了照样能启动，只是永远访问不到：

```yaml
ports:
  # ✅ 对外 18080 → 容器内 8080（容器内应用监听 8080）
  - "18080:8080"
  # ❌ 写反：对外 8080 → 容器内 18080，容器里没人在听 18080
  - "8080:18080"
  # ⚠️ 只绑本机：容器没问题，但**外网永远访问不到**（本机 curl 却是通的）
  - "127.0.0.1:18080:8080"
```

三种"打不开"要按**从内到外**的顺序排除，不要一上来就换端口：

```bash
docker compose ps                                  # ① 容器在跑吗
ss -lntp | grep 18080                              # ② 宿主在监听吗（看绑定地址！）
curl -fsS http://127.0.0.1:18080/health            # ③ 本机通吗
# ④ 外网仍不通 → 云安全组 + 宿主防火墙，两处都要放行
```

| 现象 | 根因 |
|---|---|
| ② 显示 `127.0.0.1:18080` 而不是 `0.0.0.0:18080` / `*:18080` | 映射里写了 `127.0.0.1:` 前缀，只对本机开放 |
| ③ 通、④ 不通 | 云厂商安全组或宿主防火墙没放行（**两处都要**，缺一不可） |
| ② 完全没有 LISTEN | 端口映射写反了，或容器内应用没起来 |

> 💡 **别用 8080 当对外默认端口**：面板、代理、各种测试服务都爱用它，撞车概率高。
> 换一个冷门端口（如 18080）成本极低；而**容器内端口保持 8080 即可** ——
> 容器内的端口是隔离的，不存在冲突，改它只会多改一堆地方（`EXPOSE`、`ASPNETCORE_URLS`、健康检查）。



---

## 9. 症状 → 根因速查

| 报错 / 现象 | 根因 | 修法 |
|---|---|---|
| `RUN` 步骤里 `Permission denied` | `USER` 切换过早 | `USER` 移到最后 |
| `adduser: Unknown option` / `addgroup: invalid option -- 'g'` | **发行版方言用错**：Alpine(busybox) 参数用在 Debian 上，或反之 | 按 `FROM` 选对应方言，见 1.3 |
| `groupadd: not found` / `useradd: not found` | 基镜像是 Alpine，没有 GNU 的 `useradd`/`groupadd` | 换 1.3 的 Alpine 版，或直接 `USER node` |
| `groupadd: Permission denied` | 前面已切 `USER`，当前不是 root | 把 `USER` 移到最后 |
| `adduser: uid '1000' is in use` | 基镜像已占用该 UID（`node`、`ubuntu` 镜像都用 1000:1000） | 换空闲 UID 或直接 `USER node`，见 1.3.1；**号是 `--build-arg` 传进来的**就改成容忍撞号，见 1.3.3 |
| `useradd: UID 0 is not unique`（明明传的是非 0 的号） | build ARG 名叫 `UID`，在 `RUN` 里被 shell 的只读 `UID`（root 下为 0）盖掉 | ARG 改名 `APP_UID`，见 1.8 |
| `did not complete successfully: exit code: 9`（建组那一步） | `groupadd` 的「组名已存在」：镜像**内置**了同名组（.NET 镜像内置 `app`，正是模板常用的名字） | 别自建用户，直接用内置的 `USER app`；或换个不撞名的名字。见「开工前必做」 |
| 构建在 `[2/N]` 那行失败，但**开发机上是好的** | UID/GID 来自构建参数，开发机用默认高位号、用户服务器传 `id -u`（=1000） | 见 1.3.3：探测占用、复用已有身份、全程用数字 ID |
| 容器启动即 `permission denied: ./entrypoint.sh` | 脚本缺执行位（Windows 宿主常见） | `COPY --chmod=755` 或 `RUN chmod +x` |
| `/bin/sh^M: bad interpreter` | CRLF 行尾 | `.gitattributes` 设 `*.sh text eol=lf` |
| 运行期 `EACCES: ... open '/app/...'` | COPY 未 `--chown`，或绑定挂载盖掉属主 | `COPY --chown` / 换命名卷 / 对齐 UID |
| 写 `~/.cache`、`$HOME` 失败 | 用户没有 HOME 或 HOME 不可写 | `useradd -m` 或 `ENV HOME=...` |
| `bind: permission denied` 监听 80 | 非 root 不能绑 <1024 | 监听 8080 + `-p 80:8080` |
| 卷内文件属主是 `root` | 挂载点在镜像中不存在 | 镜像里先 `mkdir -p` + `chown app:app` |
| 容器里出现宿主的 `node_modules` | `.dockerignore` 漏排 | 加 `node_modules`、`.git` |
| 改一行源码就重装依赖 | COPY 顺序错 | 先拷依赖清单再 `COPY . .` |
| 构建上下文几百 MB / 极慢 | 缺 `.dockerignore` | 补 `.dockerignore` |
| `exec format error` | 架构不匹配 | 补 `--platform` / `TARGETARCH` |
| `docker stop` 要等超时 | PID 1 是 shell | 入口脚本 `exec "$@"`、`--init` |
| `Conflict. The container name "/X" is already in use` | 固定 `container_name` 关掉了项目名前缀；或那个名字属于**别的项目** | 先 `docker inspect` 看它属于谁；`docker rename` 解封（别急着 `rm -f`）；见 8.1 |
| 容器里连不上"本机服务"（源码运行却正常） | 容器里的 `127.0.0.1` 指容器自己 | 改 `host.docker.internal` + `extra_hosts: host-gateway`；见 8.2 |
| 升级后突然写不进数据卷（`EACCES`、`unable to open database file`） | 从 root 切非 root，而**已有的卷**是 root 属主（只有新建的空卷才继承镜像属主） | 入口脚本 chown 后降权，或文档给一次性 chown；见 3.3 |
| `.dockerignore` 里写了规则却没生效 | 行尾是 CRLF，规则被解析成 `node_modules/\r` | `.gitattributes` 加 `.dockerignore text eol=lf`；见 5.5 |
| `user specified IP address is supported only when connecting to networks with user configured subnets` | 面板/编排工具的「更新容器」把动态 IP 回填成了静态 IP，而 compose 默认网络没有自定义子网 | 改用 `docker compose up -d` 重建；或在面板里把静态 IP 留空；见 8.3 |
| `open //./pipe/docker*: Access is denied` | DSH 受限模式（**非 Dockerfile 问题**） | 见第 0 节，申请完全权限 |

---

## 改 Dockerfile 之前先问自己

1. 这个报错是**构建期**还是**运行期**的？两套身份不一样。
2. 是**镜像内**的问题，还是**挂载**把属主盖掉了？
3. 是 Docker 的问题，还是**沙箱**不让 docker 说话？（第 0 节）
4. **这个报错真的在我的文件里吗？** 容器名冲突、端口占用这类报错，肇事者可能是**别的项目**
   （第 8.1 节）——先 `docker inspect` 查归属，再决定改不改自己的 compose。
5. 配置里的 `127.0.0.1` 是**从哪一侧**访问的？宿主上跑得通不代表容器里跑得通（第 8.2 节）。
6. 这次改动会不会**只在新环境成立**？改属主、改 UID、加 `USER` 时，想想**已有数据卷**的老用户（第 3.3 节）。
7. 我这次改动**验证过**了吗？没验证就不要说"修好了"——并说清哪些验了、哪些没验。
