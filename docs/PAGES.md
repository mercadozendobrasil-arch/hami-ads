# HAMI Ads V1 页面结构

## 1. 页面树

```text
/
├─ login
├─ register
├─ onboarding
│  ├─ company-settings
│  ├─ connect-platform
│  ├─ import-data
│  ├─ cost-setup
│  └─ data-quality
├─ dashboard
│  ├─ overview
│  ├─ profitability
│  ├─ ad-performance
│  ├─ recommendations
│  └─ alerts
├─ campaigns
│  ├─ index
│  └─ [campaignId]
├─ catalog
│  ├─ products
│  ├─ products/[sku]
│  ├─ abc
│  └─ stock-risk
├─ promotions
│  ├─ index
│  └─ [promotionId]
├─ reports
│  ├─ profitability
│  ├─ campaign-comparison
│  └─ exports
├─ integrations
│  ├─ index
│  ├─ [accountId]
│  └─ sync-runs
├─ automation
│  ├─ rules
│  ├─ rules/new
│  └─ action-runs
├─ settings
│  ├─ company
│  ├─ users-roles
│  ├─ roles
│  ├─ costs
│  ├─ notifications
│  └─ audit-log
└─ assistant
   └─ index
```

## 2. 第一阶段页面说明

### Dashboard Overview

首屏显示净销售额、广告花费、贡献利润、利润率、ACOS、TACOS、ROAS、库存风险和数据更新时间。所有数字均可点击下钻。

### Profitability

表格按 SKU 展示收入、COGS、平台费、运费、税费、广告费和贡献利润。支持日期/平台/店铺/分类/品牌筛选，标记成本缺失和数据延迟。

### Ad Performance

从 campaign → ad group → ad → SKU 四层下钻。提供趋势、预算、花费、曝光、点击、订单、销售额和利润贡献。指标旁显示计算口径。

### Recommendations

建议队列支持优先级、类型、范围、状态和证据查看。详情页必须提供“批准”“拒绝”“稍后处理”“添加备注”，并明确展示是否会产生平台写操作。

### Integrations

以卡片显示 Mercado Livre、Shopee、TikTok Ads 的连接状态、权限、最后同步、同步错误和重试入口。连接凭据只在平台授权弹窗中处理，HAMI 页面不显示 token。TikTok Ads 显示 advertiser account；TikTok Shop 若未来接入，必须单独显示为另一种连接类型。

### Catalog / SKU Detail

查看 SKU 价格、成本、库存、自然销售、广告销售、促销、Buy Box 和利润趋势。第一阶段使用静态/同步数据，建议可以引用此页作为证据。

### Settings

公司配置、内部用户、角色、成本配置、时区、币种、通知和审计日志。

## 3. 通用交互状态

每个数据页面必须有 loading、empty、partial、error、stale data 五种状态；同步延迟不应伪装成实时数据。

## 4. 导航与权限

| 页面域 | Viewer | Analyst | Operator | Admin |
|---|---:|---:|---:|---:|
| Dashboard / Reports | 查看 | 查看 | 查看 | 查看 |
| Recommendations | 查看 | 创建/评论 | 批准 | 全部 |
| Integrations | 查看状态 | 查看状态 | 重试同步 | 连接/断开 |
| Costs | 查看 | 编辑建议 | 编辑 | 编辑 |
| Users/Roles | 无 | 无 | 无 | 管理 |
| Action Runs | 查看 | 查看 | 执行已批准 | 全部 |

## 5. 视觉基线

- 主色强调利润状态：正向绿色、风险橙色、亏损红色；不能只用颜色表达
- 默认桌面优先，同时保证 1280px 宽度可用
- 表格列固定关键指标，细节放在展开面板
- 所有“自动化”控件显示当前模式：建议、需审批、已启用
