# HAMI Ads V1 产品需求文档

版本：V1.0 / 第一阶段

状态：Draft for implementation

## 1. 产品定位

HAMI Ads 是面向跨平台电商卖家的 profit-first 广告运营 SaaS。它把 marketplace 广告、订单、商品成本、平台费、运费、促销和库存放在同一个业务模型中，回答三个问题：

1. 哪些广告真正赚钱？
2. 哪些 SKU 正在消耗预算但没有贡献利润？
3. 下一步应该暂停、扩量、调预算，还是先修正成本/价格？

AdMan 公开页面强调的能力包括统一连接 Mercado Livre、Amazon、Shopee 与 ERP，持续计算 SKU 真实利润，分析 ACOS/TACOS/ROAS，管理广告/促销/目录，并由 Amanda 提供建议和受控执行。本 PRD 将这些公开能力重新定义为 HAMI Ads 的独立 V1 范围。

## 2. 用户与痛点

### 目标用户

- 多 marketplace 经营的品牌卖家
- 管理多个店铺的电商运营负责人
- 为多个客户服务的广告代理商/顾问
- 需要从“投放指标”转向“投放后净利润”的财务或老板

### 核心痛点

- 每个平台的广告数据、订单、费率和库存分散
- ROAS 很高但扣除成本、税费、运费后实际亏损
- 广告优化依赖人工导出表格，反应慢且不可审计
- 商品成本、促销、Buy Box 和库存风险没有进入投放决策
- 自动化规则容易误伤高价值商品，缺少边界和审批

## 3. 产品目标

### V1 必须达成

- 连接至少一个 marketplace 账户并完成历史数据同步
- 以 SKU 为粒度展示销售额、广告花费、费用、成本、贡献利润和利润率
- 展示 campaign/ad/product 的 ACOS、TACOS、ROAS、CTR、CPC、转化率
- 识别至少四类问题：亏损广告、浪费预算、库存风险、未投放但有潜力的 SKU
- 生成有依据的优化建议，支持批准、拒绝、延后和备注
- 所有数据同步、建议和人工操作可追踪

### V1 不做

- 不在第一阶段实现无审批的自动调价、自动改竞价或自动暂停
- 不做完整会计总账、发票和支付结算
- 不承诺覆盖所有 marketplace 的每一个广告类型
- 不把 AI 聊天包装成没有数据依据的“万能助手”
- 不把第三方平台 token、Partner Key 或店铺密码提交到 Git

## 4. 功能范围与优先级

### P0：第一阶段必须交付

#### 4.1 组织与权限

- 创建组织、成员邀请、角色：Owner、Admin、Analyst、Operator、Viewer
- 按组织隔离数据
- 成员只能访问被授权的 marketplace 账户和店铺
- 记录登录、授权、审批、同步和平台写操作审计

#### 4.2 Marketplace 连接

- 连接/断开 Mercado Livre、Amazon、Shopee
- OAuth 或平台授权流程使用 state、PKCE（平台支持时）和加密 token 引用
- 记录连接状态、最近同步时间、错误原因和权限范围
- 第一阶段至少实现一个真实 adapter；其他平台使用相同接口保留扩展位

#### 4.3 数据同步与标准化

- 初始同步最近 90 天的商品、订单、广告 campaign/ad、促销和库存数据
- 增量同步按平台 cursor/time window 执行
- 任务具备幂等性、重试、退避、暂停和错误可见性
- 原始 payload 保存在受控 raw 表或对象存储引用，标准化表供产品查询

#### 4.4 利润驾驶舱

- 总销售额、广告花费、平台费、运费、税费、COGS、贡献利润、贡献利润率
- 按组织、店铺、平台、商品、SKU、campaign、日期筛选
- 真实利润计算必须显示数据新鲜度和缺失成本提示
- 提供 SKU 贡献利润排序、亏损 SKU、广告花费占比和库存天数

#### 4.5 广告管理只读视图

- campaign、ad group、ad、listing 层级
- 状态、预算、花费、曝光、点击、订单、销售额、ACOS、ROAS、TACOS
- 近 7/14/30/90 天对比
- 显示平台原始指标和 HAMI 计算指标，避免口径混淆

#### 4.6 建议与审批

- 建议类型：降低预算、增加预算、暂停广告、提升竞价、降低竞价、检查成本、库存预警
- 每条建议包含规则/模型版本、证据时间范围、计算值、目标、预期影响和风险
- 状态：pending、approved、rejected、snoozed、executed、failed、expired
- V1 可批准但默认不执行平台写操作；执行器由 feature flag 控制

#### 4.7 基础报表

- 利润日报/周报
- campaign 对比
- SKU ABC 曲线
- 广告浪费清单
- CSV 导出

### P1：第二阶段候选

- 促销中心和优惠券利润护栏
- Buy Box/价格竞争和目录自动化
- 自定义业务规则和 Agent
- Amanda 风格的基于数据上下文的问答助手
- 预算跨 campaign 重分配建议
- 多店铺批量操作

## 5. 关键业务定义

### 5.1 贡献利润

```text
contribution_profit =
  net_revenue
  - cogs
  - marketplace_fees
  - payment_fees
  - shipping_cost
  - tax_cost
  - promotion_discount
  - ad_spend
```

缺少关键成本时不得静默记为 0；界面必须标注 `incomplete_cost_data`，并允许管理员配置默认成本策略。

### 5.2 ACOS / TACOS / ROAS

```text
ACOS  = ad_spend / attributed_ad_revenue
TACOS = ad_spend / total_revenue
ROAS  = attributed_ad_revenue / ad_spend
```

分母为 0 时返回 null，并显示“暂无可计算数据”，不得返回 Infinity。

### 5.3 库存天数

```text
stock_days = available_units / max(avg_daily_units_sold, small_positive_number)
```

广告建议必须同时检查库存天数，避免放大缺货或积压风险。

## 6. 核心用户流程

### 流程 A：首次接入

注册 → 创建组织 → 选择 marketplace → 授权 → 校验权限 → 同步 90 天数据 → 成本配置 → 数据质量检查 → 进入 Dashboard。

### 流程 B：发现亏损广告

Dashboard → 广告浪费卡片 → campaign 明细 → SKU 利润拆解 → 查看建议依据 → 批准或拒绝 → 记录审计。

### 流程 C：每日运营

查看昨日利润 → 查看风险/异常 → 处理建议 → 查看执行结果 → 导出日报。

## 7. 非功能要求

- 多租户查询必须带组织过滤；关键表预留 RLS policy
- API 默认超时、限流、分页和幂等键
- 外部平台失败不应阻塞 Dashboard 读取已有数据
- 同步失败需在 5 分钟内可见，并提供重试入口
- 金额用 decimal/numeric，不用浮点数持久化
- 所有时刻存 UTC，展示时按组织时区转换
- API secret 不进入日志、错误响应、截图或仓库
- 建议生成过程可回放，保留输入数据版本和计算版本

## 8. V1 验收标准

1. 新组织在没有平台数据时能完成 onboarding，并看到空状态和连接引导。
2. 至少一个 marketplace 连接成功后，90 天数据在后台任务中完成同步。
3. 同一个 SKU 的收入、成本、广告费和利润可以钻取到明细来源。
4. Dashboard 的 ACOS/TACOS/ROAS 与测试 fixture 计算结果一致。
5. 任何建议都能回答“为什么建议、依据是什么、会影响什么”。
6. 未批准的建议不会调用平台写 API。
7. 重复 webhook、重复同步和重复审批不会产生重复业务数据。
8. Viewer 无法执行审批，Analyst 无法变更连接凭据。
9. 关键路径有 API、数据库和前端测试。

## 9. 产品指标

- 首次连接成功率
- 首次数据可用时间（TTFD）
- 有成本数据覆盖的 SKU 比例
- 建议查看率、批准率、执行成功率
- 亏损广告识别准确率与人工复核通过率
- 建议后的 7 天贡献利润变化
- 同步成功率、延迟和平台错误率
