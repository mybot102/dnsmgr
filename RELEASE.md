# 版本发布流程说明

本文档描述了本项目（彩虹聚合DNS管理系统）当前的版本发布流程。

---

## 版本号体系

项目同时维护两套版本号，它们相互独立但协同工作：

### 1. Git 标签版本（用户可见版本）

格式为 `主版本.次版本[.修订号]`，例如 `2.16`、`2.14.0`、`2.15`。

- 此版本号通过 Git **轻量标签**（lightweight tag）在 GitHub 上打标记。
- 对应 GitHub Releases 页面的发布记录，用户从该页面下载安装包。

### 2. 内部数字版本（数据库迁移版本）

格式为纯整数，例如 `1048`，存储在三个地方：

| 位置 | 作用 |
|------|------|
| `config/app.php` → `version` | 当前程序版本，每次发版时必须递增，用于向更新检查服务上报 |
| `config/app.php` → `dbversion` | 本次发版所需的最低数据库版本，**只有当包含数据库结构变更时才与 `version` 保持一致**；否则保留上一个有 DB 变更的版本号 |
| `app/sql/install.sql` → `INSERT INTO dnsmgr_config VALUES ('version', ...)` | 全新安装时写入数据库的初始版本号，始终与 `dbversion` 保持一致 |

> **示例（v2.10.0 → v2.11.0，无 DB 变更）：**
> - `version` 从 `1041` 升为 `1042`
> - `dbversion` 维持 `1040` 不变（本次无数据库结构改动）
>
> **示例（v2.15 → v2.16，有 DB 变更）：**
> - `version` 从 `1047` 升为 `1048`
> - `dbversion` 从 `1045` 升为 `1048`（本次包含新建 `domain_alias` 表等 DB 变更）
> - `install.sql` 中的版本号同步更新为 `1048`

---

## 数据库升级机制

用户升级时**无需手动执行 SQL**，系统在登录首页时会自动完成迁移：

**触发条件**：`config/app.php` 中的 `dbversion` 与数据库中存储的 `version` 值不一致。

**执行逻辑**（见 `app/controller/Index.php → db_update()`）：

1. 读取 `app/sql/update.sql` 的全部内容。
2. 按 `;` 分割成独立语句，逐条执行（忽略已存在的字段/表带来的错误）。
3. 将数据库中的 `version` 更新为 `dbversion` 的值。
4. 清空 ThinkPHP 配置缓存（`Cache::clear()`）。

> **因此，`update.sql` 是一个累积文件**，包含从第一版到当前版本的全部数据库升级 SQL（使用 `CREATE TABLE IF NOT EXISTS`、`ALTER TABLE ... ADD COLUMN IF NOT EXISTS` 等幂等语句），而 `install.sql` 是一个全量文件，用于全新安装时直接建库。

---

## 发布流程（全手动，无 CI/CD 自动化）

### 第一步：功能开发与合并

- 功能开发和 Bug 修复通过 Pull Request 或直接 push 到 `main` 分支完成。
- 无固定的发版周期，通常积累若干功能/修复后集中发布。

### 第二步：更新版本号（"version" 提交）

在 `main` 分支上提交一个专门的版本号更新提交，通常提交信息为 `version` 或 `update version`：

```
修改 config/app.php：
  - version: 1047 → 1048
  - dbversion: 1045 → 1048   （有 DB 变更时才改，否则保持旧值）

修改 app/sql/install.sql：
  - 将版本号 INSERT 语句中的数字更新为新的 dbversion 值

修改 app/sql/update.sql（如有 DB 变更）：
  - 在文件末尾追加本次新增的建表/加字段 SQL 语句
```

### 第三步：打 Git 轻量标签

```bash
git tag 2.16
git push origin 2.16
```

> 当前所有标签均为**轻量标签**（lightweight tag），直接指向某个 commit，而非带签名信息的附注标签（annotated tag）。

### 第四步：打包安装包

维护者本地执行以下步骤，生成包含依赖的安装包：

```bash
# 安装生产依赖（不含 require-dev）
composer install --no-dev --optimize-autoloader

# 打包（排除 .git、.env、runtime 等目录）
zip -r dnsmgr_2.16.zip . \
  --exclude "*.git*" \
  --exclude ".env" \
  --exclude "runtime/*" \
  --exclude ".vscode/*" \
  --exclude ".idea/*"
```

生成的 zip 包（约 3-4 MB）**已内含 `vendor/` 目录**，用户下载后无需执行 `composer install`。

### 第五步：在 GitHub 创建 Release

在 [GitHub Releases 页面](https://github.com/netcccyun/dnsmgr/releases/new) 手动创建发布：

- **Tag**：选择刚刚推送的标签（如 `2.16`）
- **Title**：`v2.16`
- **Body**：手写中文 Changelog，格式如下：
  ```
  1、新增 xxx 功能
  2、修复 xxx 问题
  3、优化 xxx 体验

  **Full Changelog**: https://github.com/netcccyun/dnsmgr/compare/2.15...2.16
  ```
- **Assets**：上传本地打好的 `dnsmgr_2.16.zip`

### 第六步：更新 Docker 镜像

Docker Hub 上的 `netcccyun/dnsmgr` 镜像由维护者单独维护（Dockerfile 不在本仓库中），发版后需同步更新镜像：

```bash
docker build -t netcccyun/dnsmgr:2.16 -t netcccyun/dnsmgr:latest .
docker push netcccyun/dnsmgr:2.16
docker push netcccyun/dnsmgr:latest
```

同时华为云镜像仓库 `swr.cn-east-3.myhuaweicloud.com/netcccyun/dnsmgr` 也会同步更新。

---

## 版本检查机制

系统首页会向以下 URL 发起请求来检查是否有新版本：

```
https://auth.cccyun.cc/app/dnsmgr.php?ver={当前version值}
```

> 源码中写作 `//auth.cccyun.cc/...`（协议相对 URL），实际由浏览器根据当前页面协议补全为 `https://` 或 `http://`。

（见 `app/controller/Index.php` 第 70 行）

此接口由维护者的外部服务维护，不在本仓库中。

---

## 各文件职责速查

| 文件 | 每次发版必须修改？ | 说明 |
|------|-------------------|------|
| `config/app.php` | ✅ 必须 | `version` 每次必须递增；`dbversion` 仅在有 DB 变更时才更新 |
| `app/sql/install.sql` | 条件必须 | 有 DB 变更时，需同步新建表/加字段，并更新 `version` 初始值 |
| `app/sql/update.sql` | 条件必须 | 有 DB 变更时，在文件末尾追加幂等的升级 SQL |

---

## 用户升级方式

> 见 README.md「自部署」章节：

1. 从 [Releases](https://github.com/netcccyun/dnsmgr/releases) 页面下载最新版本的 `dnsmgr_X.X.zip`。
2. 解压后将所有文件**上传覆盖**至服务器对应目录（无需重新安装）。
3. 访问网站首页，系统会自动触发数据库升级。

---

## 当前版本信息

| 项目 | 当前值 |
|------|--------|
| Git 标签版本 | `2.16` |
| 内部 `version` | `1048` |
| 内部 `dbversion` | `1048` |
