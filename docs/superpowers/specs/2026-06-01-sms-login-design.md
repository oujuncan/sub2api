# 手机号验证码登录认证 — 设计规格

- 日期：2026-06-01
- 状态：设计待实现
- 涉及：后端（Go / Ent / Gin / Redis / Settings）、前端（Vue3 登录页）

## 1. 背景与目标

现有认证以**邮箱**为登录主体（`User.email` 为 `NOT NULL` 唯一列 + 密码哈希），并支持微信/钉钉/LinuxDo/Google/OIDC 等 OAuth 登录（`LoginOrRegisterOAuth`）。验证码基础设施已在邮箱场景成熟：`EmailService.SendVerifyCode/VerifyCode` 将验证码存 Redis（`EmailCache` 接口，TTL、冷却、频控），异步发送走 `EmailQueueService`，并已有 Turnstile 人机校验与 TOTP 二次验证。

本规格新增**手机号 + 短信验证码登录**，并将其确立为**主登录方式**，邮箱降级为可选。

### 目标
- 手机号 + 短信验证码登录/自动注册，作为主登录入口。
- 登录默认无密码（验证码即可），密码为用户可选设置。
- 短信发送采用可插拔抽象，首期实现**阿里云 + 腾讯云**两家，运行时二选一。
- 提供短信调试开关，开启时**不发送真实短信（零扣费）**且验证码固定 `123456`。
- 仅支持中国大陆 +86 号码。

### 非目标（YAGNI）
- 不支持国际号码（E.164 / 国家区号）——首期仅 +86。
- 不引入除阿里云/腾讯云外的第三方短信渠道。
- 不实现短信营销/通知类群发，仅验证码类场景。

## 2. 核心决策（已与用户确认）

| 项 | 决策 |
|----|------|
| 定位 | 手机号 + 验证码为**主登录**，邮箱降级为可选 |
| 密码 | 验证码为主，密码可选（用户可在设置中设密码后用号码+密码登录） |
| 身份模型 | 新增唯一列 `login_name` 作为统一登录句柄 + 新增 `phone` 列；`email` 改可空 |
| 现有用户迁移 | 存量用户 `login_name` 回填为其 `email` |
| 短信渠道 | 可插拔抽象，首期阿里云 + 腾讯云两家，按配置二选一 |
| 配置存储 | 存 Settings；密钥走 `*Configured` 模式（不回传明文）；参考微信配置 |
| 场景 | `login` / `bind` / `reset_password` |
| 调试开关 | `sms_debug_mock`：不发真实短信、验证码固定 `123456`、冷却频控仍生效 |
| 防滥用 | 6 位码、5 分钟 TTL、60 秒冷却、号码/IP 日上限、发码前 Turnstile |
| 号码范围 | 仅 +86，11 位，`^1[3-9]\d{9}$` |

## 3. 数据模型

### 3.1 `User` schema（`ent/schema/user.go`）

```
+ login_name   text  NULL  UNIQUE     // 唯一登录句柄：手机用户存 11 位号码，邮箱用户回填 email
+ phone        text  NULL  UNIQUE     // 规范化 +86 号码（11 位），仅手机用户
  email        text  NULL  UNIQUE     // 由 NOT NULL 改为可空（手机用户无邮箱）
  username      （不变）                // 仍为非唯一显示昵称，OAuth 显示名沿用
  signup_source （复用）                // 手机注册写 "phone"
```

- 约束：`email` 与 `phone` 至少一个非空（应用层校验 + 可选 DB 约束）。
- 格式不冲突：邮箱含 `@`，手机为纯 11 位数字，`login_name` 跨两种格式天然唯一。

### 3.2 `AuthIdentity`（`ent/schema/auth_identity.go`）

`authProviderTypes` 白名单新增 `"phone"`：

```
provider_type    = "phone"
provider_key     = "phone"            // 与现有 email 身份的 key 约定保持一致
provider_subject = 规范化后的 11 位号码
```

复用既有唯一索引 `(provider_type, provider_key, provider_subject)` 与"一用户多身份"模型，绑定/解绑天然支持。

### 3.3 迁移（`backend/migrations/`，编号续 178+）

1. `users` 新增 `login_name`、`phone` 列（nullable）+ 各自唯一索引。
2. `users.email` 去除 `NOT NULL`。
3. backfill：`UPDATE users SET login_name = email WHERE login_name IS NULL`。
4. 校验 backfill 后 `login_name` 无空且唯一。

> 迁移需与 Ent codegen 保持一致（先改 schema 再生成，再补手写 SQL 迁移）。

## 4. 短信抽象层

目录：`internal/pkg/sms/`（抽象）+ `internal/pkg/sms/provider/{aliyun,tencent,mock}.go`；service 层 `internal/service/sms_service.go` 编排。

### 4.1 Provider 接口与场景

```go
type Scene string

const (
    SceneLogin         Scene = "login"          // 登录 / 自动注册
    SceneBind          Scene = "bind"           // 绑定手机
    SceneResetPassword Scene = "reset_password" // 找回密码
)

type Provider interface {
    Name() string                                          // "aliyun" | "tencent" | "mock"
    SendCode(ctx context.Context, phone, code string, scene Scene) error
}
```

### 4.2 Registry

- 按 `sms_active_provider` 选择启用渠道；按 `Scene` 解析对应模板 ID。
- 当 `sms_debug_mock=true` 时短路到 `mock` provider，优先级高于 `active_provider`。

### 4.3 阿里云 / 腾讯云实现

- `provider/aliyun.go`：阿里云 SMS（SignName + per-scene TemplateCode，模板参数填验证码）。
- `provider/tencent.go`：腾讯云 SMS（SdkAppId + SignName + per-scene TemplateId）。
- `provider/mock.go`：不调用真实 API，记一条"模拟发送"日志。

## 5. 配置（存 Settings，参考微信配置模式）

密钥类字段对前端只暴露 `*Configured bool`（已配置与否），不回传明文。

```
# 总开关与渠道选择
sms_enabled              bool
sms_active_provider      "aliyun" | "tencent"
sms_debug_mock           bool        // 调试/模拟：不发真实短信、验证码固定 123456

# 阿里云
sms_aliyun_access_key_id          string
sms_aliyun_access_key_secret      string  (→ Configured)
sms_aliyun_sign_name              string
sms_aliyun_tpl_login              string
sms_aliyun_tpl_bind               string
sms_aliyun_tpl_reset              string

# 腾讯云
sms_tencent_secret_id             string
sms_tencent_secret_key            string  (→ Configured)
sms_tencent_sdk_app_id            string
sms_tencent_sign_name             string
sms_tencent_tpl_login             string
sms_tencent_tpl_bind              string
sms_tencent_tpl_reset             string
```

落点：`internal/service/settings_view.go`（带类型字段）、`internal/handler/dto/settings.go`（DTO + `*Configured`）、`internal/service/setting_service.go`（读写 + 校验）、admin setting handler。

## 6. 验证码存储与防滥用

复刻邮箱版 `EmailService` / `EmailCache`，新增 `SMSCache`（或通用 `VerificationCode` cache，键 `scene:phone`）：

- 6 位数字验证码（调试模式固定 `123456`），存 Redis，**TTL 5 分钟**。
- 最大尝试次数（如 5 次）后失效。
- **冷却 60 秒**：返回倒计时，复用 `SendVerifyCodeResult{Countdown:60}`。
- **每号码 / 每 IP 日发送上限**，防短信轰炸。
- 发码前复用现有 **Turnstile** 校验（`VerifyTurnstile`）。
- 调试模式下冷却/频控仍生效，仅"发送"被短路。
- 首期采用**同步发送**（发码接口内直接调用 provider，失败即返回错误）；异步队列（类比 `EmailQueueService`）作为后续增强，不在本期范围。

## 7. 核心流程

### 7.1 发码 `SendSMSCode(ctx, scene, phone, turnstileToken, remoteIP)`
1. 校验 `sms_enabled`；2. Turnstile 校验；3. 号码格式 `^1[3-9]\d{9}$`；
4. 冷却/频控检查；5. 生成验证码（调试→`123456`）存 Redis；
6. 经 Registry 选 provider 发送（调试→mock）；7. 返回 `{countdown:60}`。

### 7.2 登录即注册 `LoginWithSMS(ctx, phone, code, invitationCode?)`（类比 `LoginOrRegisterOAuth`）
1. 校验验证码（消费一次）；
2. 按 `phone` 查 `AuthIdentity(phone)` → `User`；
3. **不存在** → 创建用户：`login_name=phone`、`phone=phone`、`email=NULL`、`signup_source="phone"`；
   走 `postAuthUserBootstrap` + `assignSubscriptions`（复用注册赠送/affiliate/邀请码逻辑），写入 `AuthIdentity(phone)`；
4. **存在** → 直接登录；
5. 签发 `GenerateTokenPair`，返回 TokenPair + user。

### 7.3 绑定 / 解绑（已登录）
- `BindPhone(ctx, userID, phone, code)`：校验码 → 写 `User.phone` + `AuthIdentity(phone)`；号码已被他人占用则拒绝。
- `UnbindPhone(ctx, userID)`：解绑前必须保证账号至少保留一种登录方式（邮箱或密码或其他 OAuth），否则拒绝。

### 7.4 密码可选
- 用户在设置中设置密码后，支持 `手机号 + 密码` 登录，复用现有 `HashPassword/CheckPassword`。

## 8. API 端点（挂在现有 `/api/v1/auth`）

```
POST /auth/sms/send-code   { phone, scene, turnstileToken }        → { countdown }
POST /auth/sms/login       { phone, code, invitationCode? }        → TokenPair + user
POST /auth/sms/bind        { phone, code }            (需登录)      → ok
POST /auth/sms/unbind      {}                          (需登录)     → ok
```

落点：`internal/handler/auth_handler.go`（或新增 `auth_sms.go`）+ 路由注册 `wire.go`。

## 9. 与现有代码的耦合与迁移风险

`email` 由 `NOT NULL` 改可空，需逐一审计"假设 email 非空"的路径，手机用户跳过邮件类能力：

- 余额通知（`balance_notify_*`）、找回密码（`RequestPasswordReset` 依赖 email）、通知邮件、注册邮箱策略（`validateRegistrationEmailPolicy`）。
- `isReservedEmail` / 合成邮箱逻辑确认不与手机用户冲突。
- DTO 序列化（`email` 可能为空，前端展示需容错）。
- 唯一索引：email 由"非空唯一"变为"可空唯一"（Postgres 允许多个 NULL）。

前端：
- 登录页新增"手机号"Tab：号码输入、获取验证码（60s 倒计时）、登录。
- 用户设置新增绑定/解绑手机、设置密码入口。
- admin 设置页新增整组 SMS 配置（含调试开关、渠道选择、两家凭据与模板）。
- i18n 文案（中/英/日）。

## 10. 测试策略

- 单测：号码规范化与格式校验；验证码生成/过期/尝试次数；冷却与日频控；调试模式固定 `123456` 且不触发真实 provider；`LoginWithSMS` 新建与已存在两分支；绑定/解绑保留至少一种登录方式；email 可空后相关路径不 panic。
- Provider：阿里云/腾讯云用 mock HTTP；`mock` provider 验证短路逻辑。
- Redis 用现有 `miniredis`；集成测试复用 `testcontainers`（PG + Redis）。
- Settings：`*Configured` 不回传明文、调试开关读写、渠道切换。

## 11. 实现顺序建议

1. Ent schema 变更（`login_name`/`phone`/`email` 可空）+ codegen + 迁移 + backfill。
2. `AuthIdentity` 加 `phone` provider。
3. 短信抽象层（接口/Scene/Registry）+ 阿里云 + 腾讯云 + mock。
4. Settings 配置项（含调试开关）+ admin handler/DTO。
5. `SMSCache` + `sms_service`（发码/校验/防滥用）。
6. `AuthService.LoginWithSMS` / `BindPhone` / `UnbindPhone` + 端点。
7. email 可空相关路径审计与适配。
8. 前端登录页 / 设置页 / admin 配置页 + i18n。
9. 测试补齐。
