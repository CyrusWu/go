# AXS/Ronin 生态监控程序 MVP 规格

## 目标
用最少的数据源，持续捕捉 AXS 这一轮上涨是否“真增长”还是“投机泡沫”，并在风险窗口提前预警。

## 监控对象与输出

### 监控对象（4 层）
1. **链层（Ronin）**：链上是否真的变热（tx/active/gas/合约交互）
2. **筹码层（AXS/RON）**：是否发生集中转入交易所、鲸鱼异动、持币结构变化
3. **交易层（CEX/衍生品）**：成交量、OI、资金费率、长短手结构是否失衡
4. **社区层（X/Reddit/Discord）**：bAXS 机制是否引发大规模负反馈/套利教程扩散

### 输出（3 个）
- **Daily Snapshot（每日快照）**：一屏看懂今日健康度（0~100 分）
- **Alert（预警）**：只推“值得你动作”的事件（非噪音）
- **Weekly Trend（7 天趋势）**：判断“趋势改善/恶化/假热度”

## MVP 必须抓的指标（最小可用集）

### Ronin 链热度（每小时）
| 指标 | 说明 | 采集方式 | 频率 |
| --- | --- | --- | --- |
| ronin_tx_count_24h | 24h 交易笔数 | Ronin Stats / Explorer API | 每小时 |
| ronin_active_wallet_24h | 24h 活跃钱包 | 同上 | 每小时 |
| ronin_gas_spent_24h | 24h gas 总消耗 | 同上 | 每小时 |

目标：解决“AXS 涨，但链有没有变热”。

### AXS 筹码与风险
| 指标 | 说明 | 采集方式 | 频率 |
| --- | --- | --- | --- |
| axs_price, axs_volume_24h | 价格 & 成交量 | CoinGecko API | 每 15 分钟 |
| axs_exchange_netflow | 交易所净流入/出 | CryptoQuant/Glassnode（没有权限先跳过） | 每日/每小时 |
| axs_whale_transfer_to_cex | 鲸鱼转入 CEX 事件 | 监听大额转账 + 地址标签 | 每 10 分钟 |
| axs_holders_count | 持币地址数 | Etherscan / Ronin Explorer | 每日 |

没有 CryptoQuant 就用“鲸鱼转入已标记 CEX 地址”做替代。

### 衍生品情绪（每小时）
| 指标 | 说明 | 数据源 | 频率 |
| --- | --- | --- | --- |
| axs_open_interest | 未平仓量 OI | CoinGlass API | 每小时 |
| axs_funding_rate | 资金费率 | CoinGlass API | 每小时 |
| axs_long_short_ratio | 多空比 | CoinGlass API | 每小时 |

目标：识别“高 OI + 资金费率异常”导致的踩踏风险。

### 社区情绪（轻量版）
| 指标 | 说明 | 数据源 | 频率 |
| --- | --- | --- | --- |
| keyword_mentions_24h | 关键词提及数 | X / Reddit 搜索 | 每小时 |
| negative_ratio | 负面情绪比例（粗略） | 规则 + 小模型 | 每小时 |
| tutorial_risk_flag | 套现/绕过/刷分教程 | 关键词规则 | 每小时 |

**关键词包（初始）**
- bAXS：bAXS, bonded AXS, Axie Score fee, redeem bAXS, cash out bAXS
- 负面：scam, rug, too high fee, quit, dead game, unfair
- 套利：exploit, bypass, farm, bot, tutorial, how to

## 系统架构（逻辑）
Collector（采集） → Normalizer（清洗标准化） → Metrics DB（存储） → Analyzer（计算/规则） → Alert Engine（告警） → Report Generator（日/周报） → UI/Telegram/Email

### 技术栈建议（低成本）
- **后端**：Python（FastAPI）或 Node.js（NestJS）
- **任务调度**：Celery + Redis / BullMQ
- **存储**：Postgres（指标表） + Redis（缓存）
- **可视化**：Next.js + ECharts（简单仪表盘）
- **通知**：Telegram Bot（最顺手）

## 数据模型（表结构）

### metrics_raw（原始抓取）
- id
- source（coingecko / coinglass / ronin_stats / etherscan / x / reddit）
- metric_key（如 axs_price）
- value_number
- value_json（原始 payload）
- ts（时间戳）

### metrics_agg（聚合后）
- metric_key
- ts_bucket（1h / 1d）
- value（聚合值）
- delta_1d（对比昨日）
- delta_7d（对比 7 天均线）
- zscore_30d（异常检测）

### alerts（预警事件）
- alert_id
- level（P0/P1/P2）
- category（Onchain / Whale / Derivatives / Social）
- title
- description
- trigger_rule_id
- evidence_json
- ts
- status（open/acked/closed）

### daily_report
- date
- health_score（0~100）
- summary（核心结论）
- key_metrics_json
- alerts_json

## 告警规则（MVP <= 10 条）

### P0（强风险）
1. **鲸鱼砸盘预警**
   - 条件：whale_transfer_to_cex >= 500k AXS（24h 累计）
   - 且：axs_price 当日涨幅 > 10%
   - 输出：P0 “高位派发风险”
2. **衍生品踩踏预警**
   - 条件：OI 7 日新高
   - 且：funding_rate < 0（空头主导）
   - 且：price 波动扩大（1h 振幅 > 3%）
   - 输出：P0 “结构脆弱，可能连环清算”
3. **链下炒作 vs 链上冷清**
   - 条件：axs_volume_24h 3 日均值 ↑
   - 且：ronin_active_wallet_24h 7 日均值 ↓ 或 持平
   - 输出：P0 “上涨缺乏链上确认”

### P1（趋势恶化/转弱）
4. **用户热度拐头**
   - 条件：ronin_active_wallet_24h 7 日均值连续 3 天下降
   - 输出：P1 “生态转冷”
5. **新增持币者停滞**
   - 条件：axs_holders_count 7 天增长 < 1%
   - 且：价格继续上涨
   - 输出：P1 “筹码集中+投机主导”
6. **社区集中负反馈**
   - 条件：negative_ratio > 35% 且 mentions_24h 放大
   - 输出：P1 “机制被质疑发酵”

### P2（信息提示）
7. **X 热度爆发**
   - 条件：mentions 24h 环比 > 200%
   - 输出：P2 “情绪升温（可用作顺风）”
8. **Ronin 链活动突然上升**
   - 条件：tx_count_24h & active_wallet_24h 同比 > 50%
   - 输出：P2 “链上确认增强”

## 健康度评分（0~100）

四大维度，每维 25 分：
1. **链上热度**：active_wallet、tx、gas 的 7 日趋势
2. **筹码健康**：holders 增速、鲸鱼转 CEX 频率
3. **市场结构**：OI、funding、long/short 是否过热
4. **社区信心**：mentions、negative_ratio、套利教程标记

最终输出：
- **80~100**：强健康（趋势确认）
- **60~80**：可持有（需盯风险）
- **40~60**：偏投机（随时回撤）
- **<40**：危险期（准备撤退）

## Telegram 推送（优先）

每日固定 1 条：
- 今日评分 + 关键变化 + 是否触发 P0/P1

示例：
- Health: 58/100
- P0: 链下放量 + 链上不热（风险）
- Whale: +320k AXS 转入 Binance
- Derivatives: OI 新高、funding 转负
- Action: 减仓观察 / 等链上确认

出现 P0/P1 立即推送预警。

## 实施提醒
1. **地址标签维护是重点**：没有 Nansen/Arkham 就先做一版手工 CEX 地址列表（Binance / Coinbase / OKX / Upbit 热钱包）。
2. **社交情绪先用规则**：关键词 + 简单情绪词典 + “教程标记”即可。
3. **告警必须可追溯**：所有 alert 写入 evidence_json（原始链接/交易 hash/截图 URL）。

## 下一步（可交易信号版本）
如果需要“可交易决策系统”，追加输出：
1. 趋势确认型信号（允许加仓）
2. 结构脆弱型信号（必须减仓）

如需升级，可在评分/阈值/动作映射表上扩展。
