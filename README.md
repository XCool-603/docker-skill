# docker-skill

> 让 AI 写 Docker 时**权限一次到位**、**打包不再反复改**的规范技能。
> 兼容 DeepSeek Harness / Claude Skills 的 `SKILL.md` 目录包格式。

两条核心约定：

1. 镜像里始终有**两套身份**——构建期默认 `root`，运行期是你声明的非 root 用户。权限问题几乎都来自这两者没交接清楚。
2. 写完 Dockerfile**不算完成**。必须真的 `build` 并 `run` 过一次才算，否则就是拿使用者当编译器。

---

## 解决什么问题

用 AI 写 Docker，最常见的失败不是"语法写错"，而是**改了很多轮还在同一个坑里**——因为每一轮都在猜，而不是在定位。

```dockerfile
# ❌ 问题 1：USER 切早了，后面的步骤全部 EACCES
USER app
RUN apt-get update && apt-get install -y curl    # Permission denied

# ❌ 问题 2：COPY 出来的文件属主是 root，运行期用户写不进去
COPY . /app
USER app                                          # /app 归 root → 运行时报 EACCES

# ❌ 问题 3：Windows 宿主丢了执行位，容器一启动就失败
COPY entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]                     # permission denied
```

还有一类更隐蔽：**Dockerfile 根本没写错，是挂载把属主盖掉了**。

```bash
docker run -v "$PWD:/app" app
# 挂载后 /app 属主变成宿主 UID，镜像里 COPY --chown 的效果全部失效
# → 此时继续改 Dockerfile 是纯白改
```

以及一类最容易被误判的：**受限沙箱里 docker 根本连不上 daemon**，报错却是 `Access is denied`，看起来像权限没配好。

```bash
docker build .
# error during connect: open //./pipe/dockerDesktopLinuxEngine: Access is denied
# → 这是沙箱边界，不是 Dockerfile 的问题
```

这几类的共同后果：**人一直在改同一个文件，但问题根本不在那个文件里。**

---

## 这个技能提供什么

`SKILL.md` 是一份可直接被 AI 加载的规范，包含：

| 章节 | 拦掉的反复 bug |
|---|---|
| 最小正确模板 | 一份已规避主要坑的 Dockerfile 骨架，直接抄 |
| 0. 先判断是 Docker 还是沙箱 | 受限模式打不开命名管道，改 Dockerfile 无用 |
| 1. 构建期权限 | `USER` 时机、`COPY --chown`、Alpine/Debian 建用户差异、执行位 |
| 2. 运行期权限 | `exec "$@"`、PID 1、只读根文件系统 |
| 3. 挂载与卷属主 | 绑定挂载盖掉属主、命名卷继承规则、UID 对齐 |
| 4. 多阶段构建与打包 | `COPY --from` 路径、`TARGETARCH`、distroless、密钥处理 |
| 5. `.dockerignore` 与层缓存 | 上下文过大、依赖重装、缓存排查 |
| 6. Windows 宿主特有 | CRLF、执行位、WSL 挂载权限 |
| 7. 交付前必须验证 | 构建 + 冒烟测试命令，强制验证而非声称 |
| 8. 症状 → 根因速查 | 13 条常见报错直查根因 |

---

## 安装

**全局（推荐）**——clone 到任意位置，再挂进 DSH 的 skills 目录：

```powershell
git clone https://github.com/XCool-603/docker-skill.git C:\Work\docker-skill
New-Item -ItemType Junction -Path "$env:DSH_HOME\skills\docker" -Target "C:\Work\docker-skill\skills\docker"
```

用 junction 的好处：在 `C:\Work` 里改完 `git push`，本地已装的 skill **立即同步**，不需要两处维护。

**项目级**——直接把 `skills/docker` 复制到项目的 `.dsh/skills/` 或 `.agents/skills/` 下即可。

**Claude Code / 其他兼容 SKILL.md 的工具**——复制 `skills/docker` 到对应的 skills 目录。

---

## 目录结构

```
docker-skill/
├── README.md
├── LICENSE
└── skills/
    └── docker/
        └── SKILL.md
```

---

## 适用场景

- 新写 Dockerfile / docker-compose
- 审查已有 Docker 配置
- `docker build` / `docker run` / `docker compose` 报权限、属主、打包、缓存类错误
- 在 DSH 沙箱里跑 docker 命令被拒

---

## License

[MIT](LICENSE) © 2026 XCool
