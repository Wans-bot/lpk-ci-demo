# CI 环境探针 · LPK CI 出包验证

用 GitHub Actions 构建懒猫微服 LPK，然后把产物装到真机上验收。

## 结论先说

`lzc-cli project release` **不需要登录、不需要 docker、不需要连盒子**，所以 GitHub Actions（`ubuntu-latest`）能直接出包。本仓库就是这条链路的活证据。

## 一条命令构建

```bash
npm install -g @lazycatcloud/lzc-cli
lzc-cli project release          # 产物在 dist-lpk/
```

`buildscript` 写在 `lzc-build.yml` 里，所以这一条命令自己会跑 `npm install && npm run build`。
CI 与本地、任何智能体都是同一入口，不需要额外知道我项目怎么构建。

## 只有带镜像的应用才需要额外条件

| 应用类型 | 构建需要 |
|---|---|
| 纯静态 / 前端构建（本仓库） | 只要 node + lzc-cli |
| 镜像类，`builder: local` | 本机 docker + buildx（`ubuntu-latest` 自带） |
| 镜像类，`builder: remote`（默认） | 盒子在线 + 登录态，构建在盒子侧进行 |

本仓库属于第一类，所以 CI 里连 docker 都不用。

## workflow 做了什么

`.github/workflows/build-lpk.yml`：

1. checkout → setup-node 20
2. `npm install -g @lazycatcloud/lzc-cli`
3. `lzc-cli project release`
4. `lzc-cli lpk info` + `sha256sum` 打印可核对的指纹
5. `upload-artifact` 保存 `.lpk`（保留 30 天）
6. 打 `v*` tag 时额外把 `.lpk` 挂到 Release，盒子侧可直接下载

## 实机验收

```bash
# 在盒子上
lzc-cli install cloud.lazycat.app.lpkci-v1.0.0.lpk
```

验收点：

- 页面能打开，访问域名是盒子分配的 `lpkci` 子域
- 运行环境各项有值（不是一堆 `-`）
- 便签写入后刷新页面仍在（验证 localStorage 持久化）
- 卸载后部署确实移除

## 为什么 package.yml 里 en-US 是必须的

本地 `lzc-cli project release` 对缺 `en` 只给警告，**但开发者中心提审会直接拒绝**
（`manifest locales doesn't have en`）。这个坑在知识库 `04-平台坑总表` 有记录，
所以这里一开始就写全双语。
