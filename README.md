# HAMI Ads V1

面向多平台电商卖家的广告与利润运营 SaaS 产品设计。

HAMI Ads 参考 AdMan 公开展示的产品方向，聚焦：

- Mercado Livre、Amazon、Shopee 等 marketplace 账户统一接入
- 按 SKU 计算真实利润、贡献利润、ACOS、TACOS、ROAS
- 广告、促销、目录竞争力和库存风险的统一看板
- 面向利润目标的优化建议、人工审批和审计
- 后续可扩展为受控的自动化 Agent，而不是一开始就直接修改广告

本仓库当前是 V1 第一阶段的产品设计与工程 Prompt，不包含任何第三方平台 secret，也不声称复制 AdMan 的私有实现。

## 文档

- [V1 产品需求文档](docs/PRD-V1.md)
- [数据库结构](docs/DATABASE.md)
- [页面结构](docs/PAGES.md)
- [API 架构](docs/API-ARCHITECTURE.md)
- [第一阶段实现 Prompt](prompts/PHASE-1.md)
- [公开参考与设计边界](docs/SOURCES.md)

## V1 第一阶段边界

第一阶段交付一个可运行的“利润驾驶舱”：完成组织、用户、marketplace 账户、商品/SKU、广告数据、订单与成本数据的接入和标准化，提供利润分析、广告诊断、建议队列和审批流。

默认只读同步。任何暂停广告、修改预算、修改竞价、创建促销等写操作都必须经过明确审批，并保留幂等键、操作者和审计记录。

## 建议技术基线

- API：Go 1.26+，REST `/api/v1`
- Dashboard：Next.js、TypeScript、Tailwind、`next-intl`
- 数据：PostgreSQL 16、Redis 7
- 异步任务：Redis-backed job queue；生产环境可替换为队列服务
- 观测：structured logs、metrics、job run audit

## 设计原则

1. Profit-first：利润是主指标，ROAS/ACOS 只是诊断指标。
2. Human-in-the-loop：建议可以自动生成，外部写操作必须审批。
3. Tenant isolation：所有业务数据带 `organization_id`，并在数据库层准备 RLS。
4. Explainability：每条建议说明数据依据、计算公式、预期影响和风险。
5. Credential safety：OAuth token 只保存加密引用，不进 Git、不写日志。
