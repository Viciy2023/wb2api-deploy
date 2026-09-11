# wb2api-deploy

`workbuddy2api` 的自动构建部署仓库（构建调度器，不保存上游源码）。

## 工作流程

```
上游 Sliverkiss/workbuddy2api 更新
        ↓
每天 03:00 定时任务：GitHub API 对比上游最新 commit SHA
        ↓ 有变化
clone 上游源码 → 构建镜像 → 推送 ghcr.io/viciy2023/wb2api-deploy:latest
        ↓
FNOS 上 Watchtower（每 6 小时）检测到新镜像 → 自动拉取更新
```

## 特性

- **变更检测**：用 commit SHA 对比，上游无变化则跳过构建
- **双 tag**：`latest`（自动更新用）+ `<sha>`（回滚用）
- **配置隔离**：`config.json`、`auths/`、`data/` 均在 FNOS 本地 volume，更新镜像不丢失

## 镜像

```
ghcr.io/viciy2023/wb2api-deploy:latest
ghcr.io/viciy2023/wb2api-deploy:<commit-sha>
```

## 手动触发构建

Actions 页面 → Sync & Build workbuddy2api → Run workflow
