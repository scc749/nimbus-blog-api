# Nimbus Blog API — 项目分析与复刻计划

## 一、项目概述

Nimbus Blog API 是一个基于 Go 语言构建的现代博客系统后端服务，采用 Clean Architecture（整洁架构）分层设计，提供**后台管理（Admin）**与**公开接口（Public V1）**两套独立的 API 体系。项目以类型安全、分层清晰、接口稳定、可观测为核心目标，涵盖内容管理、评论系统、用户认证、通知推送、文件服务等完整博客业务场景。

---

## 二、技术栈详解

### 2.1 核心框架与语言

| 类别 | 技术 | 版本 | 说明 |
|------|------|------|------|
| 语言 | Go | 1.25.5 | 主开发语言 |
| Web 框架 | Fiber v3 | 3.0.0 | 高性能 HTTP 框架，基于 fasthttp |
| ORM | GORM | 1.31.1 | 数据库 ORM |
| 代码生成 | GORM Gen | 0.3.27 | 类型安全的查询代码生成 |
| 依赖注入 | Google Wire | 0.7.0 | 编译时依赖注入 |

### 2.2 数据存储层

| 类别 | 技术 | 用途 |
|------|------|------|
| 关系数据库 | PostgreSQL | 主数据持久化（文章、用户、评论、设置等） |
| 缓存/会话 | Redis | 验证码存储、管理员 Session、Refresh Token 存储、2FA 临时数据 |
| 对象存储 | MinIO | 文件上传与访问（文章封面、内容图片、头像等） |

### 2.3 认证与安全

| 类别 | 技术 | 用途 |
|------|------|------|
| JWT | golang-jwt/jwt/v5 | 用户端 Access/Refresh Token 签发与验证 |
| Session | Fiber Session + Redis Store | 管理端会话管理 |
| 2FA | pquerna/otp (TOTP) | 管理员双因素认证 |
| 密码加密 | golang.org/x/crypto (bcrypt) | 管理员密码哈希 |
| 验证码 | mojocn/base64Captcha | 图形验证码生成 |

### 2.4 通信与外部服务

| 类别 | 技术 | 用途 |
|------|------|------|
| 邮件 | gomail.v2 + SMTP | 邮件验证码发送 |
| LLM | openai/openai-go | AI 辅助 Slug 生成（DeepSeek） |
| 翻译 | go-googletrans | 标题翻译辅助 Slug 生成 |
| SSE | 自研 ssehub | 站内通知实时推送 |

### 2.5 开发工具链

| 类别 | 技术 | 用途 |
|------|------|------|
| API 文档 | swaggo/swag + Swagger UI | 接口文档生成与展示 |
| 配置管理 | spf13/viper | YAML 配置加载 |
| 参数校验 | go-playground/validator/v10 | 请求 DTO 校验 |
| 日志 | rs/zerolog | 结构化日志 |
| 数据库迁移 | golang-migrate/migrate | SQL 迁移管理 |
| JSON | goccy/go-json | 高性能 JSON 序列化 |
| UUID | google/uuid | 唯一标识生成 |

---

## 三、架构设计分析

### 3.1 Clean Architecture 分层

项目严格遵循 Clean Architecture，依赖方向始终由外向内：

```
Entry Point (cmd/)
    ↓
Controller (internal/controller/http/)
    ↓ 依赖 UseCase 接口
Use Case (internal/usecase/)
    ↓ 依赖 Repo 接口 + Entity
Repository (internal/repo/)
    ↓ 依赖 Entity + 基础设施
Entity (internal/entity/)       ← 核心领域，无外部依赖
```

**关键约束：**
- Controller 只依赖 UseCase 接口，不直接依赖 Repo 实现
- Repo 只向下依赖基础设施，不依赖 UseCase
- **UseCase 同层不互调**：不允许 UseCase A 注入/调用 UseCase B，跨领域协作通过抽象到 Repo（如 `repo.Notifier`）或在当前 UseCase 内实现
- Entity 层为纯领域模型，无任何外部依赖

### 3.2 目录结构

```
nimbus-blog-api/
├── cmd/                          # 程序入口
│   ├── app/main.go               # 主服务入口
│   ├── gen/main.go               # GORM Gen 代码生成
│   ├── keys/main.go              # 密钥生成工具
│   └── migrate/main.go           # 数据库迁移
├── config/
│   └── config.go                 # 配置结构体 + Viper 加载
├── config.yaml                   # 配置文件
├── docs/                         # Swagger 文档
├── internal/
│   ├── app/                      # Wire DI + 应用生命周期
│   │   ├── app.go                # 应用启动/关闭
│   │   ├── wire.go               # Wire Provider 定义
│   │   └── wire_gen.go           # Wire 生成文件
│   ├── entity/                   # 领域实体（纯结构体）
│   ├── usecase/                  # 业务逻辑层
│   │   ├── contracts.go          # 所有 UseCase 接口定义
│   │   ├── input/                # UseCase 入参 DTO
│   │   ├── output/               # UseCase 出参 DTO
│   │   └── {domain}/             # 各领域 UseCase 实现
│   ├── repo/                     # 数据访问层
│   │   ├── contracts.go          # 所有 Repo 接口定义
│   │   ├── persistence/          # PostgreSQL 实现（GORM Gen）
│   │   ├── cache/                # Redis 缓存实现
│   │   ├── storage/              # MinIO 对象存储实现
│   │   ├── messaging/            # SMTP 邮件实现
│   │   ├── notification/         # 通知推送实现
│   │   ├── viewbuffer/           # 浏览量异步缓冲写入
│   │   └── webapi/               # 外部 API（LLM、翻译）
│   └── controller/http/          # HTTP 控制器层
│       ├── router.go             # 主路由注册
│       ├── admin/                # Admin API 模块
│       │   ├── controller.go     # 控制器结构体
│       │   ├── router.go         # 路由注册
│       │   ├── {domain}.go       # 各领域 Handler
│       │   ├── request/          # 请求 DTO
│       │   └── response/         # 响应 DTO + 业务码
│       ├── v1/                   # Public API 模块
│       ├── shared/               # 共享工具（信封、分页、辅助函数）
│       ├── bizcode/              # 全局业务状态码
│       └── middleware/           # 中间件
├── pkg/                          # 可复用基础设施包
│   ├── httpserver/               # Fiber Server 封装
│   ├── logger/                   # Zerolog 日志封装
│   ├── postgres/                 # GORM Postgres 连接封装
│   ├── redis/                    # Redis 连接封装
│   ├── minio/                    # MinIO 客户端封装
│   └── ssehub/                   # SSE Hub（内存级连接管理 + 事件推送）
└── migrations/                   # SQL 迁移文件
```

### 3.3 依赖注入（Wire）

项目使用 Google Wire 进行编译时依赖注入，所有 Provider 集中定义在 `internal/app/wire.go` 中：

- **基础设施层**：Postgres、Redis、MinIO Client、Logger
- **Repo 层**：18 个 Repo 实现（Persistence + Cache + Storage + Messaging + WebAPI + Notification + ViewBuffer）
- **UseCase 层**：12 个 UseCase 实现
- **HTTP 层**：Server + Router 组装

Wire 生成 `wire_gen.go`，确保编译时依赖关系正确。

---

## 四、功能模块详解

### 4.1 认证与安全模块

#### 管理端认证（Admin Auth）
- **Session + Cookie 鉴权**：管理员登录后通过 Redis 存储 Session，Cookie 名 `fiber_session`
- **密码重置**：首次登录强制修改密码（`must_reset_password` 标志）
- **双因素认证（2FA/TOTP）**：
  - Setup 流程：生成 TOTP Secret → 加密存储到 Redis → 生成 QR Code → 验证后持久化到 DB
  - 恢复码：加密存储，支持验证并标记已使用
  - 禁用/重置 2FA
- **加密方案**：2FA Secret 使用 AES-GCM 加密，EncryptionKey 配置在 config.yaml

#### 用户端认证（User Auth）
- **JWT 双 Token 机制**：
  - Access Token：短期（默认 15 分钟），Bearer 方式携带
  - Refresh Token：长期（默认 7 天），HttpOnly Cookie 存储
- **注册**：用户名 + 邮箱 + 密码
- **登录**：邮箱 + 密码 → 返回 Token Pair
- **Token 刷新**：Refresh Token 换取新的 Token Pair
- **忘记密码**：邮箱验证码 → 重置密码
- **登出**：Refresh Token 加入黑名单（DB 持久化），清除 Redis 中的当前 Token
- **会话验证**：每次请求同时校验 Access Token + Refresh Token 有效性

### 4.2 内容管理模块（Content）

#### 文章（Post）
- **Admin CRUD**：创建、更新、删除、列表（分页/搜索/排序/筛选）
- **Public 查询**：
  - 公开列表：仅 `status=published` 的文章，支持分页、分类筛选、标签筛选、置顶优先
  - 详情查询：通过 Slug 访问，UseCase 层校验 `status=published`
- **Slug 生成**：翻译标题 → 英文 → slugify（优先 Google Translate → 回退 LLM → 最终报错）
- **文章点赞**：Toggle 机制（已赞则取消），计数器同步更新
- **浏览量记录**：异步缓冲写入（viewbuffer），定期 Flush 到 DB
- **文章封面**：通过 `featured_image` 关联文件，支持清空（空字符串 → nil → DB NULL）

#### 分类（Category）
- Admin CRUD + Public 全量列表
- `post_count` 由数据库触发器自动维护

#### 标签（Tag）
- Admin CRUD + Public 全量列表
- 多对多关系（post_tags 表）
- `post_count` 由数据库触发器自动维护

### 4.3 评论系统（Comment）

- **评论提交**：用户提交评论，默认 `status=pending`（待审核）
- **评论审核**：管理员变更状态（approved/rejected/spam）
- **通知触发**：仅在 `UpdateCommentStatus(approved)` 时触发通知
  - 通知评论者：评论审核通过
  - 通知被回复者：回复评论审核通过（跳过自回复）
- **评论点赞**：Toggle 机制
- **评论删除**：用户可删除自己的评论，管理员可删除任何评论
- **嵌套评论**：通过 `parent_id` 实现回复关系

### 4.4 反馈系统（Feedback）

- **Public 提交**：匿名用户可提交反馈（姓名、邮箱、类型、主题、内容）
- **反馈类型**：general / bug / feature / ui
- **反馈状态**：pending → processing → resolved → closed
- **Admin 管理**：列表、详情、状态更新、删除

### 4.5 友链管理（Link）

- **Admin CRUD**：名称、URL、描述、Logo、排序、状态
- **Public 查询**：仅返回 `status=active` 的友链
- **排序**：通过 `sort_order` 字段控制前台展示顺序

### 4.6 文件服务（File）

- **对象存储（MinIO）**：
  - 预签名上传 URL：前端直传 MinIO
  - 预签名下载 URL：文件访问
  - 对象删除
- **文件元数据（DB）**：
  - 记录 object_key、文件名、大小、MIME 类型、用途、关联资源 ID
  - 文件用途枚举：post_cover / post_content / avatar
  - 支持资源绑定与解绑
- **外部访问基址**：`minio.external_base_url` 配置，解决内网地址不可达问题

### 4.7 站点设置（Setting）

- **键值化配置**：setting_key + setting_value + setting_type
- **类型支持**：string / number / boolean / json
- **公开/私有控制**：`is_public` 标志决定是否对前端可见
- **预置配置**：站点名称、标题、描述、标语、Hero、ICP 备案、FAQ、个人信息、技术栈、社交链接等
- **Admin 管理**：列表、按键查询、Upsert

### 4.8 通知中心（Notification）

- **通知类型**：comment_reply / comment_approved / admin_message
- **数据模型**：通知记录（DB 持久化）+ Meta JSONB（关联信息与跳转链接）
- **SSE 实时推送**：通过 `ssehub.Hub` 管理内存级客户端连接
- **通知操作**：列表、未读计数、标记已读、全部已读、删除
- **Admin 发送**：管理员可向指定用户发送站内消息

### 4.9 验证码与邮件

- **图形验证码**：base64Captcha 生成，Redis 存储（5 分钟 TTL）
- **邮件验证码**：SMTP 发送，Redis 存储（10 分钟 TTL）
- **验证流程**：生成 → 返回 ID + 图片 → 前端提交 ID + 答案 → 后端验证

---

## 五、数据库设计

### 5.1 数据表总览

| 表名 | 说明 | 关键字段 |
|------|------|---------|
| admins | 管理员 | username, password_hash, two_factor_secret |
| admin_recovery_codes | 2FA 恢复码 | admin_id, code_hash, used_at |
| users | 用户 | name, email, password_hash, status, auth_provider |
| refresh_token_blacklist | Refresh Token 黑名单 | user_id, token_hash, expires_at |
| posts | 文章 | title, slug, status, category_id, author_id |
| categories | 分类 | name, slug, post_count |
| tags | 标签 | name, slug, post_count |
| post_tags | 文章-标签关联 | post_id, tag_id |
| post_likes | 文章点赞 | post_id, user_id |
| post_views | 文章浏览记录 | post_id, ip_address, referer |
| comments | 评论 | post_id, parent_id, user_id, status |
| comment_likes | 评论点赞 | comment_id, user_id |
| feedbacks | 反馈 | type, status, name, email |
| links | 友链 | name, url, status, sort_order |
| site_settings | 站点设置 | setting_key, setting_value, setting_type, is_public |
| files | 文件元数据 | object_key, usage, resource_id |
| notifications | 通知 | user_id, type, meta(JSONB), is_read |

### 5.2 数据库设计亮点

- **ENUM 类型**：PostgreSQL 原生 ENUM 确保数据一致性（user_status, post_status, comment_status 等 9 种）
- **自动更新时间戳**：`update_updated_at_column()` 触发器自动维护 `updated_at`
- **计数器触发器**：`maintain_category_post_count()` 和 `maintain_tag_post_count()` 自动维护 post_count
- **全文搜索索引**：`pg_trgm` 扩展 + GIN 索引支持文章标题/内容/摘要模糊搜索
- **条件索引**：`idx_posts_category_published_at` 仅索引已发布文章，提升查询效率
- **唯一约束**：email 条件唯一（WHERE email IS NOT NULL）、auth_provider + auth_openid 联合唯一
- **种子数据**：初始管理员账号 + 站点设置预置数据

### 5.3 ER 关系

```
admins ──1:N──> posts (author_id)
admins ──1:N──> files (uploader_id)
admins ──1:N──> admin_recovery_codes

categories ──1:N──> posts (category_id)
posts ──M:N──> tags (through post_tags)
posts ──1:N──> comments (post_id)
posts ──1:N──> post_likes
posts ──1:N──> post_views

users ──1:N──> comments (user_id)
users ──1:N──> post_likes (user_id)
users ──1:N──> comment_likes (user_id)
users ──1:N──> notifications (user_id)
users ──1:N──> refresh_token_blacklist

comments ──self──> comments (parent_id)
comments ──1:N──> comment_likes
```

---

## 六、API 路由设计

### 6.1 Admin API（`/api/admin/*`）— Session Cookie 鉴权

| 模块 | 路由 | 方法 | 说明 | 鉴权 |
|------|------|------|------|------|
| Auth | /auth/login | POST | 管理员登录 | 公开 |
| Auth | /auth/reset | POST | 重置密码 | 公开 |
| Auth | /auth/logout | POST | 登出 | 公开 |
| Auth | /auth/profile | GET | 获取个人信息 | Session |
| Auth | /auth/profile | PUT | 更新个人信息 | Session |
| Auth | /auth/password | PUT | 修改密码 | Session |
| Auth | /auth/2fa/setup | POST | 2FA 设置 | Session |
| Auth | /auth/2fa/verify | POST | 2FA 验证 | Session |
| Auth | /auth/2fa/disable | POST | 2FA 禁用 | Session |
| Auth | /auth/2fa/recovery/reset | POST | 2FA 恢复码重置 | Session |
| Content | /content/posts | GET | 文章列表 | Session |
| Content | /content/posts/:id | GET | 文章详情 | Session |
| Content | /content/posts | POST | 创建文章 | Session |
| Content | /content/posts/:id | PUT | 更新文章 | Session |
| Content | /content/posts/:id | DELETE | 删除文章 | Session |
| Content | /content/categories | GET/POST | 分类列表/创建 | Session |
| Content | /content/categories/:id | PUT/DELETE | 分类更新/删除 | Session |
| Content | /content/tags | GET/POST | 标签列表/创建 | Session |
| Content | /content/tags/:id | PUT/DELETE | 标签更新/删除 | Session |
| Content | /content/generate-slug | POST | 生成 Slug | Session |
| Comments | /comments | GET | 评论列表 | Session |
| Comments | /comments/:id/status | PUT | 审核评论 | Session |
| Comments | /comments/:id | DELETE | 删除评论 | Session |
| Users | /users | GET | 用户列表 | Session |
| Users | /users/:id/status | PUT | 更新用户状态 | Session |
| Feedbacks | /feedbacks | GET | 反馈列表 | Session |
| Feedbacks | /feedbacks/:id | GET/PUT/DELETE | 反馈详情/更新/删除 | Session |
| Links | /links | GET/POST | 友链列表/创建 | Session |
| Links | /links/:id | PUT/DELETE | 友链更新/删除 | Session |
| Settings | /settings | GET | 设置列表 | Session |
| Settings | /settings/:key | GET/PUT | 设置查询/更新 | Session |
| Files | /files/ | GET | 文件列表 | Session |
| Files | /files/upload-url | POST | 生成上传 URL | Session |
| Files | /* | DELETE | 删除文件 | Session |
| Notifications | /notifications | POST | 发送通知 | Session |

### 6.2 Public V1 API（`/api/v1/*`）— 部分 Bearer JWT 鉴权

| 模块 | 路由 | 方法 | 说明 | 鉴权 |
|------|------|------|------|------|
| Captcha | /captcha/generate | GET | 生成验证码 | 公开 |
| Email | /email/send-code | POST | 发送邮件验证码 | 公开 |
| Auth | /auth/register | POST | 用户注册 | 公开 |
| Auth | /auth/login | POST | 用户登录 | 公开 |
| Auth | /auth/refresh | POST | 刷新 Token | 公开 |
| Auth | /auth/forgot | POST | 忘记密码 | 公开 |
| Auth | /auth/logout | POST | 用户登出 | JWT |
| User | /user/me | GET | 获取个人信息 | JWT |
| User | /user/profile | PUT | 更新个人信息 | JWT |
| User | /user/password | PUT | 修改密码 | JWT |
| Content | /content/posts | GET | 公开文章列表 | 可选 JWT |
| Content | /content/posts/:slug | GET | 文章详情 | 可选 JWT |
| Content | /content/categories | GET | 分类列表 | 公开 |
| Content | /content/tags | GET | 标签列表 | 公开 |
| Content | /content/posts/:id/likes | POST/DELETE | 文章点赞/取消 | JWT |
| Comments | /content/posts/:id/comments | GET | 评论列表 | 可选 JWT |
| Comments | /content/posts/:id/comments | POST | 提交评论 | JWT |
| Comments | /comments/:id/likes | POST/DELETE | 评论点赞/取消 | JWT |
| Comments | /comments/:id | DELETE | 删除自己的评论 | JWT |
| Feedbacks | /feedbacks/ | POST | 提交反馈 | 公开 |
| Links | /links/ | GET | 友链列表 | 公开 |
| Files | /files/* | GET | 获取文件 URL | 公开 |
| Settings | /settings/ | GET | 公开设置列表 | 公开 |
| Notifications | /notifications/stream | GET | SSE 通知流 | 公开 |
| Notifications | /notifications/ | GET | 通知列表 | JWT |
| Notifications | /notifications/unread | GET | 未读计数 | JWT |
| Notifications | /notifications/:id/read | PUT | 标记已读 | JWT |
| Notifications | /notifications/read-all | PUT | 全部已读 | JWT |
| Notifications | /notifications/:id | DELETE | 删除通知 | JWT |

---

## 七、核心设计模式与规范

### 7.1 响应信封（Envelope）

```json
{
  "code": "0000",
  "message": "ok",
  "data": { ... }
}
```

- 成功：`code: "0000"`
- 分页响应：`data.list` + `data.current_page` / `data.page_size` / `data.total_items` / `data.total_pages`

### 7.2 业务状态码（AABB 格式）

- `AA`：模块编号（00=全局）
- `BB`：具体错误（01-19 数据相关、20-39 认证授权、40-59 业务逻辑、60-79 系统错误）
- 示例：`0001` 参数错误、`0020` 未授权、`0022` Token 过期、`0061` 数据库错误

### 7.3 分页约定

- 默认 `page=1, page_size=10`，最大 `page_size=100`
- 支持排序：`sort_by` + `order`（asc/desc）
- 支持筛选：`filter.{key}={value}`
- 支持搜索：`keyword`

### 7.4 路径 ID 统一处理模式

- Admin 更新接口 `PUT /api/admin/{resource}/{id}`
- 路径 `id` 赋值给请求体 `body.ID`，确保 DTO 校验一致性
- 前端无需在请求体重复携带路径 ID

### 7.5 字段清空语义

- 前端发送空字符串 → Controller 归一化为 nil → Repo 写入 DB NULL
- 未提供字段 → 不修改（Repo 不更新该列）

### 7.6 Optional JWT 中间件

- 公开接口使用 `NewOptionalUserJWTMiddleware`：有 Token 则解析用户信息，无 Token 也可访问
- 用于文章列表/详情等需要判断用户是否点赞但不需要强制登录的场景

---

## 八、复刻计划

### 阶段一：项目初始化与基础设施（1-2 周）

1. **项目脚手架搭建**
   - 初始化 Go Module
   - 创建目录结构（cmd / config / internal / pkg / migrations）
   - 配置 Viper + config.yaml
   - 配置 Zerolog 日志

2. **基础设施包开发（pkg/）**
   - `pkg/httpserver`：Fiber Server 封装（Functional Options 模式）
   - `pkg/postgres`：GORM Postgres 连接封装
   - `pkg/redis`：Redis 连接封装
   - `pkg/minio`：MinIO 客户端封装（含 Bucket 自动创建）
   - `pkg/logger`：Zerolog 封装
   - `pkg/ssehub`：SSE Hub 实现

3. **Docker 开发环境**
   - PostgreSQL 容器
   - Redis 容器
   - MinIO 容器

### 阶段二：数据库设计与迁移（1 周）

1. **设计数据库 Schema**
   - 创建 ENUM 类型
   - 创建所有数据表
   - 创建触发器（updated_at 自动维护、post_count 自动维护）
   - 创建索引（条件索引、全文搜索索引）
   - 种子数据（初始管理员、站点设置）

2. **迁移工具**
   - 实现 `cmd/migrate` 入口（golang-migrate）
   - 编写 up/down SQL 文件

3. **GORM Gen 代码生成**
   - 实现 `cmd/gen` 入口
   - 生成 Model 和 Query 代码

### 阶段三：Entity 与 Repo 层（1-2 周）

1. **Entity 定义**
   - 定义所有领域实体（Admin, User, Post, Comment, Feedback, Link, File, Notification, SiteSetting, Category, Tag, PostView）
   - 定义枚举常量

2. **Repo 接口定义**
   - 在 `internal/repo/contracts.go` 中定义所有 Repo 接口

3. **Repo 实现**
   - Persistence：基于 GORM Gen 实现所有 PostgreSQL Repo
   - Cache：Redis 实现（CaptchaStore, EmailCodeStore, RefreshTokenStore, AdminTwoFASetupStore）
   - Storage：MinIO 实现（ObjectStore）
   - Messaging：SMTP 实现（EmailSender）
   - WebAPI：Translation + LLM 实现
   - ViewBuffer：浏览量缓冲写入实现
   - Notification：Notifier 实现（DB 持久化 + SSE 推送）

### 阶段四：UseCase 层（1-2 周）

1. **UseCase 接口定义**
   - 在 `internal/usecase/contracts.go` 中定义所有 UseCase 接口

2. **Input/Output DTO 定义**
   - `internal/usecase/input/`：所有入参结构体
   - `internal/usecase/output/`：所有出参结构体（含泛型 `ListResult[T]`, `AllResult[T]`）

3. **UseCase 实现**
   - Auth（AdminAuth + UserAuth）：登录、注册、Token 管理、2FA
   - Captcha：验证码生成与验证
   - Email：邮件验证码发送与验证
   - File：文件上传/下载/删除 + 元数据管理
   - User：用户管理
   - Content：文章/分类/标签 CRUD + Slug 生成 + 点赞 + 浏览量
   - Comment：评论提交/审核/点赞 + 通知触发
   - Feedback：反馈提交与管理
   - Link：友链管理
   - Setting：站点设置管理
   - Notification：通知 CRUD + SSE 推送

### 阶段五：Controller 与路由层（1-2 周）

1. **共享组件**
   - Envelope 响应格式
   - 分页解析与构造
   - 业务状态码定义
   - 辅助函数

2. **中间件**
   - Logger：请求日志
   - Recovery：panic 恢复
   - AdminSession：管理员 Session 鉴权
   - UserJWT：用户 JWT 鉴权
   - OptionalUserJWT：可选 JWT 鉴权

3. **Admin Controller**
   - Request/Response DTO 定义
   - 各领域 Handler 实现
   - 路由注册

4. **V1 Controller**
   - Request/Response DTO 定义
   - 各领域 Handler 实现
   - 路由注册

5. **主路由注册**
   - CORS 配置
   - Swagger 路由
   - 健康检查
   - Admin / V1 分组

### 阶段六：依赖注入与集成（1 周）

1. **Wire 配置**
   - 在 `internal/app/wire.go` 中定义所有 Provider
   - 生成 `wire_gen.go`

2. **应用生命周期**
   - 实现 `internal/app/app.go`：启动、信号监听、优雅关闭

3. **密钥生成工具**
   - 实现 `cmd/keys`：生成 JWT Secret 和 2FA Encryption Key

### 阶段七：测试与文档（1 周）

1. **单元测试**
   - UseCase 层测试（Mock Repo）
   - Controller 层测试

2. **Swagger 文档**
   - 添加注解
   - 生成文档
   - 格式校验测试

3. **集成测试**
   - 端到端 API 测试

### 阶段八：优化与部署（1 周）

1. **性能优化**
   - 数据库查询优化
   - 缓存策略
   - 连接池调优

2. **部署准备**
   - Docker 镜像构建
   - Docker Compose 编排
   - Nginx 反向代理配置
   - 环境变量管理

---

## 九、复刻注意事项

### 9.1 必须遵循的原则

1. **Clean Architecture 依赖方向**：Controller → UseCase → Repo → Infrastructure，绝不反向依赖
2. **UseCase 同层不互调**：跨领域协作通过 Repo 抽象或在当前 UseCase 内实现
3. **接口集中定义**：UseCase 接口在 `contracts.go`，Repo 接口在 `contracts.go`
4. **ID 统一使用 int64**：对应数据库 BIGSERIAL/BIGINT
5. **所有方法第一个参数为 context.Context**

### 9.2 安全要点

1. **密钥管理**：JWT Secret、2FA Encryption Key 不提交到仓库，使用 `cmd/keys` 生成
2. **密码存储**：管理员使用 bcrypt（pgcrypto），用户使用 Go 的 bcrypt
3. **2FA Secret 加密**：使用 AES-GCM 加密存储，EncryptionKey 配置化管理
4. **Refresh Token 黑名单**：DB 持久化 + Redis 当前值双重校验
5. **CORS 配置**：明确允许的 Origin，携带 Credentials

### 9.3 开发顺序建议

1. 先完成基础设施层（pkg/），确保可独立测试
2. 再完成 Entity + Repo 接口 + Repo 实现，确保数据层可用
3. 然后完成 UseCase 接口 + UseCase 实现，确保业务逻辑可用
4. 最后完成 Controller + 路由 + Wire 集成，确保端到端可用
5. 每完成一个模块就编写测试，不要等到最后

### 9.4 可选增强方向

1. **OAuth2 社交登录**：当前已预留 `auth_provider` / `auth_openid` 字段
2. **Prometheus 监控**：配置中已有 Metrics 开关，可集成 fiberprometheus
3. **Rate Limiting**：API 限流
4. **国际化**：错误消息多语言支持
5. **WebSocket**：替代 SSE 实现双向通信
6. **全文搜索引擎**：集成 Elasticsearch/Meilisearch 替代 pg_trgm
7. **缓存层**：Redis 缓存热点数据（文章列表、站点设置）
8. **CI/CD**：GitHub Actions 自动化测试与部署
