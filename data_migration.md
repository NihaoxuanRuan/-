# 数据盘点 & 任务清单
二十年期数据迁移工程，由聚宽 + Qlib转移到通联 + Qlib，目标是维持现有格式数据不变并增加应有的新字段。每日自动落至本地。

个股范围：沪深全股。
时间要求：2006至今。

---

## 一、日级字段（CSV 共 33 列）

| 字段（CSV 列） | 来源 API | 进 .bin? | 备注 |
|---|---|---|---|
| 代码 / 简称 / 日期 | get_all_securities / — | 简称否，日期=日历 | 简称是 ST 识别唯一来源 |
| 前收 / 开 / 高 / 低 / 收 | get_price `fq='none'` | open/high/low/close ✓，**pre_close ✗** | 不复权 |
| 涨跌(元) / 涨跌幅(%) | 派生(close − pre_close) | ✗ | 仅在 CSV |
| 成交量 / 成交额 / 均价 | get_price | ✓（均价 → vwap） | Alpha360 用 vwap |
| 换手率 | get_valuation | ✓ turnover_rate | |
| 总股本 / 流通股本 | get_valuation | **✗（进 staging 但不 dump）** | 见下方提醒 |
| 总市值 / 流通市值 | get_valuation | ✓ market_cap / float_market_cap | 亿元 → 元 |
| 市盈率 / 市净率 / 市销率 / 市现率 | get_valuation | ✓ | |
| 涨停价 / 跌停价 / 停牌 | get_price 直给 | ✓ | 限板逻辑依赖 |
| 营收同比 / 净利同比 / ROE / 毛利 | get_fundamentals | ✓ | 季频采样 + ffill |
| 融资余额 / 融资买入额 | get_mtss | ✓ | 非两融保持 NaN（不 ffill） |
| 所属行业 / 所属概念 | get_industry / get_concept | ✗（非数值） | 仅申万一级 + 概念串（`\|` 分隔） |

**提醒**
- `daily_ret` 只在 `price_matrices.pkl`，不落 CSV。
- 单位对齐（通联侧必核对）：市值 → 元、股本 → 股（见 `VALUATION_MAP` 乘数）。

---

## 二、分钟级字段（12 字段，1m，按天 parquet）

字段：`open, close, high, low, volume, money, avg, pre_close, factor, high_limit, low_limit, paused` + `code / time`

- 本地存储位置：`MinuteData/stock/` 和 `MinuteData/index/`，以每日为单位形成一个 `.parquet` 文件进行存储。

---

## 三、指数（INDEXES 共 7 个；932000 注释未启用）

| JQ → Qlib | 名称 | 用途 | 日 .bin | 分钟 | benchmark CSV | 成分股 CSV |
|---|---|---|---|---|---|---|
| 399303 → SZ399303 | 国证2000 | **主基准** | ✓ | ✓ | ✓ gz2000 | ✓ |
| 000852 → SH000852 | 中证1000 | MERA 基准 | ✓ | ✓ | ✓ csi1000 | ✓ |
| 000300 → SH000300 | 沪深300 | 大盘对比 | ✓ | ✓ | ✓ hs300 | ✓ |
| 000905 → SH000905 | 中证500 | 中盘 / regime / marketfeat | ✓ | ✓ | ✓ csi500 | ✓ |
| 000903 → SH000903 | 中证100 | regime | ✓ | ✓ | ✗ | ✗ |
| 000906 → SH000906 | 中证800 | regime | ✓ | ✓ | ✗ | ✗ |
| 000985 → SH000985 | 中证全指 | regime / marketfeat | ✓ | ✓ | ✗ | ✗ |


---

## 四、还需要哪些（数据缺口）

### 高（需求）
- [ ] **指数成分权重**：现只有成分**名单**、无权重 需要补充。

### 中（可以补充）
- [ ] 北向 / 陆股通持股（未拉）
- [ ] 资金流主力净额（JQ Pro 不含被跳过，通联本地若有可补）
- [ ] 业绩预告 / 快报（现只有定期财报 indicator，事件型缺）
- [ ] 行业到申万二级 / 中信（现仅 sw_l1，细分轮动 + 中性化粒度不足）
- [ ] 股息率 / EV 类（估值仅 PE/PB/PS/PCF）

---

## 五、转通联任务清单（最后两步为qlib框架转换，暂缓）

- [ ] 确认本地服务器接入方式：SDK / REST / 直连 DB / 落地 parquet（决定 `tl_sync.py` 怎么写）
- [ ] 日级：上表的 30 字段映射 → 复用 `_build_matrices` + `_export_csv` 落 CSV（核查单位 / 不复权 / code 格式）
- [ ] 涨停跌停 / 停牌：确认通联直给，否则按板块 10/20/30% 派生 + tradeStatus 推 paused
- [ ] qualified_mask：确认通联有 ST 历史；无则定一个起算日（早期回测会缺）
- [ ] 行业 / 概念：通联口径映射（注意非申万、概念 ≠ 同款）
- [ ] 指数：7 个日线 + 成分**（带权重）** + benchmark CSV
- [ ] 分钟：通联分钟接入 → 同结构 parquet（12 字段 / 1m / 按天 / stock + index）
- [ ] 跑 `qlib_converter`（不改）→ .bin + `market_features_v6.pkl`
- [ ] 等价校验：小样本 diff JQ vs 通联同 `(code, date)`，过了再全量切

---

## 附：你可能需要的的契约文件依赖（迁移需逐字节验证是否对齐的部分）

- `Dataset2006JQ/MarketData/{code6}.{SH/SZ}.CSV`（GBK，data_loader 读）
- `Dataset2006JQ/price_matrices.pkl`
- `Dataset2006JQ/qualified_mask.pkl`（Qlib 代码格式）
- `Dataset2006JQ/industry_map.pkl` / `concept_map.pkl`
- `Dataset2006JQ/alpha191/alpha191_{date}.pkl`
- `Dataset2006JQ/benchmark_*.csv` / `cons_*.csv`
- `qlib_data/cn_jq/`（calendars / instruments / features/<code>/*.day.bin）
- `Dataset2006JQ/MinuteData/`（分钟 parquet）
