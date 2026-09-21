# HAMI Ads V1 平台接入说明

## 平台范围

| 平台 | V1 状态 | 接入对象 | 第一阶段数据 |
|---|---|---|---|
| Mercado Livre | 目标平台 | 店铺/广告账户 | 商品、订单、广告、促销、库存 |
| Shopee | 目标平台 | 店铺/广告账户 | 商品、订单、广告、促销、库存 |
| TikTok Ads | 目标平台 | TikTok for Business advertiser account | Campaign、Ad Group、Ad、预算、花费、报表指标 |
| Amazon | 不接入 | — | — |
| TikTok Shop | 预留，不作为 V1 默认连接 | Shop/Partner account | 后续单独评估商品、订单和 GMV Max |

## TikTok Ads 接入边界

HAMI Ads V1 以 TikTok Marketing API 为目标，首先实现：

1. advertiser account 授权和连接状态
2. campaign、ad group、ad 的只读同步
3. standard/integrated report 的同步或异步任务
4. spend、impressions、clicks、conversions、revenue 等可用指标的标准化
5. 通过 HAMI SKU 映射计算广告相关贡献利润

第一阶段不默认实现：

- 创建/编辑/暂停 TikTok campaign
- audience 上传与管理
- creative 上传、替换和删除
- TikTok Shop 商品/订单写操作
- 未经审批的自动化 Agent

TikTok 官方文档包含 campaign management、creative、audience 和 reporting 等能力；HAMI Ads 先从报表和只读同步开始，等权限、审核和指标口径确认后再开启写操作。

## 账户和凭据

- UI 只显示平台、advertiser/account ID、状态、授权范围和最近同步时间
- access token 使用 Secret Manager/KMS 保存；数据库只保存 `credential_ref`
- OAuth callback 必须校验 `state` 和 redirect URI
- 连接断开时撤销或删除 credential reference，并保留审计记录
- 所有平台错误统一转换为 `provider_code`、`retryable` 和用户可读说明

## 数据口径

TikTok Ads 的广告收入归因与 Mercado Livre/Shopee 的订单收入可能不是同一套归因模型。系统必须同时保留：

- 平台原始指标
- HAMI 标准化指标
- 归因窗口、时区、币种和数据更新时间

禁止把平台 revenue 直接当作净销售额；利润计算必须继续扣除 COGS、平台费、运费、税费、促销折扣和其他配置成本。
