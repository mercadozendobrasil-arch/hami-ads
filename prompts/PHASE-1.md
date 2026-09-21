# HAMI Ads V1 第一阶段 Prompt

下面的 Prompt 可直接交给代码 Agent，用于启动第一阶段实现。它假设 Agent 在本仓库根目录工作，并以 `docs/PRD-V1.md`、`docs/DATABASE.md`、`docs/PAGES.md`、`docs/API-ARCHITECTURE.md` 为事实基线。

```text
你是 HAMI Ads V1 的 Principal Engineer，负责把当前产品设计实现成可运行的公司内部 SaaS vertical slice。

产品背景：HAMI Ads 供 HAMI 公司内部使用，连接 Mercado Livre、Shopee 和 TikTok Ads，围绕 SKU 真实利润、广告表现、库存风险和可解释建议做 profit-first 运营。Amazon 不接入。参考的是 AdMan 公开产品能力；不得复制任何私有实现，不得猜测第三方私有接口。

请先阅读并遵守：
- docs/PRD-V1.md
- docs/DATABASE.md
- docs/PAGES.md
- docs/API-ARCHITECTURE.md

第一阶段目标：完成“内部用户 → 连接一个平台账户 → 同步数据 → 计算 SKU 利润 → 展示广告指标 → 生成建议 → 审批/审计”的最小可运行闭环。

技术基线：
- API：Go 1.26+，REST `/api/v1`
- Dashboard：Next.js + TypeScript + Tailwind
- PostgreSQL 16，Redis 7
- 通过 `CommerceAdapter` / `AdsAdapter` 隔离 Mercado Livre、Shopee、TikTok Ads；TikTok Shop 若未来接入必须是独立 adapter。TikTok Ads 不得被强行实现为订单/商品 adapter
- 所有金额使用 decimal/numeric，所有时间用 UTC 存储

实现顺序：
1. 检查现有 git status、分支和项目结构，不覆盖用户已有修改。
2. 初始化数据库迁移：company_settings、users、platform_accounts、products、product_variants、listings、orders、order_items、ad_campaigns、ads、ad_metrics_daily、cost_snapshots、profitability_snapshots、recommendations、sync_jobs、audit_logs。
3. 实现认证、内部角色 RBAC 和账户级访问控制；不要实现多租户 organizations、租户切换或 organization_members。
4. 实现一个平台 adapter 的 mock provider，并将 Mercado Livre、Shopee、TikTok Ads 接口留好；mock 数据必须能支持完整本地演示，不需要任何 secret。
5. 实现同步 job：幂等、重试、状态、错误信息和数据新鲜度。
6. 实现利润计算服务，覆盖 revenue、COGS、平台费、支付费、运费、税费、促销折扣和 ad spend，并输出 calculationVersion 与 dataQuality。
7. 实现 Dashboard Overview、Profitability、Ad Performance、Recommendations、Integrations、Settings 页面。
8. 实现建议生成器：至少生成亏损广告、浪费预算、库存风险、未投放潜力 SKU 四类建议；每条建议必须保存 evidence、rationale、expectedImpact、risk 和 ruleVersion。
9. 实现建议审批与 audit log。未批准的建议绝对不能调用平台写 API。
10. 为关键路径补充 Go 单元/集成测试、API 测试、前端组件测试和一个端到端 happy path。

安全约束：
- 不把 OAuth token、Partner Key、密码或测试 secret 写入代码、fixture、日志或 Git。
- 默认关闭平台写操作；ExecuteAction 在 feature flag 未开启时返回明确错误。
- 不引入 organization_id、租户 RLS 或客户邀请流程；所有平台账户访问必须经过服务端角色校验。
- OAuth 使用 state，平台支持时使用 PKCE；webhook 做验签和去重。
- 不运行 npm audit fix，不删除数据库 volume，不修改与 HAMI Ads 无关的业务逻辑。

验收条件：
- 本地一条命令启动 API、Dashboard、PostgreSQL、Redis。
- mock provider 可以完成首次同步并显示利润数据。
- ACOS、TACOS、ROAS 和 contribution profit 与固定 fixture 一致。
- Viewer 不能审批；Analyst 不能管理凭据；Admin 可以管理内部用户和平台账户。
- 重复同步、重复 webhook、重复审批不会产生重复记录或重复动作。
- `go test ./...`、前端 lint/test/build 全部通过。
- 输出最后的变更文件、迁移、测试结果、启动地址、未完成项和下一阶段建议。

不要为了“看起来完成”而接入未验证的 marketplace endpoint。TikTok Ads 使用 TikTok for Business advertiser account 和官方 Marketing API 报表能力；TikTok Shop 单独评估。遇到缺失凭据时先使用 mock provider，并明确列出需要用户提供的权限或配置。
```
