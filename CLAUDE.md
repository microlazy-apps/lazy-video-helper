# lazy-video-helper — 维护笔记

懒猫微服 wrapper for [LDJ-creat/video-helper](https://github.com/LDJ-creat/video-helper)。

## Lazycat appstore identifiers

- **package id**: `cloud.lazycat.app.video-helper`
- **app_id**: `5340` (recorded 2026-05-08)
- **subdomain**: `video-helper` → `https://video-helper.<box-domain>`
- **bootstrap workflow**: when re-running `bootstrap-app.yml` to
  resubmit a fix, pass `app_id=5340` so the workflow skips
  `/app/create` (which would 500 on duplicate package).

## 仓库结构

```
lazy-video-helper/
├── vendor/video-helper/                  上游 git subtree（保持原样）
├── patches/
│   └── 01-lazycat-dockerfile.patch       新增 Dockerfile / supervisord / entrypoint
├── lazycat/                              懒猫元数据
│   ├── package.template.yml
│   ├── lzc-manifest.template.yml
│   ├── lzc-build.yml
│   ├── lzc-deploy-params.yml
│   ├── appstore.yml
│   ├── icon.png
│   └── screenshots/
└── .github/workflows/                    薄壳，调用 microlazy-apps/lazycat-ci
    ├── release.yml
    └── bootstrap-app.yml
```

## 架构

**单容器，supervisord 管理两个进程**：

- `core` — FastAPI（Python 3.12 + uv），监听 `127.0.0.1:8000`
- `web`  — Next.js 16 standalone，监听 `0.0.0.0:3000`

入口仅暴露 `:3000`。Next.js 自带 rewrites 把 `/api/v1/*` 转发到 `127.0.0.1:8000`，所以**不需要 nginx**。

容器内 PATH：

- `/app/data` — 持久化（SQLite、视频缓存、whisper 模型）
- `/app/services/core/.venv` — uv build 出的 Python venv
- `/app/web` — Next.js standalone bundle (`apps/web/server.js` 是入口)

## Patch 流程

vendor 上游保持纯净，所有改动通过 `patches/01-lazycat-dockerfile.patch`，CI 会在 build 前 apply。

新增/修改 patch：

```sh
# 1. 把 patch apply 到 vendor 做实时编辑
git apply patches/01-lazycat-dockerfile.patch -p1 --directory=vendor/video-helper

# 2. 修改 vendor/video-helper/* 里的文件，新文件需要 git add -N
git add -N vendor/video-helper/<new-files>

# 3. 重生成 patch
git diff --no-color --relative=vendor/video-helper \
  vendor/video-helper/ > patches/01-lazycat-dockerfile.patch

# 4. 还原 vendor
git checkout HEAD -- vendor/video-helper/<modified-files>
rm vendor/video-helper/<new-files>

# 5. 自检
git apply --check patches/01-lazycat-dockerfile.patch -p1 --directory=vendor/video-helper
```

## 健康检查

`http://main:3000/api/v1/health` —— FastAPI 通过 Next.js rewrite 暴露。

容器内 healthcheck 用 `curl -fsS http://localhost:3000/api/v1/health`。

## 已知问题 / 待办

- **Whisper 模型首次启动下载** —— 几百 MB，可能需要几分钟。已通过 entrypoint 把 `HF_HOME / TORCH_HOME / XDG_CACHE_HOME` 指向持久化目录，模型只下载一次。
- **CPU 推理慢** —— 懒猫微服当前不暴露 GPU，转录长视频慢。建议用 `tiny` / `base` 模型。
- **monorepo workspace** —— pnpm workspace 包含 `apps/desktop`，build web 时需要 `apps/desktop/package.json` 才能解析 workspace。Dockerfile 已 COPY。

## 发布

**第一次** 注册 app（package id 不存在）：

```
gh workflow run bootstrap-app.yml -F version=0.0.1
```

成功后再走标准 release：

```
git tag v0.0.2 && git push origin v0.0.2
```

**重试 bootstrap**（pending review 阻塞）：先在懒猫开发者后台删除 app，把 version 升到下一个 tag，再 dispatch。

## 上游同步

```sh
git subtree pull --prefix=vendor/video-helper \
  https://github.com/LDJ-creat/video-helper.git main --squash

# 检查 patch 还能 apply
git apply --check patches/01-lazycat-dockerfile.patch -p1 --directory=vendor/video-helper
```

如果 patch 失效（上游改了 next.config / Dockerfile / pyproject）：apply 后手动改，重生成 patch。
