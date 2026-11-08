# 📬 Outlook 邮件管理

<div align="center">

**基于 Cloudflare Workers 的轻量级 Outlook 邮件管理工具**

🆓 完全免费 · ☁️ 无需服务器 · 🌍 全球加速 · 🌗 深浅主题 · 🌐 中英双语

[![License: GPL-3.0](https://img.shields.io/badge/License-GPL%203.0-blue.svg)](./LICENSE)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.8+-3178C6?logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Cloudflare Workers](https://img.shields.io/badge/Cloudflare-Workers-F38020?logo=cloudflare&logoColor=white)](https://workers.cloudflare.com/)
[![Hono](https://img.shields.io/badge/Hono-4-E36002?logo=hono&logoColor=white)](https://hono.dev/)
[![D1](https://img.shields.io/badge/D1-SQLite-003B57?logo=sqlite&logoColor=white)](https://developers.cloudflare.com/d1/)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/roseforyou/cf-outlook-email/pulls)

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/keweiya/cf-outlook-email)

### 🚀 部署：4 步，全在 Cloudflare 网页端

> **本仓库不预置数据库。** 第一次部署一定会「失败 / 不可用」——因为还没有 D1。
> 数据库由你自己创建、初始化、绑定，第二次部署才正常工作。这是刻意设计的顺序，不是 bug。

| 步骤 | 操作 | 预期结果 |
|:--:|---|---|
| ① | fork 本仓库 → Cloudflare → **Workers & Pages → Create → Import a repository** 选你的仓库<br>（或直接点上面的 **Deploy to Cloudflare** 按钮，向导里 Worker 名填 `outlook-email`） | 构建成功、站点能打开，但登录/接口提示「数据库未就绪」——**正常** |
| ② | **Storage & Databases → D1 → Create**，名字 `outlook-email-db` → 进这个库 → **Console** → 粘贴下面的建表 SQL 执行 | 库里有 7 张表（settings / groups / accounts / temp_emails / tags / account_tags / push_state） |
| ③ | 你的 Worker → **Settings → Bindings → Add → D1 database**，Variable name 必须填 **`DB`**，选刚建的库<br>再到 **Settings → Variables and Secrets** 添加 `ADMIN_PASSWORD`（登录密码）、`COOKIE_SECRET`（随机 32+ 位） | 绑定列表出现 `DB`，两个 Secret 就位 |
| ④ | 保存绑定后**直接刷新页面**（控制台保存时会自动部署新版本，不需要再点 Retry） | 打开 `https://<worker名>.<你的子域>.workers.dev`，用密码登录成功 ✅ |

<details open>
<summary><b>② 的建表 SQL（整段复制，粘到 D1 的 Console 里执行）</b></summary>

```sql
-- ── 0001_init.sql ──
-- settings: login password hash, site config, GPTMail config, etc.
CREATE TABLE IF NOT EXISTS settings (
  key        TEXT PRIMARY KEY,
  value      TEXT NOT NULL,
  created_at TEXT DEFAULT CURRENT_TIMESTAMP,
  updated_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- groups: email account groups
CREATE TABLE IF NOT EXISTS groups (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  name        TEXT NOT NULL UNIQUE,
  description TEXT DEFAULT '',
  color       TEXT DEFAULT '#2563eb',
  created_at  TEXT DEFAULT CURRENT_TIMESTAMP,
  updated_at  TEXT DEFAULT CURRENT_TIMESTAMP
);
INSERT OR IGNORE INTO groups (id, name, description, color)
VALUES (1, '默认分组', '默认邮箱分组', '#2563eb');

-- accounts: Outlook email accounts
CREATE TABLE IF NOT EXISTS accounts (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  email         TEXT NOT NULL UNIQUE,
  client_id     TEXT NOT NULL,
  refresh_token TEXT NOT NULL,
  password      TEXT DEFAULT '',
  group_id      INTEGER DEFAULT 1,
  remark        TEXT DEFAULT '',
  status        TEXT DEFAULT 'active',
  created_at    TEXT DEFAULT CURRENT_TIMESTAMP,
  updated_at    TEXT DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (group_id) REFERENCES groups(id)
);

-- temp_emails: temporary email records
CREATE TABLE IF NOT EXISTS temp_emails (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  email      TEXT NOT NULL UNIQUE,
  source     TEXT DEFAULT '',
  remark     TEXT DEFAULT '',
  created_at TEXT DEFAULT CURRENT_TIMESTAMP,
  updated_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- ── 0002_tags.sql ──
-- tags: free-form labels for accounts (many-to-many, complements single-select groups)
CREATE TABLE IF NOT EXISTS tags (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  name       TEXT NOT NULL UNIQUE,
  color      TEXT DEFAULT '#6366f1',
  created_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- account_tags: join table
CREATE TABLE IF NOT EXISTS account_tags (
  account_id INTEGER NOT NULL,
  tag_id     INTEGER NOT NULL,
  PRIMARY KEY (account_id, tag_id),
  FOREIGN KEY (account_id) REFERENCES accounts(id) ON DELETE CASCADE,
  FOREIGN KEY (tag_id) REFERENCES tags(id) ON DELETE CASCADE
);
CREATE INDEX IF NOT EXISTS idx_account_tags_tag ON account_tags(tag_id);

-- ── 0003_push_state.sql ──
-- push_state: per-account watermark for Telegram new-email push.
-- last_pushed_at holds the receivedDateTime (ISO string) of the newest email
-- already pushed, so each cron run only forwards strictly newer messages.
CREATE TABLE IF NOT EXISTS push_state (
  account_id     INTEGER PRIMARY KEY,
  last_pushed_at TEXT DEFAULT '',
  updated_at     TEXT DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (account_id) REFERENCES accounts(id) ON DELETE CASCADE
);
```

执行完自检：

```sql
select name from sqlite_master where type='table' and name not like 'sqlite_%' order by name;
```

应返回 7 张表：`account_tags, accounts, groups, push_state, settings, tags, temp_emails`（`sqlite_sequence` 是内部表，已用 `name not like 'sqlite_%'` 过滤掉）。

> 如果控制台提示「一次只能执行一条语句」，就按分号把上面的语句拆开逐条执行。
> 这三段对应仓库里的 `migrations/0001_init.sql`、`0002_tags.sql`、`0003_push_state.sql`。

</details>

- 🔁 **任何人 fork 后都按同样 4 步走**：把按钮 URL 换成自己仓库即可 —— `https://deploy.workers.cloudflare.com/?url=<你的仓库URL>`。
- ⚙️ **所有变量和绑定都在网页端设置**，仓库里不含任何密钥；`wrangler.toml` 也故意不写 `[[d1_databases]]`，避免部署时因「库里没有这个 ID」而报 `7404`。
- ⚠️ **别混用两种模式**：控制台绑的 D1，会在下一次 `git push`/`Retry deployment`（wrangler 以仓库配置为准）时被覆盖。长期用 Git 部署就把 `[[d1_databases]]` 写进 `wrangler.toml`；只用控制台部署就不要再触发构建。
- 💡 **环境变量不能替代 D1 绑定**：`env.DB` 必须是绑定对象（有 `.prepare()`），传 `D1_DATABASE_ID=xxx` 字符串没用，代码会直接 `TypeError`。
- 🧰 想改用 wrangler 管绑定：把 `wrangler.toml` 里注释掉的 `[[d1_databases]]` 四行取消注释、填自己的 `database_id`，然后 `pnpm exec wrangler d1 migrations apply DB --remote && pnpm exec wrangler deploy`。
- 📖 每步细节与排错：👉 **[docs/DEPLOY-CLOUDFLARE.md](./docs/DEPLOY-CLOUDFLARE.md)** ｜ 纯 CLI 部署见 [docs/GUIDE.md](./docs/GUIDE.md)

🌐 [English](./README_EN.md) · 📖 [详细部署教程](./docs/GUIDE.md) · 🔌 [对外 API 文档](./docs/API.md)

</div>

---

| 🌙 深色模式 | ☀️ 浅色模式 |
|:---:|:---:|
| ![深色模式](./docs/preview.png) | ![浅色模式](./docs/preview-light.png) |

## ✨ 特性

- 🔐 **一键授权** — 浏览器弹窗登录微软账号，自动获取凭证，无需手动复制 token
- 🔄 **Token 自动续期** — 每次读邮件自动刷新 token，只要定期使用就不会过期
- 📦 **批量管理** — 批量导入/导出/删除/移组，支持单条与选中导出、分组和状态筛选
- 📨 **邮件阅读** — 通过 Microsoft Graph API 实时读取，支持收件箱/垃圾箱/已删除文件夹切换、聚合视图、分页加载、搜索和 HTML 渲染
- 📭 **临时邮箱** — 集成 GPTMail API，一键生成临时邮箱接收邮件
- 🎨 **精致主题** — 深色/浅色/跟随系统，毛玻璃质感 + 圆形扫掠切换 + 低频呼吸光晕
- 🌐 **中英双语** — 默认中文，顶栏一键切换 English，偏好本地记忆，后端消息同步翻译
- 🆓 **完全免费** — 运行在 Cloudflare 免费层，无需信用卡

## 🚀 快速部署

> 💡 完整步骤请看 [详细部署教程](./docs/GUIDE.md)

```bash
# 1. 克隆 & 安装
git clone https://github.com/roseforyou/cf-outlook-email.git
cd cf-outlook-email
pnpm install

# 2. 登录 Cloudflare
pnpm exec wrangler login

# 3. 创建数据库（把输出的 database_id 填入 wrangler.toml）
pnpm exec wrangler d1 create outlook-email-db
cp wrangler.toml.example wrangler.toml
# 编辑 wrangler.toml，替换 REPLACE_WITH_YOUR_DATABASE_ID

# 4. 配置密码
pnpm exec wrangler secret put ADMIN_PASSWORD
pnpm exec wrangler secret put COOKIE_SECRET

# 5. 初始化 & 部署
pnpm exec wrangler d1 migrations apply outlook-email-db --remote
pnpm exec wrangler deploy
```

部署完成后访问输出的 URL，用设置的密码登录即可。🎉

## 📮 添加邮箱

登录后点击 **添加账号** → **一键授权** → 弹出微软登录窗口 → 授权后自动填入凭证 → 保存。

支持所有 Outlook / Hotmail / Live 邮箱，也支持批量导入（格式：`邮箱----密码----client_id----refresh_token`）。

## 🧱 技术栈

| 层 | 技术 |
|---|---|
| ⚙️ 运行时 | Cloudflare Workers (TypeScript) |
| 🧭 路由 | Hono |
| 🗄️ 数据库 | Cloudflare D1 (SQLite) |
| 🎨 前端 | 原生 HTML/CSS/JS |
| 📧 邮件 | Microsoft Graph API |
| 🚀 部署 | Wrangler |

## 🗂️ 项目结构

```
src/                     后端源码（Worker）
├── index.ts             入口 + 路由
├── auth.ts              HMAC-SHA256 Cookie 鉴权
├── graph.ts             Graph API 集成
├── routes/              业务路由（6 个模块）
└── utils/               加密、校验工具
public/                  前端（静态 SPA）
migrations/              D1 数据库建表
tools/                   辅助脚本
```

## 💰 免费额度

| 资源 | 免费额度 | 够用？ |
|------|----------|:------:|
| ⚡ Workers 请求 | 10 万/天 | ✅ |
| ⏱️ CPU 时间 | 10ms/请求 | ✅ |
| 🌐 外部请求 | 50/次 | ✅ (单账号单请求) |
| 💾 D1 存储 | 5 GB | ✅ |

## 🗺️ 路线图

**核心功能（已实现）**

- [x] 🔐 一键 OAuth 授权 & Token 自动续期
- [x] 👤 邮箱账号管理（增 / 删 / 改 / 查、测试连接）
- [x] 🗂️ 分组管理（自定义颜色、按分组与状态筛选）
- [x] 📦 批量导入 / 导出 / 删除 / 移组
- [x] 📤 单条 / 选中导出
- [x] 📨 邮件阅读（实时收件、搜索、HTML 渲染）
- [x] 📁 文件夹切换（收件箱 / 垃圾箱 / 已删除）
- [x] 🔀 聚合视图（收件箱 + 垃圾箱合并按时间排序，找验证码神器）
- [x] 📄 分页加载（加载更多）
- [x] 📭 临时邮箱（集成 GPTMail）
- [x] 🎨 主题切换 + 圆形扫掠过渡 + 呼吸光晕
- [x] 🔑 对外 API + API Key（免登录拉取邮件，自动化取验证码，见 [API 文档](./docs/API.md)）
- [x] 🗑️ 删除邮件（单条 / 批量，软删除到「已删除」）
- [x] 📎 附件下载
- [x] 🏷️ 标签系统（一个账号多标签，跨分组筛选）
- [x] ⏰ 定时刷新 Token（Cron Trigger，可配间隔/批量，自动保活账号）
- [x] 🤖 Telegram 推送新邮件（Cron 轮询，新邮件实时推送到 Telegram，可配间隔）
- [x] 🧭 界面打磨（页面级工具栏、仪表盘健康度卡片、设置页响应式网格、账号表分页、可搜索账号下拉）
- [x] 🌐 界面国际化（默认中文，一键切换 English）

**计划中（欢迎 PR）**

- [ ] 🔔 更多推送渠道（企业微信 / 钉钉等）

> ⚠️ 受 Cloudflare Workers 平台限制，以下功能无法实现：IMAP（Gmail / QQ / 163 等非微软邮箱）、SMTP 转发、HTTP/SOCKS5 代理。

## ⚠️ 免责声明

本项目仅供个人学习和管理自己的邮箱使用。请确保你对所管理的邮箱账号拥有合法授权，不得用于未授权访问他人邮箱或其他违法用途。默认 Client ID 为 Mozilla Thunderbird 公开 ID，仅供快速体验，正式使用建议[注册自己的 Azure 应用](./docs/GUIDE.md#自己注册-azure-应用)。使用者应自行承担因不当使用产生的一切法律责任，作者不承担任何责任。

## 🙏 致谢

本项目基于 [xiaozhi349/outlookEmail](https://github.com/xiaozhi349/outlookEmail) 改造而来。原项目为 Python Flask + SQLite 实现，本项目将其迁移至 Cloudflare Workers + D1，并重写了前后端代码。感谢原作者的工作。

## 友情链接

[LINUX DO](https://linux.do/) —— 新的理想型社区，技术爱好者的聚集地。

## 📜 许可证

[![License: GPL-3.0](https://img.shields.io/badge/License-GPL%203.0-blue.svg)](./LICENSE)

基于 **GPL-3.0** 协议开源。你可以自由使用、修改和分发本项目，但任何分发的衍生作品也必须以 GPL-3.0 协议开源并提供完整源代码。
