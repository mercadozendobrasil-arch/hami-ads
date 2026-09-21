# HAMI Ads V1 API 架构

## 1. 服务边界

```text
Dashboard
   │ HTTPS / JSON
   ▼
API Gateway / Go API
   ├─ Auth & RBAC
   ├─ Organization / Members
   ├─ Marketplace Accounts
   ├─ Catalog & Orders
   ├─ Ads & Promotions
   ├─ Profitability Engine
   ├─ Recommendations & Approvals
   └─ Reports / Exports
        │
        ├─ PostgreSQL  ── canonical normalized data
        ├─ Redis       ── cache, lock, rate limit, job queue
        └─ Workers     ── sync, calculate, recommend, digest
                         │
                         └─ Marketplace adapters
```

## 2. 基础约定

- Base URL：`/api/v1`
- JSON 字段：`camelCase`
- 分页：`limit` + `cursor`
- 列表统一返回 `{ data, nextCursor, hasMore }`
- 错误统一返回 `{ code, message, details, requestId }`
- 写操作支持 `Idempotency-Key`
- 每个请求带 `X-Request-ID`，日志不记录 token/cookie/password
- 所有外部平台调用经过 adapter，业务层不直接拼平台 API

## 3. 认证与组织

```text
POST   /auth/register
POST   /auth/login
POST   /auth/refresh
POST   /auth/logout
GET    /me
GET    /organizations
POST   /organizations
GET    /organizations/{organizationId}/members
POST   /organizations/{organizationId}/members/invitations
PATCH  /organizations/{organizationId}/members/{memberId}
```

权限在 middleware 中解析，handler 仍必须使用 organization-scoped service。

## 4. Marketplace 与同步

```text
GET    /marketplace-providers
GET    /marketplace-accounts
POST   /marketplace-accounts/{platform}/authorize
GET    /marketplace-accounts/{accountId}/oauth/callback
POST   /marketplace-accounts/{accountId}/disconnect
GET    /marketplace-accounts/{accountId}/sync-runs
POST   /marketplace-accounts/{accountId}/sync-runs
GET    /sync-runs/{syncRunId}
POST   /sync-runs/{syncRunId}/retry
```

同步 API 只创建任务并立即返回，不在 HTTP 请求内拉取 90 天数据。worker 使用锁和 idempotency key 防止重复同步。

## 5. Dashboard 与利润

```text
GET /dashboard/summary?from=&to=&accountId=
GET /dashboard/trends?metric=profit&granularity=day
GET /profitability/skus
GET /profitability/skus/{sku}
GET /profitability/abc
GET /profitability/stock-risks
GET /reports/profitability
POST /reports/exports
GET /reports/exports/{exportId}
```

响应必须包含：`dataFreshness`、`calculationVersion`、`currency` 和 `dataQuality`。

## 6. 广告与促销

```text
GET /campaigns
GET /campaigns/{campaignId}
GET /campaigns/{campaignId}/metrics
GET /ad-groups/{adGroupId}/ads
GET /ads/{adId}
GET /ads/{adId}/metrics
GET /promotions
GET /promotions/{promotionId}
```

V1 以上接口只读。平台写操作必须走 action workflow，不允许前端直接调用 marketplace adapter。

## 7. 建议、审批和执行

```text
GET   /recommendations
GET   /recommendations/{recommendationId}
POST  /recommendations/{recommendationId}/approve
POST  /recommendations/{recommendationId}/reject
POST  /recommendations/{recommendationId}/snooze
POST  /recommendations/{recommendationId}/execute
GET   /action-runs
GET   /action-runs/{actionRunId}
POST  /automation-rules
PATCH /automation-rules/{ruleId}
```

`execute` 必须检查：用户权限、建议状态、过期时间、feature flag、平台权限、幂等键和最新数据版本。执行失败不能自动无限重试。

## 8. Worker 与事件

推荐队列：

```text
sync.account.requested
sync.account.completed
sync.account.failed
profitability.recalculate.requested
recommendations.generate.requested
recommendation.approved
action.execute.requested
action.execute.completed
daily_digest.generate.requested
```

worker 规则：

1. 每个 job 带 organization/account scope。
2. 失败记录 error code 和可操作的 remediation。
3. 外部 API 使用平台级 rate limit 和退避。
4. 任务结果可重放，写入 `sync_jobs` / `action_runs`。

## 9. Adapter 接口

```go
type MarketplaceAdapter interface {
    Provider() string
    AuthorizeURL(ctx context.Context, state OAuthState) (string, error)
    ExchangeToken(ctx context.Context, code string) (Credential, error)
    SyncCatalog(ctx context.Context, account Account, cursor string) (CatalogPage, error)
    SyncOrders(ctx context.Context, account Account, window TimeWindow, cursor string) (OrderPage, error)
    SyncAds(ctx context.Context, account Account, window TimeWindow, cursor string) (AdsPage, error)
    SyncPromotions(ctx context.Context, account Account, window TimeWindow, cursor string) (PromotionPage, error)
    ExecuteAction(ctx context.Context, action ApprovedAction) (ActionResult, error)
}
```

第一阶段可将 `ExecuteAction` 实现为拒绝/feature flag disabled，从架构上预留但不开放危险写操作。

## 10. 观测与安全

- metrics：sync duration/success、API latency、recommendation count、action success
- tracing：request → job → adapter → action run
- health：数据库、Redis、worker、平台授权状态
- webhook 必须验签、去重、限时；原始请求存引用
- OAuth callback 校验 state 和 redirect URI
- 所有写操作进入 audit log
