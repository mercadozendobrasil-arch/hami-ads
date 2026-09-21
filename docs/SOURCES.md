# 公开参考与设计边界

## 公开参考

本设计参考以下 AdMan 公开页面，访问时间为 2026-09-21：

1. [AdMan 官方产品页](https://ad-man.io/en)：公开展示了 Mercado Livre、Amazon、Shopee、Tiny、Bling 接入，真实 SKU 利润、ACOS/TACOS、促销中心、目录自动化、广告预算/竞价管理、Agent 和每日摘要等方向。
2. [AdMan 帮助中心：Amanda IA](https://ajuda.ad-man.io/hc/centro-de-ajuda/articles/1782133931-amanda-ia)：公开展示了 campaign 诊断、利润分析、目录管理、促销比较、ABC 曲线、Buy Box、价格竞争、ACOS 控制、预算调整、暂停低效 campaign 和智能提醒等能力。
3. [TikTok API for Business](https://business-api.tiktok.com/gateway/docs/index?language=ENGLISH)：官方文档显示 Marketing API 覆盖 campaign 管理、Business Center、creative、audience 和 reporting；HAMI Ads V1 只采用 advertiser account 的标准报表和只读同步能力。
4. [TikTok for Developers：TikTok Shop 开发者更新](https://developers.tiktok.com/blog/tiktok-shop-developer-updates)：官方文档将 TikTok Shop 商品、订单和店铺能力作为独立 API/Partner Center 能力，因此本设计不把 TikTok Shop 与 TikTok Ads 混为一个连接器。

## 设计边界

- 以上仅用于竞品/公开功能分析，不代表 HAMI Ads 与 AdMan 存在合作或代码共享。
- HAMI Ads 是公司内部单实例系统，不设计多租户组织、租户切换或客户邀请流程。
- HAMI Ads V1 不接入 Amazon；平台范围为 Mercado Livre、Shopee、TikTok Ads。
- HAMI Ads 的数据库、API、页面和业务名词是独立设计。
- 不复制 AdMan 未公开的算法、提示词、内部接口、代码、客户数据或品牌资产。
- “真实利润”必须基于 HAMI Ads 自己接入的数据、成本配置和计算版本。
- 第一阶段默认 human-in-the-loop；只有经过明确授权和审批才允许触发第三方平台写操作。
