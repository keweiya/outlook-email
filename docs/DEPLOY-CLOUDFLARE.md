# 部署到 Cloudflare（自己建库、自己绑定）

本仓库**不预置 D1 数据库**，也**不在 `wrangler.toml` 里声明 D1 绑定**。原因很简单：
数据库 ID 只存在于你自己的 Cloudflare 账号里，写进仓库只会导致
`could not be found [code: 7404]`（别人的账号里没有这个 ID）。

所以部署是「两段式」的：**第一次部署先把 Worker 跑起来（此时不可用）→ 建库、初始化、绑定 → 第二次部署可用**。
这不是配置漏了，是刻意的顺序。

---

## 一、第一次部署：预期「失败」

### 方式 A：Deploy to Cloudflare 按钮（推荐）

```
https://deploy.workers.cloudflare.com/?url=https://github.com/keweiya/cf-outlook-email
```

点开后：授权 GitHub（会把仓库复制一份到你账号）→ 授权 Cloudflare → Worker name 填 `outlook-email`
→ Secrets 里填 `ADMIN_PASSWORD`、`COOKIE_SECRET`（`openssl rand -hex 32` 或乱敲 32+ 位）。

### 方式 B：Git 连接（fork 后在控制台接仓库）

1. GitHub 上 fork 本仓库
2. Cloudflare → **Workers & Pages → Create → Import a repository** → 选你的 fork
3. Build command：`npm run build`　Deploy command：`npx wrangler deploy`
4. 部署完到 **Settings → Variables and Secrets** 加 `ADMIN_PASSWORD`、`COOKIE_SECRET`

### 这次部署之后的状态

**构建会成功，站点能打开，但用不了**：

- 登录接口返回 500，页面提示「数据库未就绪」或「服务器内部错误」
- 打开 `https://<worker名>.<你的子域>.workers.dev` 时最多只能看到登录页

这就是「第一次部署一定失败」的含义：**不是构建失败，是运行时缺数据库**。
（之所以不让构建阶段就失败，是因为构建阶段根本无从知道你的数据库 ID。）

---

## 二、创建并初始化数据库

1. Cloudflare → **Storage & Databases → D1 → Create**
2. 名字填 `outlook-email-db`（也可以叫别的，但绑定时的对象要选对）→ 创建
3. 进这个库 → 顶部 **Console**（Query 编辑器），把下面这段**整段**粘进去执行
   （等同于依次执行 `migrations/0001_init.sql`、`0002_tags.sql`、`0003_push_state.sql`）：

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

   > 如果编辑器提示「一次只能执行一条语句」，就按分号把上面的语句拆开逐条执行。

4. 自检：

   ```sql
   select name from sqlite_master where type='table' and name not like 'sqlite_%' order by name;
   ```

   应返回 7 张表：`account_tags, accounts, groups, push_state, settings, tags, temp_emails`
   （`sqlite_sequence` 是 AUTOINCREMENT 的内部表，已被过滤）

---

## 三、绑定数据库 + 设置密钥

### 绑定 D1（关键：变量名必须是 `DB`）

1. 你的 Worker → **Settings → Bindings → Add → D1 database**
2. **Variable name**：`DB`（大小写敏感，写错就是 `env.DB` 为 undefined → 接口 500）
3. **D1 database**：选第二步建的那个库
4. 保存

### 设置密钥

**Settings → Variables and Secrets → Add**，类型选 **Secret**：

| 名称 | 必填 | 值 |
|---|:--:|---|
| `ADMIN_PASSWORD` | ✅ | 面板登录密码，自己定 |
| `COOKIE_SECRET` | ✅ | 随机 32+ 位（`openssl rand -hex 32`） |
| `GPTMAIL_API_KEY` | ⬜ | 临时邮箱功能用，不用就跳过 |

> 密钥改动**下次请求即生效**，不需要重新部署；绑定保存时控制台会自动创建并部署一个新版本，
> 所以**不需要**再去点 `Retry deployment`。

---

## 四、让改动生效 → 可用

保存绑定的那一刻，控制台已经用「新配置」部署了一个版本，直接刷新页面即可。

**只有一种情况需要再部署**：你之后又用 Git 推送/`Retry deployment` 触发了一次**构建部署**。
这时 `wrangler` 会以仓库里的 `wrangler.toml` 为准 —— 仓库里没有 D1 绑定，控制台加的绑定就**可能被覆盖掉**，
于是又变回「数据库未就绪」。两种模式选一种，别混用：

| 模式 | 绑定写在哪 | 谁来部署 | 需要 database_id 吗 |
|---|---|---|---|
| **A 控制台模式**（本文档） | Cloudflare 控制台 Settings → Bindings | 控制台（保存即部署） | ❌ 不需要 |
| **B 仓库模式** | `wrangler.toml` 的 `[[d1_databases]]` | Git 推送 / `Retry deployment` / 本地 wrangler | ✅ 需要（填自己的 ID） |

如果你会长期用 Git 推送部署，就用模式 B：把 `wrangler.toml` 里注释掉的 `[[d1_databases]]` 取消注释、填上 ID。

打开 `https://<worker名>.<你的子域>.workers.dev`：

1. 未登录会自动跳 `/login.html`
2. 用 `ADMIN_PASSWORD` 登录 → 进仪表盘
3. 添加账号 → 「方式二：手动授权」（默认 Thunderbird Client ID + `https://localhost` 回调，免注册 Azure）→ 能读邮件
4. 系统设置里改点东西并保存（验证 D1 写权限）

---

## 五、为什么不做成「自动建库」

自动建库有两种写法，都会带来更糟的体验：

| 做法 | 问题 |
|---|---|
| 在 `wrangler.toml` 里写死 `database_id` | 那是某个账号的私有 ID，别人部署必然 `7404` |
| 部署时脚本调 API 建库（本次尝试过） | 需要额外的账号权限探测；多账号时容易建到别的账号里；出问题时用户完全看不懂 |

所以这里回到最直白的方式：**库你自己建、表你自己初始化、绑定你自己在网页上点**。
每一步都能在控制台看到结果，出错也知道错在哪。

---

## 六、能不能用「环境变量」传 D1 的库 ID？

**不能。** 环境变量/Secret 只能传字符串，而 `env.DB` 必须是 **D1 绑定对象**，代码里要调用它的方法：

```ts
await env.DB.prepare('select * from settings').all();   // env.DB 是对象，不是字符串
```

如果只用 `[vars]` 或 Secret 传一个 `D1_DATABASE_ID=xxx`，`env.DB` 依旧是 `undefined`，
`env.DB.prepare(...)` 会直接抛 `TypeError` —— 这也是「登录 500」最常见的另一半原因。

库 ID 只有在两个地方有意义，且都不是运行时环境变量：

| 场景 | 哪里用 ID | 说明 |
|---|---|---|
| wrangler / CI 部署 | `wrangler.toml` 的 `database_id` | 部署时告诉 Cloudflare「绑定到哪个库」 |
| 走 REST API 直连 D1（不推荐） | 自己攒 URL：`/accounts/{account_id}/d1/database/{database_id}/query` | 需要额外的 API Token，每次查询走 HTTPS，延迟更高 |

也就是说：**用控制台绑定这条路，ID 由 Cloudflare 自己记录，你从头到尾都不用碰它。**
只有你选择「用 wrangler 管理绑定」时，才需要把 ID 写进 `wrangler.toml`。

---

## 七、想用 wrangler 管理绑定的写法

如果你更喜欢 CLI：

1. `wrangler.toml` 里取消注释这四行并填自己的 ID：

   ```toml
   [[d1_databases]]
   binding = "DB"
   database_name = "outlook-email-db"
   database_id = "你的-database-id"
   migrations_dir = "migrations"
   ```

2. 然后：

   ```bash
   pnpm exec wrangler d1 migrations apply DB --remote
   pnpm exec wrangler deploy
   ```

---

## 八、排错

| 现象 | 原因 | 处理 |
|---|---|---|
| 登录 500，提示 `DB_NOT_READY` / 「数据库未就绪」 | 表没建，或库绑错 | 跑三个 SQL；确认绑定变量名是 `DB` |
| 登录 500，提示「服务端密钥未配置」 | `COOKIE_SECRET` 没设 | Settings → Variables and Secrets 补 |
| 部署报 `could not be found [code: 7404]` | `wrangler.toml` 里还留着别人的 `database_id` | 删掉 `[[d1_databases]]` 段（本仓库默认就没有） |
| 页面 404 / 只有 JSON | 静态资源没生效 | 确认 `wrangler.toml` 里有 `[assets] directory = "./public"` |
| 登录后又跳回登录页 | Cookie 签名失败 | 检查 `COOKIE_SECRET` 是否有值、是否被改过 |
| 迁移命令显示 `Resource location: local` | 用 `npm exec` 吃掉了 `--remote` | 改用 `npx` / `pnpm exec` |
| cron 不触发 | 手动创建的 Worker 不自动继承 cron | Settings → Triggers → Cron 加 `0 */6 * * *` |
| 一键授权报 `redirect_uri is not valid` | 默认 Thunderbird Client ID 没登记你的域名 | 用「方式二：手动授权」，或注册自己的 Azure 应用 |

---

## 九、仓库里都放了什么

```
wrangler.toml          Worker 名 / 入口 / [assets] / cron；故意不含 D1 绑定
.dev.vars.example      声明 ADMIN_PASSWORD、COOKIE_SECRET（按钮向导据此弹输入框）
migrations/*.sql       手动在 D1 Console 里执行的建表脚本
docs/GUIDE.md          纯 CLI 部署路线（wrangler 管理一切）
```

业务代码（`src/`、`public/`）与上游一致，未做改动。
