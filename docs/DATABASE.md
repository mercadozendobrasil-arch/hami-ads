# HAMI Ads V1 数据库结构

目标数据库：PostgreSQL 16。

## 1. 通用约定

- 主键：`uuid`，默认 `gen_random_uuid()`
- 金额：`numeric(20,6)`，金额币种单独存储
- 时间：`timestamptz`，统一 UTC
- 单公司部署：不设置 `organization_id`、租户切换或 RLS；应用权限由用户角色和账户授权控制
- 状态：优先使用受控字符串或数据库 enum，避免自由文本
- 外部 ID：`platform` + `external_id` 联合唯一
- Token：只保存 `credential_ref`，真实密钥放在 Secret Manager/KMS
- 删除：业务数据默认 soft delete；原始同步记录不得无痕删除

## 2. 核心表

### `company_settings`

单行公司配置表，使用固定 `id=1`，保存公司名称、时区、默认币种、成本策略和系统状态。V1 不创建组织实体，也不提供租户切换。

### `users`

保存内部账户身份、角色和状态。角色包括 `admin`、`analyst`、`operator`、`viewer`；关键唯一约束为 `users.email`。

### `platform_accounts`

保存店铺/广告账户的非敏感元数据：

```text
id, platform, account_kind, external_account_id, display_name,
country_code, currency, status, credential_ref, scopes_json,
last_sync_at, last_error_code, last_error_message, created_at, updated_at
```

唯一约束：`(platform, external_account_id)`。

### `sync_jobs`

```text
id, platform_account_id, job_type, cursor,
window_from, window_to, status, attempt_count, idempotency_key,
started_at, finished_at, error_code, error_message, metrics_json
```

唯一约束：`(platform_account_id, job_type, idempotency_key)`。

## 3. 商品与商业数据

### `products` / `product_variants`

`products` 是公司级商品；`product_variants` 是 SKU 粒度，保存 `sku`、条码、名称、品牌、分类、成本策略和库存汇总。

唯一约束：`sku`。

### `listings`

平台商品/广告可投放对象，关联 `platform_account_id` 和 `product_variant_id`。保存 `external_listing_id`、标题、价格、buy_box 状态、listing_status 和 raw reference。

### `orders` / `order_items`

保存订单收入、币种、状态、平台费汇总和订单行。订单行必须关联 SKU，并记录数量、成交价、折扣、平台费、运费、税费和归因广告信息。

### `cost_snapshots`

记录 SKU 在某一有效期间的 COGS、包装成本、税率、固定成本分摊策略和来源（manual/imported/erp）。同一 SKU 同一时间段不能有两个 active 版本。

## 4. 广告数据

### `ad_campaigns`

```text
id, platform_account_id, external_id,
name, ad_type, status, objective, daily_budget, currency,
start_at, end_at, raw_data_ref, created_at, updated_at
```

### `ad_groups` / `ads`

`ad_groups` 关联 campaign；`ads` 关联 ad group、listing 和 SKU。三层都保存外部 ID 和平台状态。

### `ad_metrics_daily`

按广告对象和日期保存平台原始指标：

```text
id, platform_account_id, metric_date,
campaign_id, ad_group_id, ad_id, listing_id, product_variant_id,
impressions, clicks, spend, attributed_orders,
attributed_revenue, conversions, cpc, ctr, raw_json, source_updated_at
```

唯一约束：`(platform_account_id, metric_date, ad_id)`，没有 ad ID 时使用平台对象级 fallback key。

## 5. 促销、利润和建议

### `promotions`

保存促销类型、平台、时间窗口、折扣、SKU/listing 关联和状态。第一阶段只同步和分析，不执行创建/修改。

### `profitability_snapshots`

每天或每次重算生成 SKU/日期快照：

```text
id, snapshot_date, product_variant_id,
revenue, ad_spend, cogs, marketplace_fees, payment_fees,
shipping_cost, tax_cost, promotion_discount, contribution_profit,
margin_rate, acos, tacos, roas, stock_units, stock_days,
data_quality_status, calculation_version, created_at
```

### `recommendations`

```text
id, recommendation_type, priority, status,
scope_type, scope_id, title, rationale, evidence_json,
proposed_action_json, expected_impact_json, risk_json,
rule_version, expires_at, created_by, reviewed_by, reviewed_at,
executed_at, failure_reason, created_at, updated_at
```

建议内容必须可解释；`evidence_json` 只存数据引用和计算摘要，不存 secret。

### `automation_rules`

保存可选的业务规则草稿和审批配置。V1 默认 `mode=advisory`；只有 Admin 开启并配置执行权限后，才允许进入执行队列。

### `action_runs`

记录每次批准后的外部动作：`idempotency_key`、目标平台、请求摘要、前后状态、响应摘要、重试次数和审计关联。

## 6. 支撑表

- `alerts`：异常和通知，关联 recommendation 或 sync job
- `audit_logs`：谁在什么时间对什么对象做了什么操作
- `raw_import_batches`：批量导入和原始 payload 的对象存储引用
- `webhook_events`：外部事件去重、验签、处理状态
- `daily_digests`：日报生成版本、收件人和发送状态

## 7. 关键索引

```sql
create index idx_profit_date
  on profitability_snapshots (snapshot_date desc);

create index idx_ad_metrics_date
  on ad_metrics_daily (metric_date desc);

create index idx_recommendations_queue
  on recommendations (status, priority desc, created_at desc);

create index idx_sync_jobs_account_status
  on sync_jobs (platform_account_id, status, created_at desc);
```

## 8. 单公司权限与数据安全

V1 不启用多租户 RLS。API middleware 必须在每个请求中解析用户角色，并由 service 层检查其是否可以访问目标 platform account。后台同步 worker 只读取已配置账户，所有外部写操作通过 `action_runs` 和 `audit_logs` 留痕。
