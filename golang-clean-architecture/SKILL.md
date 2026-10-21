---
name: golang-clean-architecture
description: >-
  Go 项目整洁架构（Clean Architecture）实践指南。涵盖四环分层规则、依赖方向判定、
  domain/application/adapter/infra 职责划分、组合根（bootstrap）编排、
  Go 接口放置策略、过度抽象识别与回退、context 传递/派生/脱离取消原则。
  适用：新建或重构 Go 服务的分层结构、审查依赖方向、判断某个 import 是否合规、
  迁移 DDD 分层 / 洋葱架构到整洁架构、审查 context 与日志透传。
  关键词：clean architecture, 整洁架构, onion, hexagonal, 六边形, 端口适配器,
  ports and adapters, 依赖反转, 分层, layer, ring, domain, application,
  adapter, infra, infrastructure, bootstrap, composition root, 组合根,
  context, WithoutCancel, DetachTimeout, 生命周期 ctx。
---

# Go 整洁架构实践指南

## 四环模型

```
┌─────────────────────────────────────┐
│  Ring 4: Infrastructure             │  config, otel, log, db driver, redis driver
│  ┌─────────────────────────────┐    │
│  │  Ring 3: Adapters           │    │  http handler, repo impl, inference client
│  │  ┌─────────────────────┐    │    │
│  │  │  Ring 2: Application │    │    │  use case, pipeline, orchestration
│  │  │  ┌──────────────┐   │    │    │
│  │  │  │ Ring 1: Domain│   │    │    │  entity, value object, domain service, port
│  │  │  └──────────────┘   │    │    │
│  │  └─────────────────────┘    │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
```

**唯一规则：所有依赖指向圆心。**

## Go 目录映射

```
internal/
├── domain/            # Ring 1 — 实体、值对象、领域服务、端口接口
│   ├── repository/    #   端口（接口）定义
│   ├── recommendation/#   纯业务规则（无外部依赖的函数）
│   └── types/         #   跨上下文共享的值对象
├── application/       # Ring 2 — 用例编排（pipeline node, filter, service）
│   └── recommendation/
├── adapter/           # Ring 3 (inbound adapter) — HTTP handler, cron, mq 等
│   └── http/
├── infra/             # Ring 3~4 (outbound adapter + driver)
│   ├── storage/redis/ #   Ring 3: 实现 domain 端口的适配器
│   ├── inference/     #   Ring 3: 外部推理服务适配器
│   ├── abtest/        #   Ring 3: AB 测试适配器
│   ├── config/        #   Ring 4: 配置加载
│   ├── otel/          #   Ring 4: 可观测性
│   ├── log/           #   Ring 4: 日志
│   └── http/          #   Ring 4: HTTP server / router
└── bootstrap/         # 组合根 — 唯一允许 import 所有层的地方
```

## 依赖规则速查

| 源层 → 目标层 | 合法？ | 说明 |
|---------------|--------|------|
| domain → (任何其他层) | **禁止** | domain 只依赖 stdlib 和外部纯库 |
| application → domain | **合法** | 用例通过端口接口调用外部能力 |
| application → infra | **禁止** | 这是最核心的规则 |
| adapter ↔ infra | **合法** | 同属外环，handler 可以用 infra 的 config/otel/具体类型 |
| infra → domain | **合法** | 适配器实现端口接口 |
| bootstrap → 所有层 | **合法** | 组合根负责接线 |

### 与 DDD 分层的关键区别

DDD 分层把 adapter 和 infra 视为不同层，禁止互引。
**整洁架构把它们视为同环**，可以互引——不需要为 handler 使用 config/otel/具体类型额外抽象接口。

## 接口放置策略

### 原则：接口由消费者定义

```go
// ✅ 正确：接口定义在 domain（被 application 消费）
// domain/repository/pool.go
type PoolItemReader interface {
    GetPoolItems(ctx context.Context, poolKey string, start, stop int64, scoreDesc bool) ([]string, error)
}

// infra/storage/redis/pool_reader.go — 实现
type RedisPoolItemReader struct { client *redis.Client }
func (r *RedisPoolItemReader) GetPoolItems(...) ([]string, error) { ... }
```

```go
// ✅ 正确：application 层的 node 只依赖接口
// application/recommendation/homepage_pool_recall.go
type homepagePoolLevelSource struct {
    reader repository.PoolItemReader  // 不是 *redis.Client
}
```

### 什么时候不需要接口

外环（Ring 3~4）之间互用时，直接用具体类型：

```go
// ✅ handler 直接用 infra 类型（同环合法）
type InfoHandler struct {
    cfg         *infraconfig.Config
    dbManager   *infradatabase.Manager
    bloomMerger *storageredis.BloomMerger
}
```

```go
// ❌ 过度抽象：为外环互用专门定义 domain 接口
type DatabaseHealthChecker interface {  // 不该在 domain
    PingDatabase(ctx context.Context, name string) bool
}
```

### 判断清单

需要接口吗？问自己：

1. **消费者在 application 或 domain 层？** → 需要，接口放 domain
2. **消费者在 adapter 层，依赖的是 infra？** → 不需要，直接用具体类型
3. **是为了测试 mock？** → 在测试文件内定义局部接口即可
4. **有多个实现需要运行时切换？** → 需要，接口放消费者所在层

## 组合根（Bootstrap）编排

组合根是唯一允许 import 所有层的地方。职责：

1. 创建所有基础设施资源（Redis 客户端、DB、AB SDK）
2. 构造 domain 接口的实现实例
3. 注入到 application 层的 `Deps` 结构体
4. 返回 cleanup 函数供 main defer

```go
// bootstrap/infra.go
type Infra struct {
    Deps           Deps                        // application 层需要的接口集
    DBManager      *infradatabase.Manager       // handler 直接需要的 infra 类型
    DiagnosticRepo repository.DiagnosticRepository
    BloomMerger    *infrastorageredis.BloomMerger
}

func BuildInfra(ctx context.Context, cfg *infraconfig.Config, rc *redis.Client) (*Infra, func(), error)
```

```go
// main.go — 纯编排
func main() {
    cfg := mustLoadConfig(...)
    rc := mustConnectRedis(...)
    infra, cleanup, _ := bootstrap.BuildInfra(ctx, cfg, rc)
    defer cleanup()
    svc := bootstrap.BuildRecommendationService(ctx, cfg, infra.Deps)
    // wire HTTP handlers, start server
}
```

## 过度抽象的信号

遇到以下情况时，可能做过头了：

| 信号 | 处理 |
|------|------|
| 接口只有一个实现且消费者在外环 | 去掉接口，直接用具体类型 |
| 为了传 config 给 handler 而创建了映射 struct | 直接传 `*Config` |
| 为外环之间传值创建了 domain 层类型 | 用 infra 层自己的类型 |
| 适配器只是转发调用无额外逻辑 | 评估是否真的需要这层间接 |

## 业务规则提取

纯业务逻辑（不依赖外部资源）应在 domain 层：

```go
// domain/recommendation/rules.go
func IsNewUser(totalDays, threshold int) bool { return totalDays <= threshold }
func ComputeRates(f map[string]float64)       { /* 纯计算 */ }
```

application 层调用这些函数，自己不重复实现：

```go
// application/recommendation/checkin_node.go
if domainrec.IsNewUser(stats.TotalDays, n.NewUserThreshold) { ... }
```

## Context 使用原则

一句话：**能传就传；要限时就派生；父 ctx 已死还要收尾时，脱离取消但必须设超时。**

Context 不属于某一环的专属概念，但贯穿 bootstrap → adapter → application → infra。
分层合规不等于 context 用对；审查时两者一起看。

### 三类用法

| 场景 | 做法 | 说明 |
|------|------|------|
| **传递** | 原样往下传当前 `ctx` | 请求链路、用例调用、仓储/客户端、`slog.*Context` |
| **派生** | `WithTimeout` / `WithCancel` / `WithDeadline` | 需要独立超时、局部取消时，从上游派生，不要另起无关的 `Background` |
| **脱离取消** | `WithoutCancel` + **明确超时** | 生命周期 ctx 已取消后仍要做的收尾（关连接、释放租约、退出日志） |

```go
// ✅ 传递：请求 ctx 一路带到应用层与日志
func (h *Handler) Verify(w http.ResponseWriter, r *http.Request) {
    result, err := h.uc.Verify(r.Context(), cmd)
    // ...
}

// ✅ 派生：对外部调用限时
func (r *Repo) Find(ctx context.Context, id string) error {
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()
    return r.db.WithContext(ctx).First(...).Error
}

// ✅ 脱离取消 + 超时：shutdown 后 cleanup 不被 context.Canceled 打断，也不会卡死
func cleanup(lifecycleCtx context.Context) {
    closeCtx, cancel := context.WithTimeout(context.WithoutCancel(lifecycleCtx), 5*time.Second)
    defer cancel()
    _ = db.Close()
    logger.ErrorContext(closeCtx, "db close done")
}
```

### 不要混用的边界

| 对象 | 该用哪个 ctx | 原因 |
|------|--------------|------|
| HTTP/gRPC 业务处理、业务日志 | **request ctx**（`r.Context()`） | 带 trace / user / 客户端断开取消 |
| main / bootstrap 启动日志、选主、注册 | **lifecycle ctx**（信号通知的根 ctx） | 进程级，不是某次请求 |
| `cleanup` / `app.Start` 返回后的退出日志 | **脱离取消 + 超时** | lifecycle 通常已 cancelled |
| readiness / liveness 探针 | **独立短超时 Background**（或等价） | 不要绑 lifecycle，避免停机时探针因 cancel 误抖 |
| 后台 worker 主循环 | 从 lifecycle **派生** | 停机可取消；单次任务再套超时 |

```go
// ❌ 启动日志丢掉 lifecycle ctx
logger.Info("channel registered", "channel", code)

// ✅
logger.InfoContext(ctx, "channel registered", "channel", code)

// ❌ cleanup 仍用已取消的 lifecycle ctx 做 I/O
func cleanup() { redis.Ping(lifecycleCtx) } // 常直接 context.Canceled

// ❌ 脱离取消却不设超时 —— 外部依赖卡住会拖死进程退出
closeCtx := context.WithoutCancel(lifecycleCtx)
```

### 与分层的关系

- **domain / application**：方法签名带 `ctx context.Context`，只传递或按需派生；不创建进程级根 ctx。
- **adapter**：从 `r.Context()` / 消息消费 ctx 传入 application；异常路径日志必须 `*Context`。
- **bootstrap / main**：持有 lifecycle ctx；启动用它；收尾用脱离取消 + 超时。
- **禁止**：application 为了“方便”自己 `context.Background()` 发起外部调用（丢失取消与 trace）。

## 审查步骤

对已有项目做整洁架构合规审查时：

1. **扫描 domain 层 import** — 不应出现 `internal/application`、`internal/infra`、`internal/adapter`
2. **扫描 application 层 import** — 不应出现 `internal/infra`（核心规则）
3. **adapter 和 infra 互引** — 在整洁架构下合法，不必修
4. **识别 application 层的 infra 泄漏** — `net/http` 客户端调用、直接 `*redis.Client` 等
5. **检查 domain 层是否有不应存在的接口** — 仅为外环服务的接口不属于 domain
6. **扫描无 Context 的日志** — 请求/启动路径避免 `logger.Info/Error`（无 ctx）；应用 `InfoContext/ErrorContext`
7. **扫描 cleanup / shutdown** — 收尾是否 `WithoutCancel` + timeout，避免已取消父 ctx 或无限等待
8. **扫描错误的 Background** — 业务链路中的 `context.Background()` 多为漏传；探针/纯 init 除外并应注释原因

```bash
# 快速验证核心规则
rg 'internal/infra' internal/domain/      # 应为 0
rg 'internal/(infra|adapter|bootstrap)' internal/application/  # 应为 0

# Context / 日志透传（按项目路径调整）
rg 'logger\.(Info|Warn|Error)\(' --glob '*.go'   # 业务路径应优先 *Context
rg 'context\.Background\(\)' --glob '*.go'      # 业务链路命中需说明理由
```
