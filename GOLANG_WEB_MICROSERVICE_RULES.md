# Golang Web 微服务开发规则

本文档基于当前项目（cloud_shang_ji_server）的代码结构与约定，整理 Golang Web 微服务的开发规则，供新功能开发与代码评审时遵循。

---

## 1. 项目结构约定

### 1.1 目录布局

```
<module>/
├── cmd/                    # 各服务入口，一个子目录一个可执行程序
│   ├── api-gateway/        # 网关
│   ├── auth-service/
│   ├── user-service/
│   ├── order-service/
│   ├── ledger-service/
│   ├── feast-service/
│   ├── message-service/
│   └── common-service/
├── internal/               # 私有代码，不对外暴露
│   ├── api/                # 网关/路由层公共能力
│   │   ├── middleware/     # 鉴权、限流、RequestID、访问日志等
│   │   └── routes/         # 路由注册、健康检查、探针、指标
│   ├── config/             # 配置加载（viper + 环境变量）
│   ├── domain/
│   │   └── models/         # 领域模型（与 DB 表对应，统一放此处）
│   ├── pkg/                # 内部公共包（database、logger、consulutil、serverutil）
│   └── <domain>/           # 按业务域划分：auth、user、order、ledger、feast、message、common
│       ├── http/           # HTTP Handler + 路由注册函数
│       ├── service/        # 业务逻辑
│       └── repository/     # 数据访问（GORM）
├── pkg/                    # 可被外部引用的公共包
│   ├── auth/               # JWT 签发与校验
│   ├── errors/             # AppError 及工厂方法
│   ├── response/           # 统一 HTTP 响应（Data/List/Error）
│   └── validator/          # 结构体验证
├── configs/                # 配置文件（config.yaml / config.dev.yaml 等）
├── migrations/             # 数据库迁移（如 migrate.go）
└── docs/                   # 文档
```

- **cmd/**：每个微服务一个 `main.go`，只做配置加载、依赖初始化、路由挂载、Consul 注册（可选）、HTTP 服务启动与优雅退出。
- **internal/**：业务与基础设施代码；新增业务域时在 `internal` 下新增 `<domain>/http`、`<domain>/service`、`<domain>/repository`。
- **pkg/**：与具体服务解耦的通用能力（错误码、响应格式、JWT、校验），可被多个 cmd 引用。

### 1.2 包命名与引用

- 使用 **小写、简短** 的包名：`http`、`service`、`repository`、`models`。
- 引用 internal 子包时使用 **清晰别名** 避免冲突，例如：
  - `userhttp "go-test-server/internal/user/http"`
  - `userrepo "go-test-server/internal/user/repository"`
- 领域模型统一从 `go-test-server/internal/domain/models` 引用，不在各 domain 下重复定义实体。

---

## 2. 配置与启动

### 2.1 配置加载

- 统一通过 `config.LoadFromEnv()` 加载配置（内部根据 `APP_ENV` 选择 `configs/config.yaml` / `config.dev.yaml` / `config.prod.yaml`）。
- 配置结构定义在 `internal/config/config.go`，新增配置项时在 `Config` 或其子结构体中扩展，并在 `LoadConfig` 中赋默认值。
- 敏感信息（如 DB 密码、JWT Secret）通过环境变量或配置文件注入，禁止硬编码。

### 2.2 服务启动顺序（main 中建议顺序）

1. 加载配置：`config.LoadFromEnv()`
2. 初始化日志：`logger.Init()`
3. 初始化数据库（若该服务需要）：`database.Init(&cfg.Database)`
4. 创建路由：`routes.NewRouter(cfg)`
5. 注册全局中间件（如 RateLimit、RequestID）
6. 注册探针与指标：`SetupHealth`、`SetupReady*`、`SetupMetrics`
7. 注册业务路由：`SetupXxxRoutes(r, ...)`
8. 若启用 Consul：注册 HTTP 服务，并在退出时 `defer` 反注册
9. 启动 HTTP 服务：`serverutil.ServeWithGracefulShutdown(srv, shutdownTimeout)`

- 需要 DB 的服务使用 `SetupReadyWithDB(r, database.GetDB)`；仅网关等无 DB 服务使用 `SetupReadySimple`。

---

## 3. HTTP 层（Handler + 路由）

### 3.1 Handler 结构

- 每个业务域在 `internal/<domain>/http/` 下提供 `Handler` 结构体，通过构造函数注入 **Service** 与 **Config**（以及必要的 DB），例如：

```go
type Handler struct {
    userService    *service.UserService
    sysUserService *service.SysUserService
    cfg            *config.Config
}

func NewHandler(cfg *config.Config) *Handler {
    db := database.GetDB()
    return &Handler{
        userService:    service.NewUserService(db, cfg),
        sysUserService: service.NewSysUserService(db, cfg),
        cfg:            cfg,
    }
}
```

- Handler 方法签名统一为 `func (h *Handler) MethodName(c *gin.Context)`，只做参数绑定、调用 Service、写响应，不写复杂业务逻辑。

### 3.2 请求绑定与校验

- 使用 `c.ShouldBindJSON(&req)` 绑定 JSON 请求体；需要校验时在结构体上使用 `binding:"required"` 等 tag，或使用 `pkg/validator` 做额外校验。
- 绑定失败时统一返回 `errors.BadRequest("invalid request")` 或更具体说明，通过 `response.Error` / `response.ErrorApp` 写出。

### 3.3 统一响应格式

- **单条/创建成功**：`response.DataOK(c, data, msg)` / `response.DataCreated(c, data, msg)`（当前项目创建成功也用 200 + Data）。
- **列表**：`response.ListOK(c, rows, total, msg)`，保证前端得到 `{ code, rows, total, msg }`。
- **错误**：`response.Error(c, httpStatus, msg)` 或 `response.ErrorApp(c, appErr)`，其中 `appErr` 来自 `pkg/errors`（如 `errors.Unauthorized(...)`、`errors.Forbidden(...)`）。
- Handler 内不直接 `c.JSON(200, gin.H{...})` 写业务数据，统一走 `pkg/response`，便于前端统一解析。

### 3.4 路由注册

- 路由注册函数放在 `internal/api/routes/` 或各域的 `http/routes.go`，命名为 `SetupXxxRoutes(r *gin.Engine, ...)`。
- 需要鉴权的路由组先 `Use(middleware.AuthMiddleware(cfg))`；需要角色时再 `Use(middleware.RequireRoles("admin"))`。
- 路由路径约定：
  - 管理端/旧版：`/api/users`、`/api/auth` 等
  - C 端：`/api/user`、`/api/ledgers`、`/api/entries` 等
  - 探针：`/health`、`/ready`、`/metrics`（网关可对部分路径做限流豁免）

---

## 4. 业务逻辑层（Service）

### 4.1 Service 职责

- Service 层只依赖 **Repository**、**Config** 及必要的 **pkg**（如 `pkg/auth`），不直接依赖 `gin.Context`。
- 业务校验、事务边界、多表协调、调用外部能力（如 JWT 签发）均在 Service 内完成；Handler 只做“入参 → Service → 出参/错误”的转换。

### 4.2 构造函数与依赖

- 使用 `NewXxxService(db *gorm.DB, cfg *config.Config) *XxxService` 形式，在构造函数内初始化所需 Repository。
- 同一域内多个 Service 可互相依赖时，在 Handler 或上层组装，避免循环依赖。

### 4.3 错误与 HTTP 状态

- 业务错误返回 `error`；若需与 HTTP 状态码对应，在 Handler 中根据 `err` 类型或内容映射为 `errors.Unauthorized`、`errors.Conflict` 等，再 `response.ErrorApp(c, appErr)`。
- 避免在 Service 中直接写 HTTP 状态码；保持 Service 与传输层（HTTP）解耦。

---

## 5. 数据访问层（Repository）

### 5.1 职责与命名

- Repository 仅负责对 `internal/domain/models` 的 CRUD 与简单查询，方法命名清晰：`GetByID`、`GetByUsername`、`Create`、`Update`、`ListByXxx` 等。
- 使用 GORM，接收 `*gorm.DB` 注入；不在 Repository 内开启全局事务，事务由 Service 层 `db.Transaction(...)` 控制。

### 5.2 与 Model 的关系

- 所有持久化实体定义在 `internal/domain/models`，使用 GORM tag（如 `gorm:"primaryKey"`、`gorm:"size:50"`）；JSON 序列化用 `json:"xxx"`，敏感字段用 `json:"-"`。
- Repository 入参/出参使用 `*models.Xxx` 或 `models.Xxx`，不在此层再定义 DTO；若需与 API 解耦，可在 Service 或 Handler 层做转换。

---

## 6. 领域模型（domain/models）

- 一张表一个结构体，字段与表结构一致，并包含 `CreatedAt`、`UpdatedAt`（若使用 GORM 约定）。
- 关联关系用 GORM 的 `Preload`、`many2many` 等在 Repository 或 Service 中按需加载，避免在 model 中写业务逻辑。

---

## 7. 错误处理

### 7.1 使用 pkg/errors

- 业务与 HTTP 层统一使用 `pkg/errors` 的工厂方法：`BadRequest`、`Unauthorized`、`Forbidden`、`Conflict`、`Internal`、`ServiceUnavailable` 等。
- 返回给前端的错误格式由 `response.Error` / `response.ErrorApp` 统一为 `{ code, data, msg }`（code 为 HTTP 状态码数字）。

### 7.2 日志与错误链

- 关键错误使用 `logger` 打日志，并尽量保留错误链：`fmt.Errorf("...: %w", err)`，便于排查。

---

## 8. 鉴权与权限

### 8.1 JWT

- 使用 `pkg/auth` 签发与校验 JWT；配置来自 `cfg.JWT`（Secret、ExpiresIn）。
- 鉴权中间件从 Header `Authorization: Bearer <token>` 解析，校验通过后在上下文中写入 `user_id`、`username`、`roles`，供后续 Handler 和 RBAC 使用。

### 8.2 RBAC

- 需要角色限制的路由使用 `middleware.RequireRoles("admin", ...)`，与 `AuthMiddleware` 组合使用；未满足角色时返回 403 Forbidden。

---

## 9. 中间件与可观测性

- **RequestID**：全链路请求 ID，建议全局启用。
- **RateLimit**：网关等入口使用 `RateLimitWithSkips`，对 `/health`、`/ready`、`/metrics` 等做豁免。
- **AccessLog**：按需在路由上挂载访问日志中间件。
- **探针**：`/health`、`/ready`（有 DB 的服务用 DB Ping）、`/metrics`（Prometheus），与现有 `internal/api/routes` 中实现保持一致。

---

## 10. 网关与下游服务

- 网关通过 Consul 发现下游服务（或配置/环境回退）；下游服务在启动时根据 `cfg.Consul.Enabled` 决定是否注册。
- 网关转发时保持路径与 Method，并统一错误响应格式；可结合现有 `gateway_routes` 中的重试、熔断逻辑扩展。

---

## 11. 数据库与迁移

- 使用 GORM `AutoMigrate` 时，将新增 model 在 `internal/pkg/database` 的 `Init` 中一并迁移（或使用独立 migrations 脚本）。
- 生产环境建议使用独立迁移流程，避免在服务启动时自动改表。

---

## 12. 测试与代码质量

- 单元测试放在与被测代码同包的 `*_test.go` 中，或集中在 `internal/<domain>/service` 等包下。
- 公共逻辑（如 `pkg/auth`、`pkg/errors`）应有对应测试。
- 新增接口时，保持与现有 API 风格一致（路径、Method、请求/响应体、错误码），并在文档（如 `docs/api`）中更新说明。

---

## 13. 小结：检查清单

- [ ] 新服务入口在 `cmd/<service-name>/main.go`，启动顺序符合第 2 节。
- [ ] 新业务域在 `internal/<domain>/` 下包含 `http`、`service`、`repository`，模型在 `internal/domain/models`。
- [ ] Handler 只做绑定、调 Service、写响应；统一使用 `pkg/response` 与 `pkg/errors`。
- [ ] Service 不依赖 gin，错误通过 error 返回，由 Handler 映射为 AppError。
- [ ] Repository 仅做数据访问，使用 `*models.Xxx`，事务在 Service 层控制。
- [ ] 配置与敏感信息通过 config/env 注入；探针与指标按服务类型正确挂载。
- [ ] 鉴权与 RBAC 使用现有 middleware，路径与命名与现有项目一致。