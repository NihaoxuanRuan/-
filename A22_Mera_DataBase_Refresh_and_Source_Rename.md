# A 股端到端深度学习量化策略 — 项目知识库 v-A22
# 数据底座本阶段刷新 + etl_RN 上游脚本改名（对 Mera 的影响;模型/信号零改）
# （完整承载 A21 全部正文）

---

## 0_A22. A22 阶段更新导读(数据底座本阶段刷新 + 上游脚本改名)★★★【重点】

> 本节是 A22 顶层导读。下方完整保留 A21 及更早全部正文,A22 改动以 `★ A22` / 【A22】标注,过时陈述用 ~~划线~~ + 修正。**本阶段对 Mera 的影响集中在本节与下方"A21 增量"的就地修正 —— 阅读时重点关注"上游脚本改名"与"必跑一次重建"。**
> **A22 不动 mera 模型层(训练 / 推理 / ckpt / 预训练 / 信号结论),只记录一次主线侧数据底座变动对 Mera 的连带影响 —— 主线把日频 `.bin` 增量刷新到当前(`cn_tl` 日历 4969 交易日),并把 `etl_RN/` 数据脚本改名为 `daily_*` / `minute_*` 双族。** 这些都是"数据源换轨/改名",与 A20 及更早的信号/模型结论正交。

**A22 阶段一句话总结**:① **上游脚本改名** —— Mera 重建 `market_features_v6.pkl` 依赖的指数源脚本 `index_full.py` 已改名 `daily_index_full_tl.py`,行情源脚本 `stock_full.py`(INNER JOIN 那条)已改名 `daily_stock_full_tl.py`;重建脚本本身 `rebuild_market_features_tl.py` **不改名**,但其内部/文档引用同步更新。② **必跑一次重建** —— 主线本期已把 `cn_tl` `.bin` 增量刷新到当前(日历 4969 交易日、末日接最新交易日),若 Mera 要在最新窗口训练/回测,须在**全新进程**重跑一次 `python etl_RN/rebuild_market_features_tl.py` 拾取刷新后的数据,否则用到的是旧 `market_features_v6.pkl`。③ **窗口口径补正** —— `cn_tl` 日历实测 4969 交易日(2006–2026 全市场并集),但**非两融小盘个票仍被主查询 INNER JOIN 截到 2014 起**(主线 §七十八 LEFT JOIN 待办未结);Mera 的 csi1000 universe 多为较新成分,实际训练/回测起点**仍建议 ≥ 2014** 直到该项修复。

**A22 待办【重点】**:
1. **【若上线最新窗口】重跑 `market_features_v6.pkl`**:全新进程 `python etl_RN/rebuild_market_features_tl.py`;依赖的 3 个 market 指数(`SZ399303` / `SH000905` / `SH000985`)须在 `daily_index_full_tl.py` 的 7 指数清单内(`399303/000852/000300/000905/000903/000906/000985`,已确认含)。
2. **【承主线待办】LEFT JOIN 修复后回拉起点**:主线把 `daily_stock_full_tl.py` 两处 INNER JOIN 改 LEFT JOIN、补回 2006–2014 后,Mera 训练/回测起点可前移;在此之前维持 ≥ 2014。
3. **【对照验证】** `QLIB_REGION` cn_jq vs cn_tl 同配置对照 pred_score / 回测指标同量级(两边窗口都取 2014+ 才可比)。

---

**最后更新**:2026-04-29(A19 阶段:MERA 端 ST/IPO 过滤死代码清理 + 隐式继承主策略合规矩阵)

**覆盖阶段**:Phase A1-A19(MERA v5/v6 / Path 2 RSAP-DFM 借鉴 / Path 3 Hierarchical MoE / Path 4 UMI RankIC loss / 融合版三件套首战 + 基准对齐 csi1000 + 参数区分层整合 / A17 `qlib_pipeline_mera_v6.py` 真重复抽公因子 + 长函数瘦身 / A18:`mera_pretrain_v6.py` 实质实现(13 维输入版,A11/A16/A17 三轮列为首要任务后终于落地) + Pipeline `_load_pretrained_encoder_into` 真实加载逻辑 + 文件命名规范统一(`mera_*` 起头)+ 输出目录归一(`Output/Mera_Output`)/ **★ A19:`find_st_stocks` 死代码清理 + 隐式继承主策略 `qualified_mask` 合规过滤(无需 plumbing)**)

**自包含说明**:本文件为可移植的项目知识库,保留所有数字、表格、代码片段、配置默认值、文件路径、失败实验数据、修复步骤。所有模型训练 / 回测 / 数据 / 工程方法论的关键决策点均完整收录,不依赖任何外部历史文档即可上手该项目。**A19 阶段的所有改动均以 `★ A19` 标注;前阶段的过时陈述用 ~~划线~~ + 修正格式标注**。

**A19 阶段一句话总结**:主策略侧重构 ST/IPO 过滤为基于 `qualified_mask.pkl` 的合规矩阵后,MERA 端两个文件 `qlib_pipeline_mera_v6.py` 和 `mera_single_window_diag_v6.py` 的 `find_st_stocks()` 调用被识别为**死代码**(`_ = find_st_stocks()` 调用后立即丢弃返回值,无副作用),原因是 MERA 走中证1000/csi1000 成分股 universe(`_filter_universe` 按当日成分股过滤),指数自身规则已隐式排除 ST 股。两个文件各删 2 处:`from qlib_pipeline_native import find_st_stocks` 删,`_ = find_st_stocks()` 这一行删。MERA 端**完全不直接读 `qualified_mask.pkl`**,但回测端调 `run_qlib_backtest` → `HBTopKStrategy.generate_trade_decision()` 内部已经有 `get_qualified_universe(pred_start_time)` 的合规过滤兜底,即使中证1000 在某次再平衡日还没把刚被 ST 的股剔出去,mask 兜住。MERA 端享用合规过滤但零 plumbing。`run_qlib_backtest` 签名同步:`listing_dates=None` 形参已删除,MERA 调用本来就没传过这个参数,无需修改调用点。文件命名规范不变。

**A18 阶段一句话总结**:`mera_pretrain_v6.py` 从无到有实质实现,与 v5 旧版 `pretrain_mera.py` 完全独立并存(命名规范"非管线 mera 资产以 `mera_` 起头"),输入对齐 v6 [60, 13],`SimilarProjector` 输入维度 360 维(v6 [M3] 检索 bank 设计选择,非 780),`PretrainDataset` 切出 6 量价字段产生 `flat_x_price [N, 360]`,`FeaturePreprocessor` 完整搬过来含两阶段(逐字段非线性变换 + 行业中性化 + 全局 z-score),checkpoint 存 `Output/Mera_Output/mera_v6_pretrain/pretrain_encoder_v6_window_NN.pt` 含 `version=6` / 完整 `config` / 完整 `preprocessor.to_state()`。Pipeline 端 `_load_pretrained_encoder_into` 由 A17 的"警告 stub"升级为真实加载,带四重校验:文件存在性 / `version==6` / `input_size==13` 与 `retrieval_flat_dim==360` / 关键 key 必须真的被加载(防 strict=False 静默全 missing 的隐藏 bug,检查 `input_bn.weight` `input_proj.weight` `encoder.layers.0.self_attn.in_proj_weight` `similar_proj.0.weight` 四个不在 missing 里)。`USE_PRETRAINED_ENCODER` 默认 True 不再被强制覆盖。`PRETRAIN_DIR_V6 = OUTPUT_DIR / "mera_v6_pretrain"` 新常量。第一次落地时由"OUTPUT_DIR 老路径 `Output/Qlib_DL` 残留 → cache 重生成 36GB + ckpt 找不到"的工程教训沉淀为方法论 §15.24"两脚本配套时 OUTPUT_DIR 必须共享同一常量"。文件命名规范明确:除 `qlib_pipeline_mera_*` 管线本体外,所有 mera 资产以 `mera_` 起头,与 ensemble 系列对应。

---

## 0_A20. A20 阶段更新导读(MERA 信号再归因)★★★

> 本节是 A20 顶层导读。下方完整保留 A19 全部正文,A20 阶段改动以 `★ A20` 标注,过时陈述用 ~~划线~~ + 修正。**A20 不动 mera 模型层(训练 / 推理 / ckpt / 预训练),只新增"主线 KB 66 对 mera 信号的取证级再归因"——这些发现来自主线对 v3_24f 实验 B(mera 当因子)80 格数据的复核,直接修正了主线 KB 65 对 mera 的三个论断,并把 mera 的战略用法从"因子"收紧为"短×长 horizon ensemble"。** 新增独立章节 §十六。

**A20 阶段一句话总结**:主线复核 v3_24f(=v3_23f + mera_score)80 格逐年数据后发现:(1)mera_score 的 XGB feature_importance 排名第一**不是 alpha 证据**——它与实际 OOS 改善**负相关**(corr 全期 −0.18 / 2026 −0.33),且完全由 label period 决定(period 5→7.1% / 30→16.4%),是训练 importance ≠ 泛化 importance 的过拟合形态;(2)mera 是 **short-horizon 信号**——在 period≤10 的配置上系统性加分(14 个全期+2026 双赢格里 12 个 period≤10),在 period 15/20 长 horizon 配置上系统性减分,且 0/80 格出现"全期差但 2026 好"(排除 2026 单年运气);(3)mera 在它适配的短 period 上有**大幅全期收益增益**(p10/.10/k30 在 baseline 口径全期 Sharpe 仅 0.914,叠 mera 后累计超额从 120% 拉到 236%,+116pp 近翻倍),而它的 2026 增量几乎全负(中位 −4.4pp,p10 仅 +0.2pp,说明 mera 不是 2026 题材)。**结论**:mera 当因子注入到长 period 主策略配置是减分(用法错,非信号没价值);正确用法是**短 period mera 子模型 × 长 period baseline 主模型的 ensemble**(horizon 互补、相关性低);ensemble 的价值在低相关而非子策略单独多强,因此**v6 不需先提质到 v7 才能用**——先做零成本正交化诊断 + ensemble 验证,v7 殿后。这修正了 A19 §十二.5"v7 重构值得投入、严格按优先级"的隐含排序(v7 仍可做,但顺序上 ensemble-first)。

**A20 对 A19 的修正点**:
- A19 §十二.5 实施优先级"★ 严格按顺序,不可跳过、第 1 步路径 4(信号增强)"→ ★ A20 重排:在投入 v7 信号增强前,先用 v6 现状做 ensemble 验证(零成本正交化诊断 → 短×长 horizon ensemble)。理由:主线数据显示 v6 短 period 价值未被榨干,且 ensemble 不要求子策略单独强、只要求低相关。v7 路径不变,但从"立即第一步"降为"ensemble 验证后条件触发"。
- A19 §战略定位"做成独立、高质量、低相关性的信号源融进 Ensemble"→ ★ A20:此定位现在有主线数据背书——mera 的 short-horizon 性质 + 与长 period 主策略的低相关,正是 ensemble 该利用的结构;不是泛泛的"信号源",而是明确的"短 horizon 子策略"。

---

## 一、项目定位与当前状态

### 1.1 项目目标

构建一套端到端深度学习 A 股量化选股策略。以 MERA(WWW 2025)为模型核心,从原始量价数据(Alpha360)出发,因子发现和收益预测均由模型完成,regime 适应是 MoE 架构的内生能力。

管线结构:

- **ETL 管线**:`python -m etl` 入口,JQData 数据获取 → pkl + 逐股 CSV → 自动调 `qlib_converter`(staging + .bin + 市场特征 pkl 一步到底)
- **★ MERA 管线**:`qlib_pipeline_mera_v6.py` 为当前研究入口。13 维输入(Alpha360 量价 6 维 + 基本面/两融 7 维)→ MASTER gate(30 维市场向量驱动)→ BatchNorm → Transformer backbone → 检索增强(仅 360 维量价 bank)→ **★ A14: 可选 stock-hidden regime extractor φ_DM + FiLM 调制 MoE gate(USE_RSAP_DFM=True)** → MoE 收益预测 → HBTopKStrategy 组合构建 → Qlib native 回测。v5 保留不动,用于对比
- **★ A18 MERA 自监督预训练管线**:`mera_pretrain_v6.py` 为预训练入口。SimCLR 双视图增强对比 + cross-space 360↔[60,13] align 双任务联合训练。每个 expanding window 一个 ckpt,写入 `Output/Mera_Output/mera_v6_pretrain/pretrain_encoder_v6_window_NN.pt`。先于 pipeline 跑一次,pipeline 在 `USE_PRETRAINED_ENCODER=True` 时自动按 window 加载。命名规范:除管线本体 `qlib_pipeline_mera_v6.py` 外,所有 mera 配套资产一律以 `mera_` 起头,与融合版 `ensemble_*.py` 系列对应
- **★ 单窗口诊断管线(v6)**:`mera_single_window_diag_v6.py` 为参数实验入口。**★ A14: 内置回测模块(RUN_BACKTEST=True 默认开),复用 pipeline 的 `run_qlib_backtest` + `save_basic_outputs`,产出 daily_detail.csv / risk_analysis.csv / backtest_nav.png 五面板图 + 整体 Sharpe/年化/回撤/超额/换手 + 分年分解,backtest stats 直接入 diag_summary.json**
- **★ A14 新增 HB Sweep 工具**:`hb_sweep_v2.py` 调用 pipeline 的 `run_qlib_backtest` 扫描 HB ∈ {0.00, 0.02, 0.05, 0.08, 0.12, 0.20, 0.30},输出 `hb_sweep_v2_summary.csv` + 分年分解 + `hb_sweep_v2_full.json`。**v1 版本 hb_sweep.py 自写简化回测(close-to-close 替 open-to-open + turnover 算错)与 pipeline 口径不一致已废弃**
- **Qlib 交易管线(XGB)**:`python qlib_trading.py` 入口
- **MERA 交易管线**:`qlib_trading_mera.py` 入口(v5,A7 新增)—— 主策略目前仍用 XGB/LGB,MERA 交易集成为后续工作
- **执行方式**:`gm_strategy.py` 在掘金终端运行,每日 09:30 自动读取 `target_holdings.csv`,双重日期校验

### 1.2 核心原则(精简后)

**Alpha360 是标准量价输入格式。** 6 字段(open/close/high/low/vwap/volume)× 60 天窗口展开 = 360 维。零公式计算,模型自动从原始量价中学习特征。

**异构特征扩展是第二阶段的必要升级(A12)。** 纯量价 Alpha360 信号在 2023-2024 系统性失效。v6 在量价 6 维之外额外引入 7 维基本面/两融:log_mv、pe_signed_log、pb_signed_log、roe、turnover_log、mtss_stock_rel、mtss_flow_rel。所有带 `Ref(d+1)` 滞后,无前瞻。

**Regime 适应应是选股模型的内生结构,不是外挂预测器。** MERA 的 MoE GateNet 动态路由让不同 expert 自动适配不同市场模式。历史实验已证明外挂 regime 预测器在 OOS 无效。

**市场状态作为门控条件是合理的内生 regime 信号(A12)。** MASTER(AAAI 2024)把全市场统计量(收益率/波动率/成交量/涨跌天数占比)编码为 30 维向量,通过 `α = 2·softmax(W · m)` 转换为通道级 gating 系数。13 维输入在进入 BatchNorm 之前先被 α 重缩放(`x̃ = α ⊙ x`)。α 是 end-to-end 学出来的 attention,不依赖任何人工 regime 标签。

**Broadcast 全局标量因子无法直接用于截面选股。** Market 向量恰好是这种 broadcast 标量,但它不直接进入 prediction head,而是作为 gate 系数影响特征重要性分配。

**Qlib 管线使用 .bin 格式。** 通过 `etl/qlib_converter.py` 从 JQ CSV 转换。

~~**回测与交易过滤逻辑必须一致。** IPO 过滤和 ST 过滤作为共享函数在回测和交易模块中统一调用。~~ **★ A19 修正(继承自主策略 Phase 53)**:回测与交易的合规过滤不再走两套共享函数 `find_st_stocks` + `filter_ipo`,统一改用基于 `qualified_mask.pkl` 的合规矩阵。MERA 端无 plumbing 改动,自动通过 `run_qlib_backtest → HBTopKStrategy → get_qualified_universe` 链路享用。

~~**动态过滤优于静态过滤。** 上市日期过滤每天动态判断(排除 pred_date 前不满 1 年上市的股票)。~~ **★ A19 修正**:动态过滤原则不变,但实现机制从"每天动态计算 listing_date - pred_date"改为"预先生成 `qualified_mask.pkl`(date×code bool 矩阵),策略每天读 mask 当日行"。语义更宽:同时排除 ST 历史(过去 365 天有过 ST)+ IPO 不满 1 年 + 退市股。

**不同基准下的超额不可直接比较。** 比较策略能力应看绝对收益或 Sharpe。

**模拟交易不输出脚本文件,直接读 CSV。** 掘金策略直读 `target_holdings.csv`。

**Pipeline BACKTEST_END 不能是 Qlib calendar 的最后一天。** `_resolve_end()` 返回 today-3 天。

**前瞻偏差在评估中极易隐藏。** 用 T 日收盘数据决定 T 日交易 = 前瞻。

**Sharpe 计算口径必须全局统一。** 使用几何年化 Sharpe = (cum^(252/n) - 1) / (std × √252)。

**基准切换后所有历史指标不可比。**

**★ 训练 Universe 必须匹配论文实验规模。** MERA 及其他顶刊在 CSI300/500/1000 上做实验,不是全市场 5000 只。

**BankCollator 把 bank tensors 从 Dataset 中分离(A7)。** Dataset 只存轻量索引数据(seq_x + neighbor_indices + y)。

**Qlib instrument 代码格式是大写前缀。** `D.features()` 返回的 MultiIndex instrument 是 `SH600000` / `SZ000001`(大写)。frozenset 查找是大小写敏感的。

**torch.compile / torch._dynamo 在 Windows 上不可用。** 必须在 `import torch` 之前设 `os.environ["TORCHDYNAMO_DISABLE"] = "1"` 彻底禁用。

**★★ 所有脚本必须支持 PyCharm 直接 Run。** 所有可配置参数一律在 `.py` 文件顶部参数区硬编码。不允许引入 argparse 必选参数、yaml 配置文件、`os.environ` 读取、bash wrapper 脚本、`RUN_*` / `USE_CLI_ARGS` 等双轨配置。`from __future__ import annotations` 必须紧跟 docstring 之后、参数区之前(Python 语法要求)。

**★★ 诊断脚本应直接复用 pipeline 模块,不重写模型/数据代码(A11)。** 诊断脚本 `import qlib_pipeline_mera_v6 as _pipe`,顶部参数区改完后通过 `setattr(_pipe, name, value)` 同步,然后调用 `_pipe.X` 里的函数。坏处:参数同步要维护 `_PARAMS_TO_SYNC` 列表,漏一个就会用 pipeline 默认值。

**★★ Expert "分化"有多个层次,不能只看 gate usage 均值(A11)。** 6 项指标:(1) gate usage 均值,(2) gate entropy ratio,(3) gate variance,(4) expert output cosine,(5) expert weight L2,(6) per-sample top-1 分布 + active experts。

**★★ 异构特征不应污染检索方向(A12)。** v6 bank 只存 360 维量价,基本面/两融只参与模型 forward 不参与检索。

**★★ ETL 是市场特征的唯一真相源(A12)。** `market_features_v6.pkl` 由 ETL 产出,pipeline 只读不写。

**★★ A13 — train/val 必须按 datetime 时序切,不能按行序切。** Qlib `D.features()` 返回的 MultiIndex 默认是 `(instrument, datetime)`,instrument 在外层。如果对底层 numpy array 做 `slice(0, 80%)` / `slice(80%, 100%)` 的行序切分,实际上切的是"前 80% 股票 vs 后 20% 股票"(cross-sectional split),train/val 时间完全重叠。这样的 val IC 反映跨股票泛化而非时序外推能力,v6 实测虚高到 0.37 vs OOS 0.02(17 倍差距)。正确做法是按 datetime 切,最后 VALID_RATIO 比例的 unique 交易日进 val,其余进 train。返回 `np.ndarray` 索引而非 `slice`。

**★★ A13 — 新加任何输入字段后必须验证 scale 分布。** 检查清单:对每维度打印 Stage 0(从 Qlib .bin 读到的原始值)、Stage A(Python 层变换后)、Stage B(全局 z-score clip 后)的 p01 / p50 / p99 / NaN 占比,以及 Stage B 后的 clip 触发率。异常信号:(1) std 被极端值撑大到 >10(典型病征:volume 被 IPO 复牌首日 ratio=16900 撑到 std=200),(2) p99 和 p50 之间跨度极小,(3) clip 触发率 >5%。A13 用这套方法找到三个潜伏 bug:volume 无 log 压缩、roe 无 signed_log 压缩、turnover_log 公式 ×100 导致过度压缩。

**★★ A13 — MASTER gate 的市场特征必须用 rolling z-score 不能用 expanding。** expanding z-score(`t` 时刻用 `[0..t]` 累积 mean/std)随时间推移分母越稳定,z-score 幅度单调衰减。A12 v6 实测训练期 α 的跨样本 std=0.045,OOS 测试期 σ=0.006(衰减 7-8 倍)。修复:改 rolling 252 日(1 年交易日),min_periods=60,clip ±5σ。

**★★ A13 — FeaturePreprocessor.transform 在 pipeline 内必须改 in-place。** 单次调用产生 2× 输入体积的临时数组。多出来的 5-8 GB RAM 在 128 GB 环境下直接触发 Windows Memory Compression。修法:`transform(..., inplace=True)` 时 reshape 用 view、z-score 用 `np.subtract(out=)` `np.divide(out=)` `np.clip(out=)` 三件套原地操作。调用方承担"raw 值不再需要"的语义责任。

**★★ A13 — 数据对齐验证是定位前瞻泄漏的黄金标准,但不是所有数据问题都表现为前瞻。** A13 用手算 label + 三源对齐(JQ CSV / Qlib .bin / Python 手算变换)证实 v6 的 13 维输入**没有前瞻偏差**。但 val IC 0.37 vs OOS IC 0.02 的 17 倍差距另有原因(split 按股票切)。后续要继续查:(1) train/val split 的时序性,(2) Feature 预处理的信号丢失,(3) Gate 输入的时间一致性。

**★★★ A14+ — Path 2 是"启发式借鉴"不是"论文复现"。** RSAP-DFM(IJCAI 2024)原文包含 6 个核心组件,本项目 Path 2 仅借用 (b) "从 stock hidden 反推 regime 向量 z" 概念和 (f) "FGSM 对抗扰动"概念,其余完全跳过(没有 DFM 方程、没有 dual encoder、没有 factor VAE、没有 bilevel)。FGSM 扰动对象从原版的 `λ_prior` 换成 `gate_in`,语义是**类比不是同构**。RSAP-DFM **无开源代码**(awesome-time-series-forecasting 等多个列表均只标 "Paper" 无 "Code"),实现全凭论文描述重建,与原文可能 20-30% 偏差。

**★★★ A14+ — Hybrid loss 是零架构改动的首选增益路径。** UMI(KDD 2025)的 RankIC loss 公式可以无视 backbone 架构叠加。实现成本 ~50 行代码。关键约束:(1) rank 变换对 target 做且 stop_grad,(2) pearson 计算必须在 FP32 autocast=False 内,(3) val loss 保留纯 MSE+aux,(4) 截面最少样本阈值 10。

**★★★ A14+ — 改 Dataset __getitem__ 签名必须全局追踪所有消费者,包括间接消费者(collate_fn)。** Dataset 的 tuple size 从 4 → 5(加 date_id)后,直接消费者是三个 for-loop 解包,但**间接消费者是自定义 collate_fn 的 `zip(*batch)`**。num_workers>0 时 collate_fn 在子进程里跑,traceback 穿过 `_worker_loop` 初看不像 Dataset 问题。改 signature 前必须 `grep -n "zip(\*batch)\|collate_fn\|__call__"` 找全间接消费者。

**★★★ A14+ — FGSM 数学在 AMP 环境下必须 FP32 隔离,绝不能靠 `scaler.scale` 或 `nan_to_num` 补救。** AMP 的 `GradScaler` 只对**参数梯度**(FP32 master copy)做缩放,不对中间节点梯度生效。在 autocast block 内用 `torch.autograd.grad(loss, intermediate_node)` 拿到的梯度是 FP16,若放大到 scaler 默认的 65536 倍会直接 overflow(FP16 max ~65504)。真正的坑在**梯度归一化这一步**:`g.norm(dim=-1)` 本质是 `sqrt(sum(x²))`,FP16 下 `sum(x²)` 对接近 FP16 上限的有限值会 overflow 成 `+Inf`(不是 NaN,所以 `nan_to_num` 兜不住)。唯一正确的根治方案:整段 FGSM 数学(`g.detach().float() → nan_to_num → clamp(±1e4) → norm → 扰动 → final nan_to_num`)全放在 `with torch.amp.autocast("cuda", enabled=False)` 里强制 FP32,FP32 动态范围 ~1e38 不会溢出。然后让 autocast 自动把 FP32 的 `gate_in_adv` 在后续 MoE 内部 matmul 时降级 FP16。

**★★★ A18 — 预训练脚本与 pipeline 必须共享同一个 OUTPUT_DIR 常量值,不能各写各的。** A18 第一次落地时 `mera_pretrain_v6.py` 误用了 v5 时代的老路径 `OUTPUT_DIR = Output/Qlib_DL`,而 pipeline `qlib_pipeline_mera_v6.py` 用的是 A12 后规范的 `Output/Mera_Output`。后果:(1) pretrain 去 `Qlib_DL/alpha_cache_v6/` 找不到 cache → 重新 6 分钟 + 36GB 写盘生成一份重复 cache;(2) ckpt 落到 `Qlib_DL/mera_v6_pretrain/` → pipeline 在 `Mera_Output/mera_v6_pretrain/` 找不到 → `FileNotFoundError`。两个脚本本来共享同一份数据 + 同一个 ckpt 系列,OUTPUT_DIR 字面值不一致就是隐性 bug 源。**正确做法**:配套脚本顶部写注释 `# ★ 必须与 pipeline OUTPUT_DIR 一致`,或抽到 `project_config.py` 共享常量。复查方法:`grep -n "^OUTPUT_DIR\|OUTPUT_DIR =" *.py` 一行命令对齐所有入口脚本。

**★★★ A18 — `model.load_state_dict(strict=False)` 不会因为 key 命名错误报错,会静默把所有 key 塞进 `missing` 列表。** 这是 PyTorch 的设计:strict=False 是用来宽容下游 model 比 ckpt 多模块的(missing 大于 0 没问题)的,不是用来宽容 ckpt 比 model 多模块的(unexpected 大于 0 才是错)。但有个隐藏坑:**如果 pretrain 端拼装 state_dict 时 key 命名拼错了**(例如把 `similar_proj.0.weight` 错写成 `similar_proj.proj.0.weight`),strict=False 会把所有错命名的 key 全塞进 `missing`,日志只显示"加载了 0 个 unexpected"看起来一切正常,实际 model 等于完全随机初始化。**A18 防回归方案**:加载完后必须显式校验关键 key **真的不在 missing 列表里**。代码:`_critical = ["input_bn.weight", "input_proj.weight", "encoder.layers.0.self_attn.in_proj_weight", "similar_proj.0.weight"]; missed = [k for k in _critical if k in missing]; assert not missed`。这条原则普适所有 strict=False 加载场景。

**★★★ A18 — pretrain 脚本写出的 ckpt 必须带 `version` 与完整 `config` 字段,pipeline 加载时用这两个字段做硬校验。** 不带 version 的话,如果用户不小心把 v5 老 ckpt(`input_proj` 6→128)拷到 v6 ckpt 路径下,`load_state_dict(strict=False)` 在维度不匹配时会 raise(因为不是 strict 也救不了 size mismatch)——这条触发是好事,但堆栈信息是 PyTorch 内部的形状错误,不直观。带 version 后可以**前置校验**给出业务级错误提示(`"v6 model 需要 v6 ckpt, 当前 ckpt version=5"`)。`config` 字段还应包含 `input_size` / `retrieval_flat_dim` / `field_names`,加载方逐项校验,防 cache stale 或参数改了忘重 pretrain。

**★★★ A18 — 文件命名规范应该按"业务模块所有制"分组,不是按动作分组。** v5 时代命名 `pretrain_mera.py`(动词 + 模块)让所有 pretrain 脚本聚到一起,但项目其实没有第二个 pretrain。改为 `mera_pretrain_v6.py`(模块 + 动词 + 版本)让所有 mera 资产聚到一起,与 `qlib_pipeline_mera_*` 管线本体并列,与 `ensemble_*.py` 融合版系列对应。**项目级承诺**:除 `qlib_pipeline_mera_*.py` 与 `qlib_trading_mera_*.py` 是管线/交易主入口外,所有 mera 配套工具一律 `mera_` 起头(`mera_pretrain_v6.py` / `mera_single_window_diag_v6.py` / `mera_hb_grid_search.py`),所有融合版工具一律 `ensemble_` 起头。

### 1.3 已完成里程碑(精简表)

| 阶段 | 主题 | 关键产出 |
|------|------|---------|
| A1 | MERA v5 管线建立:5 项架构修复 | GRU Expert / Gate bucket-only / 无残差 MoE / TRA 2层Head / SMA=5 |
| A2 | 参数全面对齐官方 + 工程落地 | hidden=128 / experts=8 / top4 / dropout=0.3 / lr=1e-4 / neighbors=50 / gate_aux=1.0 |
| A3 | 全市场→成分股 Universe 缩放 | 训练样本 330万→~100万,每 epoch 7-8min→~2min,RSS 36GB→~15GB |
| A4 | 训练速度优化 + 断点续跑 + Windows 兼容性 | 每 epoch 299s→120s,instrument 大小写修复 |
| A5 | 官方代码审查 + 关键参数修正 + SimCLR 预训练 | gate_aux_weight 1.0→0.01,早停 val_IC,SimCLR pretrain |
| A6 | 训练循环全面对齐官方 + 预训练工程完善 + 回测可视化 | 删 lr scheduler,MAX_EPOCHS=500,五面板回测图 |
| A7 | 检索 Ablation + 检索空间替换 + 实验隔离 + MERA Trading | RETRIEVAL_ABLATION 三模式,EXPERIMENT_TAG,BankCollator,推理检索 35min→1min,qlib_trading_mera.py |
| A8 | 标签修正 + 短期信号对齐 + 预训练 v2 + 单窗口诊断 + HB Grid Search + 工程清理 | 前瞻偏差修复 `Ref($open,-(N+1))/Ref($open,-1)-1`,DEFAULT_PERIOD=2,IC=0.024 Sharpe=1.30,HB=0.00 |
| A9 | 预训练 v3 升级 + 数据预处理 inf 根因修复 + Pipeline 训练加速 | ContrastiveHead 128→256→256→128 + LayerNorm,FeaturePreprocessor inf→NaN 链修复,SMA→EMA |
| A10 | D 组实验结果 + Shared Expert Isolation + Pipeline WIP + Expert 分化诊断 | D 组 7 窗口 OOS Sharpe=1.09 年化 26.4% 回撤 -38.3%;DeepSeekMoE Shared Expert(1 shared+7 routed) |
| A11 | Expert 分化深度诊断 + 共享 expert 对照 + 分化指标体系 + aux loss 抑制确认 | 6 项分化指标,A vs D vs D_shared,D 组 OOS IC=0.091(vs A 组 0.047),aux=0.01 在 10-30 epoch 消除分化 |
| A12 | MERA v6 架构升级:13 维异构输入 + MASTER 市场引导门控 + 行业中性化 + ETL 集成 | INPUT_SIZE 6→13,MARKET_DIM=30(3 指数 × 10 统计量),`α=2·softmax(W·m)`,bank 降级 360 维量价,`Output/Mera_Output/` |
| A13 | MERA v6 数据管线诊断与关键 bug 修复 | split cross-sectional → time-series,Feature scale 三项修复,MASTER gate rolling 252 日,FeaturePreprocessor in-place,检索 FP16,TF32,RAM 释放 16GB |
| A14 | 顶刊机制借鉴落地与论文理解诚实化 | 单窗口诊断扩展内置回测,HB Sweep v2 证实信号层天花板,RSAP-DFM 借鉴版 Path 2 实现,九篇顶刊原文精读,§七 重写 |
| **A14+** | **Path 2 部署 + Path 3 Hierarchical MoE + UMI RankIC loss + BankCollator 5 元组修复** | **三轮 FGSM 修复(v1 scaler.scale 错加 / v2 FP16 g.norm overflow / v3 FP32 隔离根治,4× encoder forward → 1.3×);Path 2 W1 val IC 0.069 / Sharpe 1.034 过阈值但 2024 daily IC 从 0.033 衰退到 0.0185;Path 3 HierarchicalMoE 2 macro × 4 micro;RankIC loss 零架构改动;Dataset 5 元组 + BankCollator 同步;tqdm 进度条** |
| **A15** | **Path 2 W1 第二次复现确认 2024 系统性退化** | **val 0.069→0.081,Sharpe 1.034→0.910 跌破阈值,2024 daily IC 0.0185→0.0131 进一步退化** |
| **A16** | **融合版三件套首战 + 基准对齐 csi1000 + 参数区分层整合** | **D_m14_p2_p3_p4 W1 val IC 0.1087 / OOS daily IC 0.0358 / IR 6.94 / Sharpe 1.084 / 累计超额 152%,2024 daily IC 救回 0.0284(+118%)。Macro gate 真分化 0.68/0.20。Max DD -36.2% 创项目最差。基准全面切到 SH000852 中证 1000。Tag 命名重写为 Path X,参数区 9 段分层整合。Diag 同步 pipeline resume 逻辑** |
| **★ A17** | **`qlib_pipeline_mera_v6.py` 代码精简重构 + Helper 抽取** | **业务逻辑一行不动,只做真重复抽公因子 + 长函数纯组装/IO 块外移。`train_mera_window_v5` ~~560 行~~ → 456 行;`main()` ~~160 行~~ → 31 行;`build_alpha360_expressions` ~~140 行硬编码模板~~ → 13 字段 spec-table + 单 loop(69 行);`HierarchicalMoE` ~~164~~ → 138 行;`SparseSequenceMoE` ~~104~~ → 83 行。新增 7 个顶层 helper:`_build_gate_in_modulated`(合并 forward / forward_train / build_gate_input_with_regime 三处共享的 RETRIEVAL_ABLATION 三分支 + regime z 调制)、`_moe_dispatch_topk`(flat + Hierarchical MoE 共享 scatter-gather,省 70 行重复)、`_load_pretrained_encoder_into`、`_resume_wip_or_init`、`_save_window_checkpoint`、`_print_config_summary`、`_print_backtest_stats`、`_plot_five_panels`。零风险冗余清理:重复 `import torch` 移除,`from qlib.data import D` 三处内层 import 上提模块顶部。等价性:`build_alpha360_expressions` 用 `/home/claude/test_equiv.py` 跑 780×2 字符串逐字节比对,与原版完全一致。47 个诊断脚本依赖的 public 名字全部保留** |
| **★ A18** | **v6 自监督预训练落地 + Pipeline ckpt 加载链路完整化 + 文件命名规范统一** | **`mera_pretrain_v6.py` 实质实现(A11/A16/A17 三轮列首要任务终于落地):13 维输入 + SimCLR 双视图增强 + cross-space 360↔[60,13] align 双任务 + 行业中性化 + 工程级 4 重 ckpt 校验。`SimilarProjector` 输入 360 维(v6 [M3] 检索 bank 设计选择,非 780),`PretrainDataset.flat_x_price` 切出量价 6 字段产生 `[N, 360]`。Pipeline `_load_pretrained_encoder_into` 由 A17"警告 stub"升级为真实加载:存在性 / version==6 / input_size==13 / retrieval_flat_dim==360 / 关键 key 真的不在 missing 列表四重校验。`USE_PRETRAINED_ENCODER` 默认 True 不再被强制覆盖。`PRETRAIN_DIR_V6 = OUTPUT_DIR / "mera_v6_pretrain"` 新常量。文件命名规范统一:`pretrain_mera_v6.py` → `mera_pretrain_v6.py`,与 `qlib_pipeline_mera_v6.py` 并列、与 `ensemble_*.py` 系列对应。第一次落地 OUTPUT_DIR 错配 `Output/Qlib_DL` → `Output/Mera_Output` 的工程教训沉淀为方法论 §15.24。A17 待办的 `_print_backtest_stats` IR/beta 口径升级仍未做(继承到 A18 待办)** |

### 1.4 当前配置(★ A14+ 更新,★ A17 行数与 helper 注记同步)

| 参数 | 值 | 官方值 | 说明 |
|------|:---:|:-----:|------|
| 模型 | MERA_V6 | — | A12: Transformer + MoE + 检索增强 + 13 维输入 + MASTER gate |
| 输入 | [60, 13] | [T, 351] | A12 |
| 输入字段顺序 | [open, close, high, low, vwap, volume, log_mv, pe_signed_log, pb_signed_log, roe, turnover_log, mtss_stock_rel, mtss_flow_rel] | — | A12 |
| FLAT_DIM | 780 | — | A12: 60 × 13 |
| RETRIEVAL_FLAT_DIM | 360 | — | A12: 仅量价 60 × 6,bank 降级避免异构污染 |
| MARKET_DIM | 30 | — | A12: 3 指数 × 10 统计量 |
| 市场指数池 | SZ399303 / SH000905 / SH000985 | — | A12: 国证 2000 / 中证 500 / 中证全指 |
| 市场统计量 | ret_1d, ret_5d, ret_20d, ret_60d, vol_20d, vol_60d, vol_ratio_20d, up_ratio_5d, up_ratio_20d, amp_20d | — | A12 |
| **市场特征 z-score** | **rolling 252 日** | — | **A13: 原 expanding 在 OOS 上 α σ 衰减到训练期 1/7** |
| MARKET_ROLLING_WINDOW | 252 | — | A13 |
| MARKET_ROLLING_MIN_PERIODS | 60 | — | A13 |
| MARKET_ZSCORE_CLIP | 5.0 σ | — | A13 |
| MASTER gate | `α = 2·softmax(W·m)` | MASTER AAAI 2024 | A12: W 为 `nn.Linear(30, 13)`,zeros 初始化 → 初始 α=2/13 均匀 |
| MASTER gate 应用位置 | BatchNorm 之前,通道级乘法 `x̃ = α ⊙ x` | 同 | A12 |
| 行业中性化 | PE/PB signed_log → industry median & MAD z-score | — | A12 |
| **Volume 预处理** | **log1p(x) 压缩** | — | **A13: IPO/复牌首日 ratio 可达 16900** |
| **ROE 预处理** | **signed_log(x)** | — | **A13: raw 极端值 max=3230 min=-399** |
| **Turnover 预处理** | **log1p(x)** | — | **A13: Qlib $turnover_rate 是百分比形式** |
| Backbone | Transformer(2L, h=128, heads=4) | 同 | norm_first=True, FFN=512, GELU |
| MoE | 8 experts (GRU 2L), top-4 | 同 | |
| NUM_SHARED_EXPERTS | 0 (D_m13) / 1 (D_m13_s) | — | A10: DeepSeekMoE Shared Expert Isolation |
| Gate | NoisyTopKGate(d=16) | noisy_vmoe | |
| gate_aux_weight | 0.01 | 0.01 (硬编码) | |
| 预测头 | MLP(128→64→1) | 同 | TRA num_states=1 |
| SMA→EMA | decay=0.2 (≈window=5) | smooth_steps=5 | A9 |
| VAL_EVERY | 2 | — | A9 |
| cudnn.benchmark | Pipeline: True / Diag: False | — | A11 |
| **TF32** | **ON** | — | **A13: 训练 +20-40%,与 AMP 兼容** |
| **检索 bank dtype** | **FP16** | — | **A13: top-k 对精度不敏感,3:33→1:30** |
| dropout | 0.30 | 0.30 | |
| lr | 1e-4 | 1e-4 | 固定不衰减 |
| 早停标准 | val_IC (Spearman) | val_IC | A5 |
| patience | 20 | early_stop=20 | |
| EARLY_STOP_MIN_DELTA | 0.005 | — | A11 |
| EARLY_STOP_SMOOTH | 5 | — | A7 |
| max_epochs | 200 | 500 | |
| MAX_STEPS_PER_EPOCH | 500 | 200 | A7 |
| batch 组织 | 按天 2 天/步 (~4000/步) | 2 天 × ~500 只 | A6 |
| N_NEIGHBORS | 50 | 50 | |
| SIMILAR_BATCH | 2048 | — | A8 |
| 标签 | ret_2d | — | A8 |
| HB | 0.00 | — | A8 |
| TOP_K | 200 | — | |
| **UNIVERSE_CSV** | **cons_csi1000.csv** | CSI300/500 | **A16: 主 pipeline 与 diag 全面切到中证 1000,废弃国证 2000;diag 历史使用 gz2000 universe 自洽于 SZ399303 基准** |
| **基准** | **SH000852(中证1000)** | — | **A16: 与 csi1000 universe 强配对,所有 W1 实测自此切换;historic gz2000 实验自洽口径仍是 SZ399303** |
| **Universe ↔ Benchmark 配对** | **csi1000 ↔ SH000852 / gz2000 ↔ SZ399303** | — | **A16: 运行时 assert 强制配对** |
| AMP | ON (FP16) | — | |
| torch.compile | OFF | — | Windows 无 Triton |
| ~~预训练~~ **预训练** | ~~OFF (v6 首轮)~~ **ON (v6, mera_v6_pretrain)** | 未开源 | ~~A12: v5 pretrain 是 [60, 6] 维,v6 [60, 13] 不兼容~~ **★ A18: 13 维版 `mera_pretrain_v6.py` 已实现,默认开;v5 pretrain `pretrain_mera.py` 完全保留不动用作 v5 回退** |
| **★ A18 PRETRAIN_DIR_V6** | **`Output/Mera_Output/mera_v6_pretrain`** | — | A18: 与 v5 `mera_v5_pretrain` 并存隔离 |
| **★ A18 PT_LABEL_PERIOD** | **2** | — | A18: 与 pipeline `DEFAULT_PERIOD` 对齐,直接复用 v6 cache |
| **★ A18 PT_EPOCHS / PT_BATCH_SIZE** | **100 / 8192** | — | A18: 沿用 v3 SimCLR 配置 |
| **★ A18 PT_ALIGN_WEIGHT** | **0.5** | — | A18: cross-space align loss 权重 |
| **★ A18 PT_TEMPERATURE** | **0.1** | SimCLR 0.1 | A18: NT-Xent 温度 |
| **★ A18 PT_MAX_STEPS** | **200** | — | A18: 单 epoch 步数上限,大 dataset 时早停 |
| 回测可视化 | 五面板图 | — | A6 |
| DataLoader num_workers | 动态 (2-6) | — | A10 |
| Pipeline WIP | 每 10 epoch | — | A10 |
| SMA storage | CPU | — | A10 |
| **EXPERIMENT_TAG** | **"D_m14" + 自动 path 后缀** | — | **A16: Path X 命名 base/p2/p2reg/p3/p4 自动按开关组合** |
| **VALID_RATIO 切分方式** | **按 datetime 时序切** | — | **A13: 原 cross-sectional split,val IC 虚高 17 倍** |
| **split_train_val 返回类型** | **Tuple[np.ndarray, np.ndarray]** | — | **A13** |
| **USE_RSAP_DFM** | **True**(Path 2 主开关) | — | **A14: False 时行为完全等价原 v6** |
| **REGIME_D_Z** | **16** | — | A14 |
| **REGIME_EXTRACTOR_HIDDEN** | **32** | — | A14 |
| **REGIME_EXTRACTOR_DROPOUT** | **0.2** | — | A14 |
| **REGIME_Z_TO_GATE** | **"film"** | — | **A14: γ·gate_in+β。可选 "concat"/"off"** |
| **USE_ADV_PERTURB** | **True**(FGSM) | — | A14 |
| **ADV_EPSILON** | **0.01** | — | A14 |
| **ADV_TARGET** | **"gate_input"** | — | A14: 可选 "last_hidden" |
| **ADV_WARMUP_EPOCHS** | **5** | — | A14 |
| **ADV_LOSS_WEIGHT** | **0.3** | — | **A14: joint loss 不做 bilevel** |
| **DEFAULT_HB 建议值** | **0.05** | — | **A14: HB Sweep v2 证明 HB 旋钮对信号层无效** |
| **USE_HIERARCHICAL_MOE** | **False/True** | — | **A14+ Path 3: True 时强制 USE_RSAP_DFM=True** |
| **HIERARCHICAL_N_MACRO** | **2** | — | A14+ |
| **HIERARCHICAL_N_MICRO** | **4** | — | A14+: N_MACRO × N_MICRO 必须等于 NUM_EXPERTS |
| **HIERARCHICAL_MICRO_TOPK** | **2** | — | A14+: 每样本激活 1×2 = 2 个 micro expert(vs flat top-k=4) |
| **HIERARCHICAL_MACRO_TOPK** | **1** | — | A14+: 默认 1 = 硬路由 |
| **USE_RANK_IC_LOSS** | **False/True** | — | **A14+: hybrid loss = MSE_WEIGHT·MSE + RANK_IC_WEIGHT·(-RankIC) + aux** |
| **MSE_WEIGHT** | **0.7** | — | A14+ |
| **RANK_IC_WEIGHT** | **0.3** | — | A14+ |
| **RANK_IC_MIN_STOCKS** | **10** | — | A14+: 每日截面少于则跳过 |

**★ A16 配置使用建议(Path X 命名)**:

- **D_m14_base**(全开关 False):等价 A13 baseline,OOS daily IC ~0.022 / Sharpe 0.898
- **D_m14_p2**(Path 2 完整):`USE_RSAP_DFM=True` + `USE_ADV_PERTURB=True`
- **D_m14_p2reg**(消融):`USE_RSAP_DFM=True` + `USE_ADV_PERTURB=False`,仅 regime 不带 FGSM
- **D_m14_p3**(纯净):`USE_HIERARCHICAL_MOE=True` + `USE_RSAP_DFM=False`(A14+ 设计中此组合不允许,A16 视实验需求可考虑放宽)
- **D_m14_p4**:`USE_RANK_IC_LOSS=True`,零架构改动
- **D_m14_p2_p3_p4**(融合版三件套,A16 W1 已实测):2024 daily IC 救回 0.0284 ✅,Max DD -36.2% 创项目最差

**Tag 自动拼接逻辑**(diag 顶部参数区第 1 节):

```python
_paths = []
if USE_RSAP_DFM:        _paths.append("p2" if USE_ADV_PERTURB else "p2reg")
if USE_HIERARCHICAL_MOE: _paths.append("p3")
if USE_RANK_IC_LOSS:    _paths.append("p4")
if not _paths:          _paths.append("base")
_tag = "_".join([EXPERIMENT_TAG] + _paths)   # → D_m14 / D_m14_p2 / D_m14_p2_p3_p4
```

---

## 二、数据资产

### 2.1 数据规模

| 数据源 | 股票数 | 起始 | 终止 | 日频字段 |
|--------|-------|------|------|---------|
| JQ CSV (v7) | 5,188 | 2006 | 2026 | 31 列(含 2 列 mtss)|
| Qlib .bin | 5,188 + 指数 | 2006 | 2026 | 含 vwap + mtss_fin_value/mtss_fin_buy |
| 国证2000基准 | 1 | 2006 | 2026 | CSV: `benchmark_gz2000.csv` + Qlib .bin: `features/SZ399303/` |

基准指数(`qlib_converter.py::BENCHMARK_INDEXES`):

| Qlib 代码 | JQ 代码 | 指数名 |
|---|---|---|
| SZ399303 | 399303.XSHE | 国证2000 (历史 gz2000 universe 配对基准 + MASTER 市场向量主指数) |
| SH000852 | 000852.XSHG | 中证1000 (★ A16: csi1000 universe 配对基准) |
| SH000903 | 000903.XSHG | 中证100 |
| SH000905 | 000905.XSHG | 中证500 (MASTER 市场向量副指数) |
| SH000906 | 000906.XSHG | 中证800 |
| SH000985 | 000985.XSHG | 中证全指 (MASTER 市场向量副指数) |

**★ A13 数据目录布局**:JQ CSV 在 `Dataset2006JQ/MarketData/` 子目录下,文件名格式 `{code6}.{SZ|SH}.CSV`(大写扩展名),例如 `002222.SZ.CSV`。成分股 CSV 在 `Dataset2006JQ/` 根目录(不在 `MarketData/`)。Qlib features 目录可能用小写前缀,但 `D.features()` API 调用时必须用大写字符串。

### 2.2 JQ CSV 字段(31 列)

代码, 简称, 日期, 前收盘价, 开盘价, 最高价, 最低价, 收盘价, 涨跌, 涨跌幅(%), 成交量, 成交金额, 均价, 换手率(%), 总股本, 流通股本, 总市值, 流通市值, 市盈率, 市净率, 市销率, 市现率, 涨停价, 跌停价, 停牌, 所属行业, 所属概念, 营收同比(%), 净利润同比(%), ROE(%), 毛利率(%), **融资余额, 融资买入额** (A12 新增两融字段)

**★ A13 — Qlib `$turnover_rate` 单位定论**:百分比形式(3.665 表示 3.665%),保留不变。v6 `FeaturePreprocessor._transform_stage_a` 原代码 `log1p(x * 100)` 错。修复:`log1p(x)`。

### 2.3 Qlib .bin 字段

由 `etl/qlib_converter.py` 生成,存储于 `PROJECT_ROOT/qlib_data/cn_jq/`。

Staging CSV 列(~27 列):date, open, close, high, low, volume, money, vwap, factor, paused, high_limit, low_limit, turnover_rate, market_cap, float_market_cap, pe_ratio, pb_ratio, ps_ratio, pcf_ratio, total_shares, float_shares, rev_growth_yoy, profit_growth_yoy, roe, gross_margin, **mtss_fin_value, mtss_fin_buy**(A12)

代码格式转换:JQ `600000.XSHG` → Qlib `SH600000`;JQ `000001.XSHE` → Qlib `SZ000001`

converter 流程(A12 扩展为 5 步):(1) staging (2) benchmark staging (3) dump (4) verify (5) rebuild_market_features_v6 → `Mera_Output/market_features_v6.pkl`

**vwap.day.bin 断点续传陷阱**:`DumpDataAll` 对已存在的 features 目录可能不会重新生成新增字段的 .bin。新增字段(如 vwap / mtss_*)时,必须删除整个 `qlib_data/cn_jq/features/` 目录再重跑 ETL。

### 2.5 Alpha360 数据准备

Alpha360 = 6 字段 × 60 天 = 360 维。

所需 .bin 字段覆盖情况:

| 字段 | 状态 | v5 用 | v6 用 |
|------|:----:|:----:|:----:|
| `$open` | ✅ | ✓ | ✓ |
| `$close` | ✅ | ✓ | ✓ |
| `$high` | ✅ | ✓ | ✓ |
| `$low` | ✅ | ✓ | ✓ |
| `$volume` | ✅ | ✓ | ✓ |
| `$vwap` | ✅ | ✓ | ✓ |
| `$factor` | ✅ | ✓ | ✓ |
| `$float_market_cap` | ✅ | — | ✓ A12 |
| `$pe_ratio` | ✅ | — | ✓ A12 |
| `$pb_ratio` | ✅ | — | ✓ A12 |
| `$roe` | ✅ | — | ✓ A12 |
| `$turnover_rate` | ✅ | — | ✓ A12 |
| `$mtss_fin_value` | ✅ | — | ✓ A12 |
| `$mtss_fin_buy` | ✅ | — | ✓ A12 |
| `$gross_margin` | ✅ 但 75% NaN | — | ❌ A12 DROPPED |

Alpha360 表达式构建:

- **v5**: `Ref($field, d) / $close - 1`(价格归一),`Ref($volume, d) / Mean(Ref($volume, 1), 59)`(volume 归一)。6 字段 × 60 天 = 360 维
- **A12 v6**: 上述量价 6 字段照旧 + 7 字段基本面/两融扩展(每字段 60 天共 420 维),总 780 维。Qlib 表达式只取原始值,所有 Python 内部变换在 `FeaturePreprocessor` 里完成。新增字段表达式:
  - `log_mv`: `Log($float_market_cap)` × 60 天(无滞后)
  - `pe_signed_log` / `pb_signed_log`: `Ref($pe_ratio, 1)` × 60 天(Python 里 `sign(x) * log(1 + |x|)` + 行业中性化)
  - `roe`: `Ref($roe, 1)` × 60 天
  - `turnover_log`(★ A13 修正):Qlib 仅取 `Ref($turnover_rate, d)` 原始值(百分比),Python 层 `log1p(x)` 去掉 ×100
  - `mtss_stock_rel`: `Ref($mtss_fin_value, 1) / Ref($float_market_cap, 1)` × 60 天
  - `mtss_flow_rel`: `Ref($mtss_fin_buy, 1) / Ref($money, 1)` × 60 天

### 2.6 成分股过滤

**动机**:MERA 及其他顶刊实验都在 CSI300/500/1000 上做。缩小到成分股(~1000-2000 只)后训练样本降到 ~60-130 万。

**数据文件**:
- `Dataset2006JQ/cons_gz2000.csv`:国证2000 每日成分股,2014-10-17 至今
- `Dataset2006JQ/cons_csi1000.csv`:中证1000 每日成分股,同期

**过滤机制**(`_filter_universe()`):读取成分股 CSV → code 转 Qlib 大写格式 → 构建 `frozenset((date, instrument))` → 在 `load_alpha360_data()` 返回之前统一过滤。

**参数控制**:文件顶部 `UNIVERSE_CSV = "cons_gz2000.csv"` 或 `"cons_csi1000.csv"`。

**★ A13 备注**:成分股 CSV 的 `code` 列可能是整数类型(例如 `2222` 表示 `002222`)。读取时需要 `str(int(code)).zfill(6)` 转为 6 位字符串。

### 2.7 ETL 财报数据集成

| 参数 | 值 |
|------|-----|
| `FUNDAMENTAL_FIELDS` | `['inc_revenue_year_on_year', 'inc_net_profit_year_on_year', 'roe', 'gross_profit_margin']` |
| `FUNDAMENTAL_SAMPLE_INTERVAL` | `21` |
| `MTSS_FIELDS` (A12) | `['fin_value', 'fin_buy_value']` |

### 2.8 JQData 行业/概念 API 映射

| API | 数据范围 | CSV 列 |
|-----|---------|--------|
| `get_industries(name)` | 2005 至今 | — |
| `get_industry(security)` | 2005 至今 | 所属行业 |
| `get_industry_stocks(code)` | 2005 至今 | — |
| `get_concept(security)` | 2016 至今 | 所属概念 |

行业分类缓存刷新:`CLASSIFICATION_REFRESH_DAYS = 30`。

**`Dataset2006JQ/industry_map.pkl`**(A12):`dict[code6_str → industry_name_str]`,共 5194 键,JQ 最新行业分类静态快照。v6 PE/PB 行业中性化依赖此文件。静态快照近似:60 天 lookback 内行业变动几乎无影响。

### 2.9 两融数据集成(A12)

JQData API:`finance.run_query(...)`。字段映射:

| JQ 字段 | Staging 列 | .bin 字段 | v6 用途 |
|---|---|---|---|
| `fin_value` | `mtss_fin_value` | `$mtss_fin_value` | mtss_stock_rel = fin_value / float_market_cap |
| `fin_buy_value` | `mtss_fin_buy` | `$mtss_fin_buy` | mtss_flow_rel = fin_buy_value / money |

**覆盖率**:在 2023-06 月全 4903 只股票上验证:64.3% 全覆盖,35.7% 非两融返回 NaN。NaN → 中位数填充。

**数据起点**:JQData mtss 起点约 2010-03-31。v6 训练集用 2015+ 数据,完全覆盖。

**★ A13 验证**:数据对齐脚本对 002222(福晶科技)2020-02-04 手算 `mtss_stock_rel = 2.66441e8 / 4.74099e9 = 0.0561995`,与 pipeline 一致。Ref(d,1) 滞后将 T 日盘后公布的两融余额推给 T+1 日 feature 使用,合规无前瞻。

### 2.10 市场特征数据(A12 + ★ A13 z-score 修正)

每个交易日一个 30 维向量 `m_t`。30 维 = 3 个指数 × 10 个统计量:

| 统计量 | 公式 |
|---|---|
| ret_1d | close/close_1 - 1 |
| ret_5d | close/close_5 - 1 |
| ret_20d | close/close_20 - 1 |
| ret_60d | close/close_60 - 1 |
| vol_20d | std(ret_1d, 20) × √252 |
| vol_60d | std(ret_1d, 60) × √252 |
| vol_ratio_20d | volume / mean(volume, 20) |
| up_ratio_5d | mean(ret_1d > 0, 5) |
| up_ratio_20d | mean(ret_1d > 0, 20) |
| amp_20d | mean((high-low)/close, 20) |

**3 个指数**:SZ399303 + SH000905 + SH000985。三个指数的 30 维特征 inner join。

**归一化**(★ A13 修正为 rolling):`t` 时刻 mean/std 用 `[t-window+1 .. t]` 滚动窗口(window=252 日,min_periods=60)。前 60 天数据不足返回零向量。结果 clip 到 ±5σ,NaN 置零。

**为什么改 rolling**:expanding z-score 随 t 增大分母越稳定,z-score 幅度单调衰减。A12 v6 实测训练期 σ=0.045,OOS σ=0.006(衰减 7-8 倍),MASTER gate 在 OOS 上几乎瘫痪。rolling 下 mean/std 历史长度恒定,训练/OOS 波动幅度一致。

**工程实现**:A13 新增 `_recompute_market_zscore_rolling()` 模块级函数。pipeline `MarketFeatureProvider.load_or_build()` 加载 pkl 后,如果 `MARKET_ZSCORE_MODE="rolling"`(默认),直接对 `obj.raw_features` 重算 rolling z-score 替换 `obj.features`。**不需要重建 pkl**(raw_features 不变,只是 features 重算)。

**warmup**:65 天最小数据。rolling min_periods=60。

**存储**:`Output/Mera_Output/market_features_v6.pkl`。格式为 `dict` 带 `__format__="market_features_v6_dict"` 标记,避免 pickle 本地类序列化问题。字段:

```python
{
    "__format__": "market_features_v6_dict",
    "date_index": pd.DatetimeIndex,     # 交易日
    "features": np.ndarray [N, 30],     # 归一化后 (pkl 里存的是 expanding 版,pipeline 加载后按 MARKET_ZSCORE_MODE 决定是否重算为 rolling)
    "raw_features": np.ndarray [N, 30], # 原始值
    "meta": {
        "version": "v6",
        "index_codes": ["SZ399303", "SH000905", "SH000985"],
        "stat_names": [...],
        "warmup_days": 65,
        "zscore_min_history": 252,
        "build_time_sec": float,
        "rebuilt_by": "qlib_converter.py",
    }
}
```

**查询接口**:`MarketFeatureProvider` 类内联在 `qlib_pipeline_mera_v6.py`:`get(date) -> [30]` / `get_batch(dates) -> [N, 30]` / `get_dict()` / `load_or_build(start, end)`(★ A13: 加载后自动按 `MARKET_ZSCORE_MODE` 重算 z-score)。

**★ A13 观察**:改 rolling 之后,整体 z-score 值域比 expanding 更紧凑,但跨样本 std 应保持在训练/OOS 期相近水平(均为 0.03-0.08 量级)。如果 rolling 下仍然衰减,查 raw_features 是否系统性漂移。

---

## 三、模型架构 — MERA v6

### 3.1 整体架构

```
Input x [B, 60, 13]  +  market vector m [B, 30]
    │                         │
    │                    Linear(30 → 13)  ★ A12 MASTER gate
    │                    ↓
    │                    α = 2 · softmax(·)      # 通道级 gating 系数
    │                         │
    ├──────★ x̃ = α ⊙ x ★──────┤                 # pre-BN 应用
    │
BatchNorm1d(13) → Linear(13, 128) → PositionalEncoding → TransformerEncoder(2L, h=128, heads=4)
    ↓ seq_hidden [B, 60, 128]
    ↓
├── last_hidden = seq_hidden[:, -1, :]  →  [B, 128]
│
├── Similar Retrieval (GPU):
│   query(flat_price_360) → cosine topK(50) from price-only train bank
│   ★ A12: bank 只存 360 维量价(避免异构污染)
│   ★ A13: bank 用 FP16 做 matmul(3:33→1:30)
│   → neighbor_indices [N, 50]
│
├── Gate Input Construction:
│   similar_proj(360→128) → attn(query, sim_feat) → [B, 1, 50]
│   → weighted label_embedding(bucket→16) → [B, 16]
│
├── ★ A14 Path 2: Regime Extractor φ_DM (USE_RSAP_DFM=True 时启用):
│   last_hidden [B, 128] → Linear(128, 32) → ReLU → Dropout(0.2) → Linear(32, 16)
│   → z [B, 16]  (stock-hidden derived regime latent)
│   REGIME_Z_TO_GATE 决定 z 如何影响 gate_in:
│     "film":   γ,β = Linear(16, 2*gate_dim), gate_in' = γ·gate_in + β
│     "concat": gate_in' = Linear(concat([gate_in, z]), gate_dim)
│     "off":    gate_in 不被 z 调制 (z 仍走下游 Path 3 if 启用)
│
├── ★ A14 Path 2: FGSM 对抗扰动 (USE_ADV_PERTURB=True 且 epoch >= 5):
│   g = ∂loss_main / ∂gate_in   (autograd.grad, retain_graph)
│   with autocast(enabled=False):  # ★ FP32 隔离防 FP16 overflow
│     g_fp32 = nan_to_num(g.float()); clamp(±1e4)
│     gate_in_adv = gate_in + ADV_EPSILON · (g_fp32 / g_fp32.norm)
│   Forward B (复用 seq_hidden.detach()): MoE + pred_head → pred_adv
│   loss = loss_main + ADV_LOSS_WEIGHT · loss_adv
│
├── Sparse MoE (8 experts, top-4):
│   [路径 A: 扁平 MoE (USE_HIERARCHICAL_MOE=False, 默认)]
│   NoisyTopKGate(16→8 或 16→7) → route to top-k GRU Experts
│   GRU(128, 128, 2L) per expert → gate-weighted sum → [B, 60, 128]
│
│   [路径 B: ★ A14+ Path 3: Hierarchical MoE (USE_HIERARCHICAL_MOE=True)]
│   L1 macro gate: NoisyTopKGate(z [B, 16] → 2 macros, top-1 硬路由)
│   L2 micro gates (每 macro 独立): NoisyTopKGate(gate_in [B, 16] → 4 micros, top-2)
│   For each macro m in {0, 1}:
│     mask = (macro_topk_idx == m)
│     sub_out = scatter-gather(seq_hidden[mask], micro_experts[m])
│     out[mask] = sub_out · macro_gates[mask, m]
│   aux = macro_aux + Σ micro_aux[m]
│
└── Prediction Head:
    MoE_output[:, -1, :] → Linear(128, 64) → ReLU → Linear(64, 1) → pred_score

★ A14+ Loss 计算:
   std 路径: loss = _make_loss(pred, aux)
   adv 路径: loss = _make_loss(pred_main, aux_main) + ADV_LOSS_WEIGHT · _make_loss(pred_adv, aux_adv)
   
   _make_loss(pred, aux):
     if USE_RANK_IC_LOSS:
       with autocast(enabled=False):  # FP32 防 FP16 underflow
         rank_ic = rank_ic_loss(pred.float(), yb.float(), date_ids, min_stocks=10)
       return MSE_WEIGHT · MSE(pred, yb) + RANK_IC_WEIGHT · rank_ic + GATE_AUX_WEIGHT · aux
     else:
       return MSE(pred, yb) + GATE_AUX_WEIGHT · aux
```

### 3.2 关键组件详解

**MASTER Market-Guided Gate**(A12):`nn.Linear(MARKET_DIM=30, INPUT_SIZE=13)`,初始化 `nn.init.zeros_(weight)` + `nn.init.zeros_(bias)`,保证训练开始时 `α = 2 · softmax(0) = 2/13 · ones`(完全均匀)。α 的和固定为 2.0(softmax 归一化后乘以 input_size,公式里的 `2` 是 MASTER 原论文设计)。应用位置:BatchNorm 之前,`x̃ = α ⊙ x`(broadcast 乘法,[B, 60, 13] × [B, 1, 13])。

**为什么在 BN 之前**:BN 会把每个维度 normalize 到 N(0, 1),如果放在 BN 之后,α 的效果会在下一个 batch 的 BN 统计量里被消化。pre-BN 应用让 α 真正影响每个维度的相对信号强度。

**★ A13 修正 — MASTER gate 在 OOS 上的瘫痪风险**:Gate 的输入 `m` 来自 `MarketFeatureProvider.get_batch(dates)`,如果 `m` 本身的跨时间标准差随时间衰减(expanding z-score 的固有性质),那 Linear 层的输出 `W·m` 也会衰减,softmax 后 α 会退化到接近均匀的 2/13。原版 pkl 用 expanding,v6 首轮诊断实测 α 训练期 σ=0.045 vs OOS σ=0.006。修复:rolling 252 日 z-score。

**MarketFeatureProvider(内联)**(A12):dataclass,持有 `date_index` + `features [N, 30]` + `raw_features [N, 30]` + `meta`。从 `market_features_v6.pkl` 加载。位于 `qlib_pipeline_mera_v6.py` 顶部。**★ A13**:`load_or_build()` 末尾按 `MARKET_ZSCORE_MODE` 决定是否调 `_recompute_market_zscore_rolling()` 重算 `features` 属性。

**行业中性化(FeaturePreprocessor 阶段 A)**(A12):对 PE/PB 在 `signed_log(x) = sign(x) * log(1 + |x|)` 之后、全局 z-score 之前,按每个行业分组做 median 减 + MAD 除。fit 时统计每行业的中位数和绝对中位差,transform 时应用同样的系数。

**工程简化**:由于每个样本是 60 天 lookback,严格做行业中性化需要对 d=0..59 每一天分别中性化。当前做法只对 d=0 做严格中性化,d=1..59 沿用同样的系数。短期内行业分布几乎不变,此近似误差远小于 stat-sampling 噪声。

**Transformer Backbone**:2 层 `TransformerEncoderLayer`,`d_model=128, nhead=4, dim_feedforward=512, activation=gelu, norm_first=True, dropout=0.3`。所有时间步共享同一编码器。

**Similar Retrieval(GPU 全量检索,bank 降级 + ★ A13 FP16 加速)**:bank 只存 360 维量价(v6 INPUT_SIZE=13 里的前 6 维)。query 也只用量价部分。**★ A13 加速**:bank 和 query 都用 FP16 做 matmul。top-k 对数值精度不敏感(top-k 序能选对即可)。显存减半后 batch 可以翻倍。FP16 下 `-1e9` 会 overflow 成 -inf,exclude-self 时用 `-1e4` 代替。

**理由(bank 降级)**:cosine 相似度假设所有维度同 modality。基本面/两融维度 scale 差异极大,即使归一化后语义方向也和量价完全不同。

**批量查表 BankCollator**:bank tensors 从 Dataset 中分离到独立的 `BankCollator` 类。Dataset 只存 `seq_x` + `neighbor_indices` + `y`。**★ A14+ 修正**:Dataset 再多存一个 `dates_t [N]` int64 tensor(date ordinals),`__getitem__` 返回 5 元组 `(x, neighbor_idx, y, market, date_id)` 用于 RankIC loss 的截面分组。BankCollator 的 `__call__(batch)` 同步改为 `zip(*batch)` 解 5 元素。训练/验证/推理三个 for-loop 都解 5 元组。

**Gate Input**:仅用离散 bucket 的 embedding(dim=16)做注意力加权。不使用连续标签值。

**GRU Expert**:8 个独立的 `nn.GRU(128, 128, num_layers=2, batch_first=True)`。每个样本被路由到 4 个 expert(top-4 soft mixture)。

**NoisyTopKGate**:对齐官方 `NoisyGate_VMoE`。固定噪声尺度 `noise_std / num_experts`,kaiming_uniform_ 初始化 `w_gate`。

**MoE 无残差**:直接返回,不经过残差连接。

**TRA 预测头**:`Linear(128, 64) → ReLU → Linear(64, 1)`,对齐官方 TRA(num_states=1)。

**SMA→EMA 参数平滑**:EMA 就地更新 `decay=1/SMA_WINDOW=0.2`。

**Shared Expert Isolation**:对齐 DeepSeekMoE。`NUM_SHARED_EXPERTS=1` 时,E0 作为 shared expert 始终激活,E1-E7 由 gate 做 top-3 路由。

**★ A14 Path 2 — Stock-hidden Regime Extractor φ_DM + FiLM/Concat Gate 调制**:RSAP-DFM(IJCAI 2024)借鉴机制。从 Transformer last_hidden `[B, 128]` 经 3 层 MLP(`Linear 128→32 + ReLU + Dropout 0.2 + Linear 32→16`)抽出 regime latent `z [B, 16]`。z 以两种模式之一进入 MoE gate:
- **film**:FiLM 调制 `gate_in' = γ·gate_in + β`,γ,β 从 `Linear(16, 2·gate_dim)` 得到;**初始化 γ=1, β=0**(zeros_(bias) + 给 γ 分支加 1),保证 USE_RSAP_DFM=True 的冷启动与 False 完全等价
- **concat**:拼接后投回 `gate_in' = Linear(concat([gate_in, z]), gate_dim)`
- **off**:z 学但不影响 gate_in(debug/ablation 模式;Path 3 启用时推荐用这个拆贡献)

注意 z 除了走 gate 调制,在 Path 3 Hierarchical MoE 下还会作为 L1 macro 路由的直接输入,所以 `"off"` 不等于"z 不做任何事"。

**★ A14 Path 2 — FGSM 对抗扰动(FP32 隔离 v3 版本)**:在训练的 adv 路径(USE_ADV_PERTURB=True 且 epoch >= ADV_WARMUP_EPOCHS=5):

1. Forward A 跑标准路径,拿到 `loss_main = MSE(pred, y) + GATE_AUX_WEIGHT·aux`(若开了 RankIC 则走 hybrid _make_loss)
2. `g, = torch.autograd.grad(loss_main, gate_in, retain_graph=True, create_graph=False)` 拿到 gate_in 处的梯度
3. **整段 FGSM 数学进 FP32 autocast 块**(v3 根治方案,v1 用 scaler.scale 错了、v2 只加 nan_to_num 还是炸):

```python
with torch.amp.autocast("cuda", enabled=False):
    g_fp32 = torch.nan_to_num(g.detach().float(), nan=0, posinf=0, neginf=0)
    g_fp32 = torch.clamp(g_fp32, min=-1e4, max=1e4)
    g_norm = g_fp32.norm(dim=-1, keepdim=True).clamp(min=1e-8)
    gate_in_adv = gate_in.detach().float() + ADV_EPSILON * (g_fp32 / g_norm)
    gate_in_adv = torch.nan_to_num(gate_in_adv, nan=0, posinf=0, neginf=0)
```

4. Forward B 走 `forward_moe_only(seq_hidden.detach(), gate_in_adv, z=z.detach())` 只跑 MoE + pred_head(seq_hidden.detach() 阻止 encoder 参与对抗梯度)
5. 合并 loss:`loss = loss_main + ADV_LOSS_WEIGHT·loss_adv`(joint loss,不做 bilevel 简化)

原始 A14 实现(v1)做了 4× encoder forward 峰值显存 ~2× 导致 4090 24GB swap,A14+ 修复后变成 1.3× baseline,符合论文级别开销。

**★ A14+ Path 3 — HierarchicalMoE(两层 MoE)**:借鉴 RSAP-DFM 的"双 regime shifting"思想应用到 MoE 层级。总 expert 数 `N_MACRO × N_MICRO = 2 × 4 = 8` 与 flat baseline NUM_EXPERTS=8 一致(运行时护栏强制)。结构:
- **L1 macro gate**(NoisyTopKGate(d_z=16, n_macro=2, top_k=HIERARCHICAL_MACRO_TOPK=1)):输入 regime z,top-1 硬路由,把每个样本分给一个 macro
- **L2 micro gates**(每 macro 一个独立 NoisyTopKGate(d_gate=16, n_micro=4, top_k=HIERARCHICAL_MICRO_TOPK=2)):输入 gate_inp,top-2 软路由
- **Micro experts 每 macro 独立不共享权重**
- **Aux loss = macro_aux + Σmicro_aux** 两级独立 cv_squared 平衡
- **Scatter-gather 分两层**:先按 macro mask 切子 batch,再在子 batch 内按 micro 分 expert 连续 dispatch
- **梯度流**:macro_gate 从 macro_gates[mask,m] 加权回传 + L1 aux,micro_gate 从 flat_gates 加权回传 + L2 aux
- **Forward signature**:SparseSequenceMoE.forward(seq_hidden, gate_inp, z=None) 忽略 z,HierarchicalMoE.forward(seq_hidden, gate_inp, z) 强制要 z
- **OfficialAlignedMERA.__init__** 按 USE_HIERARCHICAL_MOE 条件构造,运行时双重护栏(USE_RSAP_DFM 必须 True + HIERARCHICAL_N_MACRO×N_MICRO 必须等于 NUM_EXPERTS)
- forward / forward_train / forward_moe_only 全部 plumb z:forward_train 返回 7-tuple (pred, gates, aux, alpha, seq_hidden, gate_in, z)

**★ A17 — Helper 抽取后的 OfficialAlignedMERA 内部结构**:在 forward / forward_train / build_gate_input_with_regime 三处都共享同一段 `RETRIEVAL_ABLATION` 三分支(`"none"` / `"random"` / `"replace"`)+ `USE_RSAP_DFM` regime z 调制(film/concat/off 三种模式)的代码。A17 把这段共有逻辑抽到 `OfficialAlignedMERA._build_gate_in_modulated(last_hidden, similar, z=None) → (gate_in, z)`,三处调用点都改成调用此 helper,语义不变。`forward_train` 在 A17 之前是单方法 ~110 行,A17 之后纯组装。

**★ A17 — `_moe_dispatch_topk` 模块级 helper**:`SparseSequenceMoE.forward` 与 `HierarchicalMoE.forward`(L2 micro 阶段)共享同一段 scatter-gather:按 `dispatch_idx` 排序后 token,逐 expert 切片 forward,再按 inverse permutation 还原顺序并按 `gates` 加权求和。A17 抽到 `_moe_dispatch_topk(seq, dispatch_idx, gate_lookup_idx, gates, experts, expert_iter, dispatch_minlength)`,两个 MoE 类都改用此 helper。结果:`SparseSequenceMoE` ~~104~~ → 83 行,`HierarchicalMoE` ~~164~~ → 138 行,共省 ~70 行重复。

**★ A17 — 不动的边界**:抽 helper 时严格不动(项目原则,违反会导致回归):
- 顶部参数区(项目原则,所有可配置项一律硬编码在 .py 顶部)
- std/adv 双路 FGSM FP32 隔离训练循环主体(A14+ 写过血泪注释,改一行就可能踩 §15.7 的坑)
- val loop(早停依赖,口径全局统一)
- `FeaturePreprocessor.transform` 阶段变换内部(A13 in-place 修改的语义责任在调用方)
- 任何 public 名字(诊断脚本依赖 `setattr(_pipe, name, value)` 同步)
- `_save_window_checkpoint` 写入的 config dict 字段(checkpoint 反序列化向后兼容依赖)

**★ A14+ Path 4 — UMI RankIC Loss(零架构改动)**:KDD 2025 论文 Figure 1 公式 `Σ_i (ŷ-mean(ŷ))(z-mean(z))/std(ŷ)std(z)` 其中 z_t 为 rank 变换。

`rank_ic_loss(pred, target, date_ids)` module-level helper:
- argsort 两次得 rank stop_grad(argsort 不可导但作为常数靶子可用)
- rank 内部 z-score 到零均值单位方差
- pred 中心化保留梯度
- Pearson(pred, rank(target)) = (p_c * t_rank).mean() / p_std
- 截面分组 `for d in unique(date_ids): mask = (date_ids == d)` 每日独立 Pearson 后平均
- 小截面 RANK_IC_MIN_STOCKS=10 以下跳过
- FP32 强制 `with autocast(enabled=False)` 防 FP16 小方差 underflow

Val loop 保持纯 MSE+aux 不加 RankIC——val IC 本身就是 RankIC 的 Spearman 版本,把它加到 val loss 里会自证循环。

**关键约束**:
- 必须在 `autocast(enabled=False)` 块内计算,FP32 强制;FP16 下小方差截面 `p_c.std()` 会 underflow
- rank 变换必须 stop_grad(在 `torch.no_grad()` 块内算)
- 截面分组依赖 Dataset 返回 date_id;DailyBatchSampler 保证同日股票在同一 batch,但一个 batch 内通常有 `DAYS_PER_STEP=2` 天

**零架构改动**:与 Path 2 FGSM、Path 3 HierarchicalMoE 自由叠加(`_make_loss` 在两条路径 std/adv 都用)。开关 `USE_RANK_IC_LOSS` 默认 False,与 A13/A14 baseline 完全兼容。

### 3.3 v5 → v6 → ★ A13 修正变更记录

**A12(v5 → v6)**:

| 优先级 | 模块 | v5 | v6 | 依据 |
|:------:|------|-----|-----|------|
| M0 | 输入维度 | [60, 6] (FLAT=360) | [60, 13] (FLAT=780) | A12: 基本面/两融扩展 |
| M1 | 行业中性化 | — | PE/PB signed_log + industry MAD | A12 |
| M2 | 市场引导门控 | — | `α = 2·softmax(W·m)`, pre-BN | A12 MASTER AAAI 2024 |
| M3 | 检索 bank | 360 维量价 | 360 维量价(保持) | A12: 避免异构污染 |
| M4 | MarketFeatureProvider | — | 内联 pipeline(80 行) | A12 |
| M5 | ETL 集成 | — | qlib_converter + jq_sync 自动构建 | A12 |
| M6 | 输出目录 | Output/Qlib_DL/ | Output/Mera_Output/ | A12 |

**★ A13(v6 bug 修复 + 性能优化)**:

| 优先级 | 模块 | A12 v6 原版 | A13 修正版 | 依据 |
|:------:|------|-----|-----|------|
| M7 | `split_train_val` | `slice(0, 80%) / slice(80%, 100%)`(cross-sectional) | 按 `datetime` 时序切,返回 `np.ndarray` 索引 | val IC 0.37 vs OOS IC 0.02 = 17x 差距 |
| M8 | `volume` Stage A | 无变换 | `log1p(clip(x, 0, None))` | IPO/复牌首日 ratio=16900 撑大 std=200 |
| M9 | `roe` Stage A | 无变换 | `signed_log(x)` | raw 极端值 max=3230/min=-399 std=42 |
| M10 | `turnover_log` Stage A | `log1p(x * 100)` | `log1p(x)` | Qlib `$turnover_rate` 是百分比形式 |
| M11 | MASTER gate z-score | expanding | rolling 252 日 | α 训练期 σ=0.045 vs OOS σ=0.006 衰减 7x |
| M12 | `FeaturePreprocessor.transform` | out-of-place(2× 临时数组) | `inplace=True` reshape view + `np.subtract/divide/clip(out=)` | 省 ~5 GB RAM |
| M13 | `retrieve_topk_indices` | bank FP32 | bank FP16,exclude_self 用 `-1e4` | 3:33 → 1:30 |
| M14 | 诊断脚本主流程 | DataFrames + numpy 同时持有 | 提取 numpy → 立即 `del DataFrames + gc.collect()` | 释放 ~16 GB RAM |
| M15 | TF32 | 未启用 | `torch.backends.cuda.matmul.allow_tf32=True` | 训练 +20-40%,AMP 兼容 |

**★ A17(`qlib_pipeline_mera_v6.py` 真重复抽公因子 + 长函数瘦身)**:

| 优先级 | 模块 | A16 原版 | A17 重构版 | 依据 / 等价性证据 |
|:------:|------|---------|-----------|----------------|
| R1 | `train_mera_window_v5` | 560 行(纯训练 + pretrain 加载 + WIP resume + 最终 save 全部 inline) | 456 行(三个 IO 块外移到 helper,训练循环主体一行不动) | 函数瘦身只动纯组装 / 纯 IO,业务逻辑零变更 |
| R2 | `main()` | 160 行(配置打印 + window 主循环 + backtest stats 打印 + 五面板绘图 inline) | 31 行(只剩顶层调度) | 4 个组装/IO 块外移到 `_print_config_summary` / `_print_backtest_stats` / `_plot_five_panels` |
| R3 | `build_alpha360_expressions` | 140 行(60 行硬编码模板,六字段 × N 天逐字段写出表达式) | 69 行(13 字段 spec-table + `_ref(field, lag)` 单 helper + 单 loop) | `/home/claude/test_equiv.py` 跑 780 个表达式 + 780 个名字与原版**完全字符串一致** |
| R4 | `OfficialAlignedMERA.forward / forward_train / build_gate_input_with_regime` | 三处复制相同的 RETRIEVAL_ABLATION 三分支 + regime z 调制(共 ~90 行重复) | 共享 `_build_gate_in_modulated(last_hidden, similar, z=None)` | 仅抽公因子,语义不变 |
| R5 | `SparseSequenceMoE.forward` 与 `HierarchicalMoE.forward` 内 scatter-gather | 两处共复制 ~70 行排序-切片-加权-逆排序逻辑 | 模块级 `_moe_dispatch_topk(seq, dispatch_idx, gate_lookup_idx, gates, experts, expert_iter, dispatch_minlength)` | flat 与 hierarchical 语义统一 |
| R6 | `SparseSequenceMoE` | 104 行 | 83 行 | 调用 R5 helper |
| R7 | `HierarchicalMoE` | 164 行 | 138 行 | 调用 R5 helper |
| R8 | pretrain encoder 加载 | inline 在 `train_mera_window_v5` 头部 ~25 行(ckpt 加载 + 维度校验 + 关键 key 检查) | `_load_pretrained_encoder_into(model, window_id)` | 纯 IO 块,与训练循环解耦 |
| R9 | WIP 断点续跑 | inline 在 `train_mera_window_v5` 中段 ~20 行 | `_resume_wip_or_init(model, opt, scaler, sma_buffer, ckpt_wip_path)` | 纯 IO 块 |
| R10 | 最终 ckpt save | inline 在 `train_mera_window_v5` 尾部 ~30 行 | `_save_window_checkpoint(...)`(config dict 字段保持原版逐字一致,**禁止改字段名**:checkpoint 反序列化兼容性约束) | 纯 IO 块 |
| R11 | 重复 `import torch` | line 232 处的第二次 `import torch` | 移除(模块顶部已 import) | 零风险冗余 |
| R12 | `from qlib.data import D` 内层 import | `load_alpha360_data` 内 3 处重复 | 移到模块顶部 | 零风险冗余 |
| R13 | `_print_backtest_stats` helper | inline 在 `main()` 内 ~25 行 | 抽出,且函数注释里标注 §15.13 计划改 IR + beta 是 A17 后续待办 | A16 §15.13 已确认 Sharpe 单口径不足,要补 IR;此 helper 留为单点改造入口 |
| R14 | Helper docstring 是防回归文档 | — | 每个新 helper 加 10-15 行 docstring,说明 *为什么* 抽出 + *做什么* + 与原版语义对应关系 | 与历来 `★ A12 FIX` / `★ A14-fix-v3` 注释风格一致 |

**A17 行数账**:`qlib_pipeline_mera_v6.py` ~~3235~~ → 3325(+90)。**虚胖**——增量来自 helper docstring 与防回归注释,真函数复杂度大幅下降(560→456 / 160→31 / 140→69 / 164→138 / 104→83,合计减 339 行业务代码,加回 90 行 docstring + 7 个 helper 函数签名)。

**A17 不动的边界(承诺,违反则视为回归)**:
1. 顶部参数区:**禁动**
2. `train_mera_window_v5` 内 std / adv 双路 FGSM FP32 训练循环主体:**禁动**(A14+ §15.7 血泪教训)
3. `train_mera_window_v5` 内 val loop:**禁动**(早停 + ckpt 选择全局口径统一)
4. `FeaturePreprocessor.transform` 阶段变换内部(stage 0/A/B 数学):**禁动**
5. 任何 public 名字:**禁动**(诊断脚本 47 个 setattr 同步点)
6. `_save_window_checkpoint` 写入的 config dict 字段名:**禁动**(ckpt 反序列化兼容)

**A17 等价性验证脚本**(`/home/claude/test_equiv.py` 简版):

```python
# 仅对 build_alpha360_expressions 做字符串级逐字节比对
import importlib, importlib.util
def load_module(path, name):
    spec = importlib.util.spec_from_file_location(name, path)
    m = importlib.util.module_from_spec(spec); spec.loader.exec_module(m); return m

old = load_module("a16_backup/qlib_pipeline_mera_v6.py", "old")
new = load_module("a17/qlib_pipeline_mera_v6.py",        "new")

old_exprs, old_names = old.build_alpha360_expressions()
new_exprs, new_names = new.build_alpha360_expressions()

assert len(old_exprs) == len(new_exprs) == 780
assert len(old_names) == len(new_names) == 780
for i, (a, b) in enumerate(zip(old_exprs, new_exprs)):
    assert a == b, f"expr[{i}] mismatch: {a!r} vs {b!r}"
for i, (a, b) in enumerate(zip(old_names, new_names)):
    assert a == b, f"name[{i}] mismatch: {a!r} vs {b!r}"
print("OK 780/780 表达式 + 780/780 名字字符串完全一致")
```

**A17 待办(从 A16 继承 + 本阶段发现)**:
1. ~~**MERA 自监督预训练 v6**([60, 13] 版,写 `pretrain_mera_v6.py`)— A16 待办仍是首要任务~~ **★ A18 已完成**:文件名按命名规范定为 `mera_pretrain_v6.py`(非 `pretrain_mera_v6.py`)
2. **`_print_backtest_stats` 改 IR + beta 口径**(A16 §15.13 已计划,helper 注释里已标 A17 待办,实际还没改)— **A18 仍未做,继承到 A18 待办**
3. **实际跑一次单 window 验证**:精简后的 `qlib_pipeline_mera_v6.py` 数值要与 A16 baseline 一致(val IC 0.1087 / OOS daily IC 0.0358 / Sharpe 1.084)。等价性测试只覆盖了 `build_alpha360_expressions`,完整训练-推理路径需要实跑 — **A18 用户日志显示 W1 已 resume 老 ckpt,W2 进入新训练流程并触发 pretrain 加载,等价性侧面验证;完整数值复跑需重训整个 expanding window 系列**

**★ A18(`mera_pretrain_v6.py` 实质实现 + Pipeline 加载链路完整化)**:

| 优先级 | 模块 | A17 状态 | A18 实现版 | 依据 / 关键决策 |
|:------:|------|---------|-----------|----------------|
| P1 | `mera_pretrain_v6.py`(脚本本体) | ~~不存在(A16/A17 列首要待办)~~ | ✅ 1071 行,与 v5 `pretrain_mera.py` 完全独立并存 | A18 命名规范:除 `qlib_pipeline_*` 管线/`qlib_trading_*` 交易主入口外,所有 mera 资产以 `mera_` 起头 |
| P2 | `INPUT_SIZE` / `FLAT_DIM` | v5 pretrain 是 6 / 360 | 13 / 780,与 v6 pipeline 严格一致 | A12 [M0] 输入维度对齐 |
| P3 | `MeraEncoder.input_bn / input_proj` | `BatchNorm1d(6)` / `Linear(6, 128)` | `BatchNorm1d(13)` / `Linear(13, 128)` | A12 [M0] |
| P4 | `build_alpha360_expressions` | 6 字段 × 60 天 = 360 表达式 | 13 字段 × 60 天 = 780 表达式,pe/pb/roe/mtss 全部 `Ref(d+1)` 滞后防前瞻 | 与 pipeline `build_alpha360_expressions` 在结构上对齐(pretrain 自包含一份,不 import pipeline 避免循环依赖) |
| P5 | `FeaturePreprocessor` | v5 单阶段 z-score | v6 两阶段:Stage A 逐字段非线性变换(log_mv / signed_log(pe,pb,roe) / log1p(volume,turnover) / 行业中性化 PE/PB) + Stage B 全局 z-score clip [-3, 3] | A12 [M1] + A13 三项 scale 修复;**关键**:pretrain 与下游 fine-tune 必须看到同分布的输入(否则 BN running stats 拟合错误尺度) |
| P6 | `instruments` 数组传递链 | 不需要(单阶段 z-score 不查行业) | `main()` → `pretrain_one_window(instruments_train=...)` → `FeaturePreprocessor.fit/transform(instruments=...)` | A12 [M1] 行业中性化必需 |
| P7 | `SimilarProjector` 输入维度 | 360(v5 量价 = FLAT_DIM) | **360(v6 [M3] 检索 bank 设计选择,非 FLAT_DIM=780)** | **A18 关键工程细节**:v6 [M3] 决定检索 bank 只存量价,因此 pipeline `model.similar_proj` 是 `Linear(360, 128)`。pretrain 必须严格匹配 shape 才能被 `load_state_dict` 接受。**踩过坑**:第一版误写成 `Linear(FLAT_DIM=780, 128)`,被 user 截图警告纠正 |
| P8 | `PretrainDataset.flat_x` | v5 整体 360 维 | `seq_x[:, :, [0,1,2,3,4,5]].reshape(-1, 360)`(切量价 6 列,**非整体 780**) | 与 P7 配套:pretrain 训出的 `similar_proj` 权重 shape 必须是 `(360, 128)` |
| P9 | `cross_space_align_loss` | encoder([B,60,6]) ↔ similar_proj([B,360]) | encoder([B,60,**13**]) ↔ similar_proj([B,**360**]) | 异构对齐:encoder 看 13 维全量,similar_proj 看量价 6 维(360 = 60×6) |
| P10 | `augment_timeseries` mask 数量 | v3 mask 1-2 字段 | 保持 mask 1-2 字段(13 维下保持不变) | 13 维下 mask 0 = "该字段缺席",z-scored 后 0 等于中位数语义清晰;mask 比例提到 4-6 会破坏跨字段相关性 |
| P11 | ckpt 文件名 | `pretrain_encoder_v3_window_NN.pt` | `pretrain_encoder_v6_window_NN.pt` | v6 系列独立命名 |
| P12 | ckpt 内容 | encoder + sim_proj + preprocessor(med/mu/std) | encoder + similar_proj + preprocessor 完整两阶段 state(含 industry_map / pe_neutralize_stats / pb_neutralize_stats) + `version=6` + 完整 `config`(input_size / retrieval_flat_dim / field_names / use_industry_neutralize / align_weight) | A18 §1.2 新增原则"ckpt 必须带 version + config 做硬校验" |
| P13 | ckpt 输出目录 | `Output/Qlib_DL/mera_v3_pretrain` | `Output/Mera_Output/mera_v6_pretrain` | A12 [M6] 输出目录归一,所有 v6 资产进 `Mera_Output` |
| P14 | combined_state key 拼装 | `encoder.{k}` + `sim_proj.{k}` | `{k}` (encoder 部分原样) + `similar_proj.{k.replace("proj.", "")}` (映射 SimilarProjector 内层 nn.Sequential index 到 pipeline `model.similar_proj.X.weight` 命名) | **A18 关键**:错命名时 strict=False 会静默全 missing,**必须**在 pipeline 加载侧加显式 critical key 校验 |
| P15 | label_period | 与 pipeline DEFAULT_PERIOD 不必一致(pretrain 不用 label) | 设 `PT_LABEL_PERIOD=2` 与 pipeline DEFAULT_PERIOD 一致,**直接复用 v6 cache** | 工程优化:pipeline 跑过的 `alpha_cache_v6/alpha_features_p2_d13.pkl` pretrain 秒读,免重生成 36GB |
| P16 | Pipeline `_load_pretrained_encoder_into` | A17 实现为"警告 stub":只打印 v5 不兼容 v6,model 走默认初始化 | A18 实质实现:`PRETRAIN_DIR_V6 / f"pretrain_encoder_v6_window_{window_id:02d}.pt"` → 4 重校验(存在性 / version==6 / input_size==13 + retrieval_flat_dim==360 / 关键 key 真的不在 missing) → `model.load_state_dict(strict=False)` | A17 helper 抽象的设计在 A18 兑现 |
| P17 | Pipeline 顶部参数区 `USE_PRETRAINED_ENCODER` | A17 注释"v6 第一轮强制 False" | A18 注释"True=加载 mera_v6_pretrain 的 ckpt(含 encoder + similar_proj)";不再被覆盖 | A18 §1.2 命名规范"管线本体" |
| P18 | Pipeline 新增常量 `PRETRAIN_DIR_V6` | 不存在 | `OUTPUT_DIR / "mera_v6_pretrain"`,顶部 CHECKPOINT_DIR 旁定义 | 与 pretrain 脚本共享同一物理路径 |

**★ A18 第一次落地踩过的坑**(沉淀为 §15.24 方法论):
1. **OUTPUT_DIR 字面值不一致**:第一版 pretrain 写 `Output/Qlib_DL`(v5 老路径),pipeline 是 `Output/Mera_Output`。后果:cache 路径不匹配,pretrain 重新生成 36GB cache;ckpt 落到错路径,pipeline 找不到报 `FileNotFoundError`。修复:严格对齐到 `Output/Mera_Output`,顶部加注释 `# ★ 必须与 pipeline OUTPUT_DIR 一致`
2. **`SimilarProjector(flat_dim=FLAT_DIM)` 误用 780**:第一版误以为 pretrain 整套用 13 维全量,user 截图警告 v6 [M3] 检索 bank 是 360 维量价的设计选择。修复:`SimilarProjector(flat_dim=RETRIEVAL_FLAT_DIM=360)`,`PretrainDataset` 切出量价 6 字段
3. **文件命名 `pretrain_mera_v6.py` vs `mera_pretrain_v6.py`**:v5 时代命名风格 `pretrain_*` 是动词起头,user 提议改为模块起头 `mera_*` 与 `ensemble_*` 系列对应。修复:全文改名 + pipeline 5 处错误信息引用同步

### 3.4 v4 → v5 变更记录

| 优先级 | 修改 | v4 | v5 | 依据 |
|:------:|------|-----|-----|------|
| P0 | Expert 架构 | FFN | GRU(2L) | 官方 custom_moe_layer.py L218-230 |
| P1 | Gate 输入 | bucket + continuous label | bucket only | 官方 model_moe_attn.py L427-434 |
| P2 | MoE 残差 | residual + LayerNorm | 无 | 官方 FMoETransformerMLP.forward() |
| P3 | 预测头 | Linear(H,1) | MLP(H,64,1)+ReLU | 官方 TRA L470-476 |
| P4 | SMA | 无 | smooth_steps=5 | 官方 TRAModel.fit() L241-244 |
| Fix | expert 实例 | — | 独立实例化 | 修复官方 L224 共享引用 bug |
| Keep | similar_proj | ✓ | ✓ | 修复官方维度 mismatch bug |

### 3.5 与官方 MERA 的剩余差异

| 维度 | 官方 | v6(A13) | 理由 |
|------|------|-----|------|
| Gate 实现 | NoisyGate_VMoE (fmoe) | NoisyTopKGate (纯 PyTorch) | 避免 fmoe 依赖 |
| 训练框架 | TRAModel + fmoe 分布式 | 单文件 PyTorch | 适配 expanding-window 管线 |
| input_size | 351 (Alpha158 flat) | 780 (13 维 × 60 天) | A12 异构扩展 |
| 市场引导 | 无 | MASTER gate(★ A13: rolling z-score) | A12 |
| 行业中性化 | 无 | PE/PB industry z-score | A12 |
| ★ train/val split | 未公开 | **按 datetime 时序切(A13)** | A13 |
| MAX_STEPS_PER_EPOCH | 200 (未用) | 500 | A7 |
| n_epochs | 500 | 200 | |
| batch 组织 | 2 天 × ~500 只 | 2 天 × ~2000 只 | A6 |
| 优化器 | Adam | AdamW | |
| 预训练 | 有(未开源) | 首轮关闭 | A12 |
| 检索空间 | 预训练 representation | "raw"(首轮)| A12 |
| similar_proj | 无 | Linear+GELU+Dropout(360→128) | |
| DataLoader | num_workers=0 | num_workers=4 + BankCollator | A7 |

### 3.6 参数区完整说明(★ A13 v6 版)

```python
# ┌──────────────────────────────────────────────────────────────┐
# │  ★ v6 实验快切                                                │
# │  实验          TAG          MASTER  NEU    SPACE  SHARED      │
# │  ─────────── ──────────── ─────── ────── ─────── ──────       │
# │  v6 基线     "D_m13"       True    True   "raw"   0          │
# │  关 MASTER   "D_m13_ng"    False   True   "raw"   0          │
# │  关行业中性  "D_m13_nn"    True    False  "raw"   0          │
# │  Shared exp  "D_m13_s"     True    True   "raw"   1          │
# └──────────────────────────────────────────────────────────────┘
EXPERIMENT_TAG          = "D_m13"
USE_PRETRAINED_ENCODER  = False         # v6 首轮强制 False
RETRIEVAL_ABLATION      = None
RETRIEVAL_SPACE         = "raw"         # v6 首轮强制 "raw"
NUM_SHARED_EXPERTS      = 0
USE_MASTER_GATE         = True          # A12
USE_INDUSTRY_NEUTRALIZE = True          # A12

# ── 输入维度 ──
SEQ_LEN         = 60
PRICE_FIELDS    = ["open", "close", "high", "low", "vwap", "volume"]
EXTRA_FIELDS    = ["log_mv", "pe_signed_log", "pb_signed_log", "roe",
                   "turnover_log", "mtss_stock_rel", "mtss_flow_rel"]
FIELD_NAMES     = PRICE_FIELDS + EXTRA_FIELDS       # 13
INPUT_SIZE      = len(FIELD_NAMES)                  # 13
FLAT_DIM        = SEQ_LEN * INPUT_SIZE              # 780

RETRIEVAL_INPUT_SIZE = len(PRICE_FIELDS)            # 6
RETRIEVAL_FLAT_DIM   = SEQ_LEN * RETRIEVAL_INPUT_SIZE  # 360

MARKET_DIM      = 30

# ── ★ A13 新增:MASTER gate z-score 模式 ──
MARKET_ZSCORE_MODE = "rolling"       # "rolling" | "expanding"
MARKET_ROLLING_WINDOW = 252          # rolling 窗口天数(约 1 年交易日)
MARKET_ROLLING_MIN_PERIODS = 60      # 最少观测天数
MARKET_ZSCORE_CLIP = 5.0             # z-score clip 范围(±σ)

# ── 训练超参 ──
BATCH_SIZE = 4096
MAX_EPOCHS = 200
MAX_STEPS_PER_EPOCH = 500
PATIENCE = 20
EARLY_STOP_MIN_DELTA = 0.005
EARLY_STOP_SMOOTH = 5
VAL_EVERY = 2
LR = 1e-4
WEIGHT_DECAY = 1e-4
DROPOUT = 0.30
CLIP_NORM = 1.0
LABEL_CLIP = 3.0
VALID_RATIO = 0.15                   # ★ A13: 最后 15% 交易日为 val

# ── 模型架构 ──
HIDDEN_SIZE = 128
NUM_LAYERS = 2
NUM_HEADS = 4
NUM_EXPERTS = 8
TOP_K_EXPERTS = 4
GATE_DIM = 16
GRU_EXPERT_LAYERS = 2
GATE_AUX_WEIGHT = 0.01
PRED_HEAD_DIM = 64
SMA_WINDOW = 5

# ── 检索 ──
N_NEIGHBORS = 50
SIMILAR_BATCH = 2048
LABEL_BUCKETS = 10

DEFAULT_MODEL = "MERA_V6"
```

---

## 三B、实验设计与 Expert / MASTER 分化诊断

### 3B.1 实验矩阵

**v5 实验矩阵(保留供对比)**:

| 实验 | TAG | PRETRAIN | ABLATION | SPACE | SHARED | AUX | 说明 |
|------|-----|:--------:|:--------:|:-----:|:------:|:---:|------|
| A 组 | `"A"` | False | None | `"raw"` | 0 | 0.01 | v5 基线 |
| B 组 | `"B"` | True | None | `"raw"` | 0 | 0.01 | v5 预训练 + raw 检索 |
| C 组 | `"C"` | False | None | `"encoder"` | 0 | 0.01 | v5 无预训练 + encoder 检索 |
| D 组 | `"D"` | True | None | `"encoder"` | 0 | 0.01 | **v5 论文完全体** |
| D_shared | `"D_shared"` | True | None | `"encoder"` | 1 | 0.01 | v5 + Shared Expert Isolation |
| D_aux001 | `"D_aux001"` | True | None | `"encoder"` | 0 | 0.001 | v5 + aux loss 降 10 倍 |

**v6 实验矩阵**:

| 实验 | TAG | MASTER | NEU | SPACE | SHARED | AUX | 说明 |
|------|-----|:------:|:---:|:-----:|:------:|:---:|------|
| **v6 基线** | `"D_m13"` | True | True | `"raw"` | 0 | 0.01 | 13 维 + MASTER + 行业中性 |
| **关 MASTER** | `"D_m13_ng"` | False | True | `"raw"` | 0 | 0.01 | 验证 MASTER 贡献 |
| **关行业中性** | `"D_m13_nn"` | True | False | `"raw"` | 0 | 0.01 | 验证行业中性化贡献 |
| **Shared exp** | `"D_m13_s"` | True | True | `"raw"` | 1 | 0.01 | |
| **6 维退化** | `"D_m13_6d"` | — | — | `"raw"` | 0 | 0.01 | 禁用扩展字段,等价 v5 A 组 |

### 3B.2 A11 诊断结果:A / D / D_shared 三组对照(v5 单窗口)

训练区间 2013-2018,测试 2019 单年,窗口 1。

| 指标 | A | D | D_shared |
|---|---|---|---|
| val_IC | 0.389 | 0.306 | 0.318 |
| **OOS IC** | **0.047** | **0.091** | **0.087** |
| gate_entropy_ratio | 0.644 | 0.640 | **0.528** (分母 log(7)) |
| gate_variance_mean | 0.00923 | 0.01061 | **0.02143** |
| expert output cosine mean | — | 0.157 | 0.141 |
| expert weight L2 mean | — | 43.23 | 43.28 |
| gate usage span (routed) | 0.078–0.092 | 0.062–0.120 | 0.075–0.134 |
| 日均 IC | — | 0.038 | 0.041 |
| daily IC 正比例 | — | 73.8% | 77.0% |

**A11 核心发现**:
1. D 组比 A 组 OOS IC 翻倍(0.047 → 0.091),同时 val_IC 更低。预训练 + encoder 检索的主要贡献是**减少过拟合**
2. D_shared 的 expert 分化最强,但 OOS IC 和 D 组持平
3. 三组都存在明显的"坍缩轨迹":Ep 0 时 gate 有显著分化,训练 10-30 epoch 后收敛到均匀。aux loss 是坍缩元凶

### 3B.3 ★ A13 v6 首轮诊断数据(bug 定位关键证据)

**单窗口诊断参数**:TRAIN 2015-01-01 → 2018-12-31,TEST 2019-01-01 → 2025-12-31,窗口 1,`D_m13`。

**修复前(A12 原版)指标实测**:

| 指标 | 数值 | 判读 |
|---|---|---|
| val IC | **0.3706** | 异常虚高(比 v5 同配置的 0.024 高 15 倍) |
| OOS IC (pooled) | **0.0218** | 比 v5 A 组(6 维纯量价)的 0.047 还低 |
| daily_ic_mean | 0.0182 | 低于合格线 0.038 |
| daily_ic_ir | 5.552 | IR 虚高 |
| val / OOS 差距 | **17 倍** | 严重异常 |
| MASTER α 训练期 σ | 0.045 | 训练时 gate 有学到东西 |
| MASTER α OOS 期 σ | **0.006** | OOS 时 gate 几乎瘫痪 |
| Expert gate_entropy_ratio | 0.652 | 正常 |
| top1 active experts | 8/8 | 所有 expert 都被用到 |

**定位过程(A13 工程教训)**:

1. **先怀疑数据泄漏**。写"数据对齐验证脚本":选 002222 福晶科技,对三段时间窗口(2020-01 平静期 / 2020-02 疫情开市 / 2023-06 两融期),手算 label 和所有新加的 7 维 feature,和 Qlib `D.features()` + JQ 原始 CSV 三源对齐。**结论:无前瞻泄漏**
2. **怀疑 FeaturePreprocessor fit 数据范围**。Code review 无全局 fit 问题
3. **怀疑 split_train_val 切分方式**。打印 tr_slice / va_slice 的日期范围和股票集合。**铁证**:日期交集占 va 比例 **100.0%**(tr/va 时间段完全重叠),股票交集占 va 比例 **0.4%**(仅 1 只横跨切分点)
4. **连锁怀疑 Feature scale**。写"13 维 scale 诊断脚本"检查所有维度 Stage 0/A/B 的 p01/p50/p99/clip 率。发现:
   - `volume`:raw max=16900,std 撑到 ~200,z-score 后 p99=0.084 信号被吞
   - `roe`:raw max=3230/min=-399,std=42,99% 正常 ROE 样本 z-score 后挤在 ±0.5 之间
   - `turnover_log`:Stage 0 原始 p50=1.427(远大于 1),确认是百分比形式。原代码 `log1p(x*100)` 过度压缩
5. **最后怀疑 MASTER α 在 OOS 的衰减**。改 rolling 252 日

### 3B.4 ★ A14+ Path 2 W1 部署后实测数据(单窗口,v3 FGSM 修复完整跑完)

**Config**:D_m13_rsadv,`USE_RSAP_DFM=True`, `USE_ADV_PERTURB=True`, `REGIME_Z_TO_GATE="film"`, `REGIME_D_Z=16`, `ADV_EPSILON=0.01`, `ADV_WARMUP_EPOCHS=5`, `ADV_LOSS_WEIGHT=0.3`。单窗口 W1(train 2015-2018,test 2019-2025)。

**阈值判定对照 §7.7 priority table**:

| 指标 | Baseline D_m13 (A14) | Path 2 W1 | §7.7 Path 2 过线阈值 | 判定 |
|---|---:|---:|---:|:---:|
| val IC | 0.065 | **0.0690** | — | ✅ 改善 +0.004 |
| daily IC (pooled) | 0.0226 | 0.0232 | ≥ 0.025 | ⚠️ 未过 0.025 阈值 |
| daily IC IR | — | **6.61** | — | ✅ 极强稳定信号 |
| Sharpe | 0.898 | **1.034** | ≥ 1.0 | ✅ 过阈值 +0.14 |
| 年化收益 | — | 24.7% | — | — |
| Max DD | — | **-30.8%** | — | ❌ 项目最差(risk_analysis.csv -35.1% 计算口径不同) |
| 超额 vs 国证2000 | — | +121.5% / 6.7y ≈ 18.1%/y | — | — |
| 换手(年均化) | — | 113.5% | — | — |
| α σ(MASTER gate) | 0.008 | **0.036** | — | ✅ 4.5× 意外改善 |

**Path 2 成功结论**:两大阈值(Sharpe、val IC)达标,**部署成功**。但 2024 年 daily IC 出现严重衰退。

**分年分解(Path 2 vs A13 baseline)**:

| 年份 | A13 baseline daily IC | Path 2 daily IC | Δ | Path 2 Sharpe | Path 2 年收益 |
|---|---:|---:|---:|---:|---:|
| 2019 | 0.014 | 0.022 | +0.008 | 1.62 | +40.8% |
| 2020 | 0.021 | 0.026 | +0.005 | 1.22 | +31.6% |
| 2021 | 0.020 | 0.019 | -0.001 | 1.66 | +35.1% |
| 2022 | 0.019 | 0.022 | +0.003 | **0.054** | -1.8%(基准崩得更惨,超额 +18.6%) |
| 2023 | 0.021 | 0.019 | -0.002 | 0.62 | +8.3% |
| **2024** | **0.033** | **0.0185** | **-0.015 (-44%)** ❌ | **0.72** | +19.2% |

<!-- ░░░ A16 原文行 806-895 在 v-A17 重建过程中暂缺 — Path 2 W1 ① / Path 2 W1 ② 复现详细分年表 / 融合版 D_m14_p2_p3_p4 W1 实测分年数据(关键摘要见 §3B.5/§15.6) ░░░ -->

*<small>v-A17 重建说明:此处 A16 原文 806-895 行内容因长上下文压缩缺失,如需追溯请参考 `A16_Fusion_Triple_Stack_v2.md` 原始备份。涉及主题:Path 2 W1 ① / Path 2 W1 ② 复现详细分年表 / 融合版 D_m14_p2_p3_p4 W1 实测分年数据(关键摘要见 §3B.5/§15.6)。</small>*

2. **2023 反而退步**:Path 2 W1 ② Sharpe 0.49 → 融合版 -0.05,是融合版的副作用
3. **所有年份 daily IC 全部改善**,没有任何一年比 Path 2 W1 ② 差
4. **2025 Sharpe 1.94 + 2019 Sharpe 2.05 + 2021 Sharpe 2.12** 三个高分年份说明融合版在大盘上行环境下能力很强

**Sharpe 1.08 看似一般的真正原因(A16 新发现)**:

策略相对中证 1000 的 corr = **0.976**, beta = **0.94**,说明策略大部分仓位跟着基准走。剥掉 beta 算残差 IR:

```
策略 daily_std = 1.523%
基准 daily_std = 1.583%
剥 beta 后残差 std = 0.332%/日
日 alpha = 0.040%/日 ≈ 10.2%/年
信息比率 IR = 1.93   ← 这是真实 alpha 能力
```

**Sharpe 1.08 是"绝对收益 vs 无风险利率"口径,分母里塞了大量基准 beta 波动**。换 IR 口径后 1.93 是优秀对冲基金水平。这意味着:
1. 真实 alpha 信号比 Sharpe 反映的强得多
2. 项目离"做空对冲产品形态"还有一道工程墙(IF/IC 期货对冲),过墙后 Sharpe 直接进入 2.0+ 区间
3. §7.7 阈值 "Sharpe ≥ 1.0" 应补充 "或 IR ≥ 1.5",否则未来策略提升分散度(降 beta)会反而看起来 Sharpe 退步

**Sharpe 计算口径不一致问题(A16 发现)**:

`mera_single_window_diag_v6.py::_run_single_window_backtest` 的整体 Sharpe(L680-684)用**混合口径**:
```python
ann_ret = cum ** (1/years) - 1     # 几何年化(带 volatility drag 折扣)
ann_vol = result_df['net_ret'].std() * sqrt(252)   # 算术年化
sharpe  = ann_ret / ann_vol
```
分子几何分母算术,波动越大分子被打折越多,Sharpe 被人为压低。**分年 Sharpe**(L716)用标准算术口径 `mean × √252 / std`,两口径不一致。

口径对比(融合版数据):

| 口径 | Sharpe | 用途 |
|---|---:|---|
| A 混合(diag 整体) | 1.084 | 当前 JSON 报的值 |
| B 算术(diag 分年同口径) | ≈1.12 | 推荐统一 |
| C 对数收益 | ≈1.02 | 学术标准 |

**待修**:把整体 Sharpe 改成口径 B(与分年统一),同时 backtest_stats 加 `beta_to_benchmark` / `info_ratio` / `alpha_annual_pct` / `residual_std_annual_pct` 四个字段。

### 3B.7 ★ A16 基准对齐(universe ↔ benchmark 配对)

**A16 之前的状况**:`qlib_pipeline_mera_v6.py` 默认 `UNIVERSE_CSV = "cons_csi1000.csv"`(中证 1000),但 BENCHMARK 通过 `from qlib_pipeline_native import BENCHMARK as NATIVE_BENCHMARK; BENCHMARK = NATIVE_BENCHMARK` 继承 native 默认 SZ399303(国证 2000)。**主 pipeline 实际跑出的回测一直在用错基准**。

**diag 脚本的另一种自洽错配**:`mera_single_window_diag_v6.py::_PARAMS_TO_SYNC` 把 diag 自己定义的 `UNIVERSE_CSV = "cons_gz2000.csv"` 覆盖到 pipeline 模块,所以 diag 实际跑的 universe 是 gz2000。这套 universe 与 SZ399303 基准是配对的,所以文档里所有 W1 数字(A13 baseline / Path 2 W1 ① / Path 2 W1 ② / HB Sweep)**都是 gz2000 + SZ399303 同口径自洽**。

**A16 决策**:用户偏好"主策略和 diag 都对齐 mera_v6 的目标 universe csi1000",所以全面切换。

**修改清单**:

| 文件 | 改动 |
|---|---|
| `qlib_pipeline_native.py` | `run_qlib_backtest()` 新增 `benchmark=None` 参数,None 时退回模块默认 BENCHMARK,**不破坏 native 自身行为** |
| `qlib_pipeline_mera_v6.py` | `BENCHMARK = "SH000852"`(不再继承 native),调用 `run_qlib_backtest(...benchmark=BENCHMARK)`,绘图 label/title 改"中证1000" |
| `mera_single_window_diag_v6.py` | 顶部参数区第 2 节加 `UNIVERSE_CSV / BENCHMARK` 强配对,加运行时 assert;BENCHMARK 加进 `_PARAMS_TO_SYNC` |

**配对约束代码**(A16):

```python
_UNI_BENCH_PAIRS = {
    "cons_csi1000.csv": "SH000852",
    "cons_gz2000.csv":  "SZ399303",
}
if UNIVERSE_CSV in _UNI_BENCH_PAIRS:
    _expected = _UNI_BENCH_PAIRS[UNIVERSE_CSV]
    assert BENCHMARK == _expected, (
        f"\n[配对错误] UNIVERSE_CSV='{UNIVERSE_CSV}' 应配 BENCHMARK='{_expected}', "
        f"当前 BENCHMARK='{BENCHMARK}'。请回顶部参数区第 2 节同时修正两者。")
```

**对历史数据的处理**:A14+ 文档所有 W1 数字(D_m13 / D_m13_rsadv ① / Path 2 W1 ②)都是 gz2000 + SZ399303 同口径自洽,**不重跑**,但所有"超额 vs 基准"数字必须标注 "vs SZ399303 国证2000 口径"。A16 起所有新实验用 csi1000 + SH000852 口径。

**跑前数据可用性检查(必做)**:

```python
from qlib.data import D
import qlib
from project_config import PROJECT_ROOT
qlib.init(provider_uri=str(PROJECT_ROOT / "qlib_data" / "cn_jq"), region="cn")
df = D.features(["SH000852"], ["$close"], start_time="2019-01-01", end_time="2025-12-31", freq="day")
print(df.shape, df.dropna().shape)   # 期望 ~1700 行非空
```

如果空 / 报错 → SH000852 数据未下载,先用 `qlib_converter.py` 补充。否则 mkt_ret 会全 0 而你不知道。

### 3B.8 ★ A16 参数区分层整合

**A16 之前的问题**:`mera_single_window_diag_v6.py` 顶部参数区 50 多个变量混在一起,A14+ 加 Path 3 / Path 4 后 EXPERIMENT_TAG 拼接逻辑变成嵌套 if(`_rs` + `adv` + `_h2x4` + `_ric`),用户记不清当前 tag 是什么。

**A16 整合方案(9 段分层 + Path X tag 命名)**:

| 段 | 内容 | 改不改 |
|---|---|---|
| 1 必改区 | EXPERIMENT_TAG + 4 个 path 开关 + DIAG_TAG | **每次实验都看** |
| 2 数据域 | UNIVERSE_CSV ↔ BENCHMARK 强配对(运行时 assert) | 切换 universe 时 |
| 3 时间窗 | TRAIN/TEST 起止 | 切窗口时 |
| 4 偶尔改 | MASTER gate / 行业中性 / share / pretrained / RUN_BACKTEST | 消融实验 |
| 5 通用 | DEFAULT_PERIOD / HB / MODEL / RETRIEVAL_SPACE | 几乎不动 |
| 6 数据维度 | SEQ_LEN / FIELDS / FLAT_DIM | 改 → 缓存失效 |
| 7 训练超参 | LR / batch / epochs / patience | A14+ 已稳定 |
| 8 模型架构 | HIDDEN_SIZE / NUM_EXPERTS / TOP_K | A14+ 已稳定 |
| 9 Path 子参数 | regime_d_z / hierarchical / rank_ic 权重 | 默认已最优 |

**Path X tag 命名(A16 新规范)**:

| 当前实验 | Path X tag | 旧式 tag |
|---|---|---|
| baseline 全关 | `D_m14_base` | ~~D_m13~~ |
| Path 2 完整 | `D_m14_p2` | ~~D_m13_rsadv~~ |
| Path 2 仅 regime | `D_m14_p2reg` | ~~D_m13_rs~~ |
| Path 3 only | `D_m14_p3` | ~~D_m14_h2x4~~ |
| Path 4 only | `D_m14_p4` | ~~D_m14_ric~~ |
| 融合版三件套 | **`D_m14_p2_p3_p4`** | ~~D_m14_rsadv_h2x4_ric~~ |

启动后控制台第一行就能看到 `实验: D_m14_p2_p3_p4`,一眼知道开了什么。

### 3B.9 ★ A16 Resume 逻辑统一到 diag

**问题**:`qlib_pipeline_mera_v6.py` main loop L2941 早就有 resume 逻辑(checkpoint 存在 → 跳过训练直接加载),但 `mera_single_window_diag_v6.py::train_single_window` **没有**这套逻辑。融合版 W1 训完后诊断阶段崩溃(HierarchicalMoE 嵌套 ModuleList 不兼容旧诊断函数),导致整个 ~7h 训练成果差点报废。

**A16 修复**:

1. **diag 同步 resume 逻辑**(L530 之前加 ckpt_path.exists() 检查,与 pipeline L2941 完全一致),下次 diag 阶段崩溃可直接重跑跳过训练
2. **诊断函数加 `_get_expert(moe, e_idx)` helper** 兼容 HierarchicalMoE:

```python
def _get_expert(moe, e_idx):
    if hasattr(moe, "micro_experts"):     # HierarchicalMoE
        macro_idx = e_idx // moe.n_micro
        micro_idx = e_idx % moe.n_micro
        return moe.micro_experts[macro_idx][micro_idx]
    else:                                   # SparseSequenceMoE
        return moe.experts[e_idx]
```

原来 L380 / L436 直接 `model.moe.experts[e_idx]` 改用 helper

### 3B.10 v5 D 组与 v6 首轮对比

| 指标 | v5 D 组(7 窗口 OOS) | v6 A13 首轮(单窗口 OOS,预计) |
|---|---|---|
| daily_ic_mean | 0.038 | 目标 ≥ 0.038 |
| OOS IC (pooled) | 0.091 | 目标 ≥ 0.06 |
| MASTER α OOS σ | N/A | 目标 ≥ 0.03 |
| val IC | 虚高(按股票切误导) | 目标 ~0.03-0.08(时序切真实值)|

### 3B.11 v6 A13 关键阈值表

| 指标 | v5 D 组基线 | v6 A13 合格 | v6 A13 优秀 | 说明 |
|---|---|---|---|---|
| `oos_daily_ic_mean` | 0.038 | ≥ 0.038 | ≥ 0.048 | A 股标准指标 |
| `oos_daily_ic_ir` | ~1.5 | ≥ 1.8 | ≥ 2.5 | daily IC mean / std × √252 |
| `val_oos_ratio` | N/A | ≤ 3× | ≤ 2× | **A13 新增** |
| `gate_entropy_ratio` | 0.6-0.8 | < 0.9 | < 0.7 | 分化程度 |
| `active_experts` | 8/8 | ≥ 6/8 | ≥ 6/8 | 至少作为 top-1 出现过的 expert 数 |
| `alpha_stats.alpha_sum_of_mean` | N/A | 1.8 ~ 2.2 | 1.95 ~ 2.05 | 理论 = 2.0 |
| `alpha_stats.alpha_std_overall` | N/A | > 0.05 | > 0.15 | α 跨样本 std |
| **`alpha_stats.alpha_std_oos`** | N/A | **≥ 0.03** | **≥ 0.05** | **A13 新增** |
| **每维度 Stage B clip 率** | N/A | **所有维度 < 5%** | **所有维度 < 3%** | **A13 新增** |

### 3B.12 分化指标体系(A11 + A12 + ★ A13)

`mera_single_window_diag_v6.py::diagnose_expert_differentiation()` 输出指标:

**A11 原有 6 项(Expert 分化)**:

| # | 指标名 | 含义 | 越大=分化 vs 越小=分化 |
|---|---|---|---|
| 1 | `gate_usage` | 每个 expert 的平均 gate score | 差异大=分化,易受平均掩盖 |
| 2 | `gate_entropy_ratio` | 样本层面分布的平均 entropy / log(E) | 越小越分化 |
| 3 | `gate_variance_mean` | 跨样本 gate scores 方差 | 越大越分化 |
| 4 | `expert_output_cosine_mean` | expert 输出 pair-wise cosine | 越小越分化 |
| 5 | `expert_weight_l2_mean` | expert 参数向量 pair-wise L2 | 越大越分化 |
| 6 | `top1_distribution` / `active_experts` | top-1 分布 + active 数 | hard 层面的多样性 |

**A12 新增 1 项(MASTER α 诊断)**:

| # | 指标名 | 判断 |
|---|---|---|
| 7 | `alpha_stats` | sum_of_mean ≈ 2.0 → 数学正常;std_overall > 0.05 → gate 真在学东西 |

**★ A13 新增 2 项**:

| # | 指标名 | 判断 |
|---|---|---|
| 8 | `stage_b_clip_ratios` | 任一维度 >5% → 重尾分布未压缩 |
| 9 | `val_oos_ratio` | ≤ 3× 正常;> 3× 说明 split bug 或严重过拟合 |

`alpha_stats` 完整字段:

```python
{
    "alpha_mean_per_field": [13 个值],      # 每个输入字段在测试集上的 α 均值
    "alpha_std_per_field": [13 个值],       # 每个输入字段 α 的跨样本 std
    "alpha_sum_of_mean": float,             # 应接近 2.0
    "alpha_std_overall": float,             # 13 个维度的 std 取均值
}
```

**判断逻辑**:
- `alpha_sum_of_mean` 明显偏离 2.0(<1.8 或 >2.2):MASTER gate 数学 bug
- `alpha_std_overall < 0.01`:MASTER gate 没学到东西
- `alpha_std_overall > 0.15`:MASTER gate 正常工作


### 3B.13 ★ A13 新增诊断工作流

修复任何疑似 bug 之前,必须按以下顺序走:

1. **数据对齐验证**(~2 分钟):用 `align_check.py` 三源对齐手算 label + feature
2. **Split 切分验证**(~5 分钟):用 `verify_split.py` 打印 tr_slice / va_slice 的日期范围和股票集合重叠率
3. **Feature scale 扫描**(~30 秒):用 `verify_feature_scale.py` 打印所有维度的 Stage 0/A/B 分位数 + clip 率
4. **RAM 分阶段探测**(~20 秒):用 `ram_probe.py` 在每个关键步骤打印 RSS + 系统内存
5. **最后才动架构 / 参数**

### 3B.14 实验运行顺序(★ A13 v6 更新)

1. 跑 ETL 一次生成 market_features_v6.pkl:`python etl/qlib_converter.py`
2. **★ A13: 删 v6 checkpoint**(A12 原版 checkpoint 的 best_val_ic 是按股票切的虚假值)
3. **v6 基线 D_m13 单窗口诊断**:验证 val IC 回到 0.03-0.08 区间,OOS IC ≥ 0.04 起步;MASTER α OOS σ ≥ 0.03
4. **A11 消融**(如 D_m13 合格):D_m13_ng + D_m13_nn
5. 归因:
   - 如果 D_m13 >> D_m13_ng → MASTER gate 有效
   - 如果 D_m13 >> D_m13_nn → 行业中性化有效
   - 如果 D_m13 ≤ D_m13_6d → 7 维扩展本身失败
6. **正式 pipeline**(如 D_m13 达优秀):`python qlib_pipeline_mera_v6.py`(~10-14h)
7. **后续路径**:重写 `pretrain_mera_v6.py` 支持 [60, 13] / D_m13_s

---

## 四、标签定义

**标签公式**(A8 修正,A13 验证):`label = Ref($open, -(N+1)) / Ref($open, -1) - 1`

当前默认:N=2(ret_2d)。标签预处理:z-score → clip ±3σ → NaN 置零。

**★ A13 验证**:数据对齐脚本在 002222 福晶科技 2020-01-02、2020-01-03 两行手算验证:
- 2020-01-02:`open_{t+1}=12.65`, `open_{t+3}=12.76`,手算 `12.76/12.65-1=0.008696`,与 pipeline 的 `label_manual=0.00869572` 完全一致
- 2020-01-03:`open_{t+1}=12.54`, `open_{t+3}=12.76`,手算 `12.76/12.54-1=0.01754`,与 pipeline 的 `label_manual=0.0175439` 一致

标签严格用 T+1 买入 / T+3 卖出的开盘价,T 日及之前数据一点不碰。不是 label 泄漏。

---

## 五、回测框架

### 5.1 HBTopKStrategy

继承 Qlib `BaseSignalStrategy`,完全重写 `generate_trade_decision()`。持仓股加 HB → 停牌检查 → 锁仓 → Top-K → 生成订单。

### 5.2 Qlib backtest() API

```python
portfolio_metrics, indicator = qlib_backtest(
    start_time=test_start, end_time=test_end,
    strategy=strategy_instance, executor=executor_config_dict,
    benchmark="SZ399303", account=2e7,
    exchange_kwargs={
        "open_cost": 0.001, "close_cost": 0.002,
        "min_cost": 5.0, "deal_price": "open",
        "limit_threshold": 0.095,
    },
)
```

### 5.3 成本参数

| 参数 | 值 |
|------|-----|
| OPEN_COST | 0.001 |
| CLOSE_COST | 0.002 |
| MIN_COST | 5.0 |
| TRADE_UNIT | 100 |
| DEAL_PRICE | "open" |
| LIMIT_THRESHOLD | 0.095 |

### 5.4 共享函数(`qlib_pipeline_native.py`)

| 函数 | 功能 |
|------|------|
| `select_top_k` | HB 加分 → 停牌锁仓 → Top-K |
| ~~`find_st_stocks`~~ | ~~ST/退市股识别~~ **★ A19 已删除**(主策略 Phase 53 重构) |
| ~~`filter_ipo`~~ | ~~排除上市不满 1 年的股票~~ **★ A19 已删除**(主策略 Phase 53 重构) |
| `apply_qualified_safety` | **★ A19 新增引用**:合规保险层(替代 `apply_st_safety`) |
| `_apply_qualified_mask` | **★ A19 新增引用**:NaN-mask label 列(C' 方案,只 mask label 不 mask 因子) |
| `run_qlib_backtest` | 统一回测入口(`listing_dates` 参数已删除) |
| `get_qualified_universe(date)` | **★ A19 新增引用**(`etl/jq_sync`):返回当日合规股票列表 |
| `get_qualified_mask(...)` | **★ A19 新增引用**(`etl/jq_sync`):返回整段合规矩阵 |

### 5.5 Expanding Window 训练

每个窗口:训练截止 = 测试开始前 1 天。测试长度 = 252 天。窗口间无重叠。

**★ A13 修正**:每个窗口内部的 train/val 切分方式从原版"按行序切"改为"按 datetime 时序切"。最后 VALID_RATIO=0.15 比例的 unique 交易日作为 val 集。

### 5.6 ★ 如何运行 MERA 管线

**v5 pipeline**:`qlib_pipeline_mera_v5.py`。输出 → `Output/Qlib_DL/`。

**v6 pipeline**(A12 主入口,A13 修正后,**★ A18 加预训练步骤**):`qlib_pipeline_mera_v6.py`。

**前置条件**:
1. ETL 数据就绪(含 mtss_fin_value / mtss_fin_buy .bin)
2. `industry_map.pkl` 存在于 `Dataset2006JQ/`
3. `Output/Mera_Output/market_features_v6.pkl` 存在(由 ETL 自动生成)
4. **★ A13: 如从 A12 原版升级,必须删 `Output/Mera_Output/mera_v6_checkpoints/` 整个目录重跑**

**★ A18 推荐运行顺序(预训练 + pipeline 两步)**:

**第一步**:跑预训练(只需要做一次,或当 expanding window 配置改变时重跑)
```
PyCharm 打开 mera_pretrain_v6.py → 右键 → Run
```
- 读 `Output/Mera_Output/alpha_cache_v6/alpha_features_p2_d13.pkl`(pipeline 已生成的 cache,秒读)
- 为每个 expanding window 输出 `Output/Mera_Output/mera_v6_pretrain/pretrain_encoder_v6_window_NN.pt`
- 单 window pretrain ~30-60 分钟(RTX 4090,PT_EPOCHS=100,PT_MAX_STEPS=200/epoch)
- 8 windows 全跑约 4-8 小时
- 已存在的 window ckpt 会跳过(`if ckpt_path.exists(): skipping`),只补缺的
- WIP 断点续跑:每 10 epoch 保存到 `*_wip.pt`,中途崩溃下次从 epoch 9/19/29... 继续

**第二步**:跑 pipeline
```
PyCharm 打开 qlib_pipeline_mera_v6.py → 右键 → Run
```
- 顶部参数区 `USE_PRETRAINED_ENCODER = True`(默认)→ 训练每个 window 时自动加载对应 pretrain ckpt
- 加载日志样:`[Pretrain] ✓ Loaded v6 ckpt: pretrain_encoder_v6_window_02.pt | best_loss=4.2317 | loaded=XX/YY keys | missing=N unexpected=0`
- `unexpected=0` 是必须的(>0 = pretrain 端 key 命名错);`missing>0` 是正常的(market_gate / moe / pred_head / regime_extractor 等下游模块本来就不在 ckpt 里)
- 临时关闭预训练做 baseline 对照:参数区 `USE_PRETRAINED_ENCODER = False`

**v5 vs v6 预训练分流**:
- v5 pretrain `pretrain_mera.py` → `Output/Qlib_DL/mera_v5_pretrain/` → `qlib_pipeline_mera_v5.py` 加载
- v6 pretrain `mera_pretrain_v6.py` → `Output/Mera_Output/mera_v6_pretrain/` → `qlib_pipeline_mera_v6.py` 加载
- 完全独立,两个版本可并存

**断点续跑**:每个 window 训练完保存到 `Output/Mera_Output/mera_v6_checkpoints/{TAG}/ret{N}d/mera_v6_window_XX.pt`。再次运行时自动加载跳过。注意:**resume 老训练 ckpt 时不会再走 `_load_pretrained_encoder_into`,所以 W1 已 resume 的窗口不会用 A18 新 pretrain ckpt**(这是正确语义:resume 已经包含训练完成的权重)。如果想强制让所有 window 用新 pretrain,删掉 `mera_v6_checkpoints/` 整个目录重跑。

**运行时间**:RTX 4090,~10-14 小时 / 7 窗口(不含 pretrain)。**★ A13 预期缩短到 ~7-10 小时**。**★ A18 加 pretrain 后总时长 ~12-16 小时(pretrain 4-8h 一次性 + pipeline 7-10h)**。

**输出位置**:

| 输出 | 路径 |
|------|------|
| 回测结果 | `Output/Mera_Output/MERA_V6_ret2d_hb0.00_{TAG}/` |
| pred_score | 上述目录下 `pred_score.csv` |
| Checkpoint | `Output/Mera_Output/mera_v6_checkpoints/{TAG}/ret{N}d/mera_v6_window_XX.pt` |
| **★ A18 Pretrain Checkpoint** | **`Output/Mera_Output/mera_v6_pretrain/pretrain_encoder_v6_window_NN.pt`** |
| Alpha 缓存 | `Output/Mera_Output/alpha_cache_v6/` |
| 市场特征 pkl | `Output/Mera_Output/market_features_v6.pkl` |

### 5.7 ★ 如何运行单窗口诊断

**v5 诊断**:`mera_single_window_diag_v2.py`(A11)。输出 → `Output/Qlib_DL/diag_W{ID}_{TAG}/`。

**v6 诊断**(A12 主入口,A13 修正后):`mera_single_window_diag_v6.py`。

**核心设计**:
- `import qlib_pipeline_mera_v6 as _pipe`,参数区 `setattr(_pipe, ...)` 同步
- 单窗口训练
- 6 项 Expert 分化指标 + 1 项 MASTER α 分析 + ★ A13: Stage B clip 率 + val/OOS ratio
- `predict_and_analyze()` 输出分年 IC / daily_ic_mean / IR

**A12 同步参数**:`USE_MASTER_GATE`, `USE_INDUSTRY_NEUTRALIZE`, `PRICE_FIELDS`, `EXTRA_FIELDS`, `RETRIEVAL_INPUT_SIZE`, `RETRIEVAL_FLAT_DIM`, `MARKET_DIM`。Diag 脚本启动时重置 `_pipe.FIELD_IDX = {name: i for i, name in enumerate(FIELD_NAMES)}`。

**★ A13 新增同步参数**:`MARKET_ZSCORE_MODE`, `MARKET_ROLLING_WINDOW`, `MARKET_ROLLING_MIN_PERIODS`, `MARKET_ZSCORE_CLIP`。

**★ A13 新增流程**:main 函数提取 numpy array 后立即 `del DataFrames` + `gc.collect()`(释放 ~16 GB)。训练完后立即 `del train_flat_raw` + `torch.cuda.empty_cache()`。诊断脚本启动时启用 TF32。

**输出位置**:`Output/Mera_Output/diag_v6_W{ID}_{TAG}/`

**`diag_summary.json` 结构(★ A13 扩展)**:

```json
{
  "version": "v6",
  "experiment": "D_m13",
  "window": 1,
  "train": "2015-01-01→2018-12-31",
  "test": "2019-01-01→2025-12-31",
  "val_ic": ...,
  "ic_stats": {
    "overall_ic": ..., "daily_ic_mean": ..., "daily_ic_std": ...,
    "daily_ic_ir": ..., "yearly_ic": {...}, "yearly_daily_ic": {...}
  },
  "input_size": 13,
  "retrieval_flat_dim": 360,
  "market_dim": 30,
  "use_master_gate": true,
  "use_industry_neutralize": true,
  "market_zscore_mode": "rolling",
  "market_rolling_window": 252,
  "gate_entropy_ratio": ...,
  "gate_variance_mean": ...,
  "gate_usage": [...],
  "expert_output_cosine_mean": ...,
  "expert_weight_l2_mean": ...,
  "top1_distribution": [...],
  "top1_entropy_ratio": ...,
  "active_experts": 8,
  "num_routed": 8,
  "alpha_stats": {
    "alpha_mean_per_field": [13 个值],
    "alpha_std_per_field": [13 个值],
    "alpha_sum_of_mean": 1.99,
    "alpha_std_overall": 0.11,
    "alpha_std_train": 0.045,
    "alpha_std_oos": 0.038
  },
  "stage_b_clip_ratios": {
    "open": 0.0202, "close": 0.0000, "high": 0.0203,
    "low": 0.0229, "vwap": 0.0193, "volume": 0.0008,
    "log_mv": 0.0069, "pe_signed_log": 0.0207,
    "pb_signed_log": 0.0096, "roe": 0.0012,
    "turnover_log": 0.0026, "mtss_stock_rel": 0.0215,
    "mtss_flow_rel": 0.0213
  },
  "val_oos_ratio": 1.53
}
```

**运行时间**:2015-2018 训练 + 2019-2025 测试,约 40-60 分钟。**★ A13 预期缩短到 20-30 分钟**。

---

## 六、交易管线

### 6.1 qlib_trading.py(主策略 XGB/LGB)

加载训练好的 XGB 模型 → 当天因子 → 预测 → HB+选股 → target_holdings.csv。

**qlib_trading.py 自动包含 market_features 重建**(A12):内部调 `jq_sync.main()`,jq_sync 的 `run_qlib_conversion()` 末尾会自动触发 `rebuild_market_features_v6()`。**trading 脚本本身无需修改**。

持久化日志:trade_journal / holdings_history / daily_summary / holdings_state.json。

### 6.2 qlib_trading_mera.py(MERA v5,保留)

加载 MERA v5 checkpoint → 当天 Alpha360 截面 → `predict_mera_window_v5()` → `select_top_k()` → target_holdings.csv。

**备注**:当前 v5 trading 管线仍用 Alpha360 [60, 6] 输入。未来 v6 IC 验证通过后,需要独立的 `qlib_trading_mera_v6.py` 支持 13 维输入 + 市场特征加载。暂未实现。


<!-- ░░░ A16 原文行 1305-1396 在 v-A17 重建过程中暂缺 — 诊断工作流细则 / §四 ETL 管线相关内容 ░░░ -->

*<small>v-A17 重建说明:此处 A16 原文 1305-1396 行内容因长上下文压缩缺失,如需追溯请参考 `A16_Fusion_Triple_Stack_v2.md` 原始备份。涉及主题:诊断工作流细则 / §四 ETL 管线相关内容。</small>*


| 原文组件 | 本项目实现 | 差异说明 |
|---|:---:|---|
| (a) GRU 特征提取 | ✅ 用 Transformer 代替 | 同类不同实现 |
| (b) stock-hidden regime extractor | ✅ 借用 | 用 3 层 MLP |
| (c) Dual regime encoder | ❌ 跳过 | 原文最大 novelty,无 DFM 方程可挂 |
| (d) Multi-head attention prior 因子 VAE 采样 | ❌ 跳过 | 无 factor 抽象 |
| (e) 双 DFM 主方程 | ❌ 跳过 | MoE 预测范式不是 DFM |
| (f) FGSM on λ_prior | ✅ 借用但对象改 | 扰动对象从 `λ_prior` 换成 `gate_in`,**语义类比不是同构** |
| (g) Bilevel 优化 | ❌ 跳过 | 改用 joint loss |

**代码可用性验证**:RSAP-DFM **无开源代码**(awesome-time-series-forecasting 等多个 curated list 里 RSAP-DFM 条目只标 "Paper")。本项目实现完全基于论文文字描述重建,**与原文可能 20-30% 偏差**。

**对外沟通定性**:正确表述是"借用 RSAP-DFM 的 stock-hidden regime 和 FGSM 两个概念作为 MERA v6 的增量组件,在 MoE 范式下重组"。论文未来若发表引用,应明确写 "RSAP-DFM-inspired" 而非 "RSAP-DFM applied"。

**和 MASTER gate 的关系**:
- MASTER gate 输入是**外部市场向量** m
- RSAP-DFM 的 φ_DM 输入是**内部 stock hidden**
- 两者语义互补:前者是"真实世界的市场状态",后者是"模型眼里的市场状态"。v6 Path 2 首次把两者同时用上

#### 7.1c.5 HireVAE(IJCAI 2023)— Hierarchical 两层 latent

**架构**:
1. **Market encoder**:从 multimodal market features 抽 market latent factor
2. **Market-aware stock encoder**:conditional encoder
3. **Regime-switching module**:对 market latent 做聚类(N_b 个 regime),按距离选 regime
4. **Regime-specific decoder**:每个 regime 有独立 decoder

**Online adaptive**:moving-average online learning,regime 聚类中心随时间更新。

**和 v6 关系**:HireVAE 是**完全不同的架构**(DFM/VAE 范式 vs v6 MoE 范式)。"market-aware stock encoder"思路可以部分借鉴(更深度融合),与 Path 2 的 FiLM 调制思路相通但更激进。

#### 7.1c.6 FactorVAE(AAAI 2022)— Prior-Posterior DFM+VAE

**Prior-posterior 机制**:
1. Prior encoder:只看历史(t 之前)
2. Posterior encoder:额外能看未来 label
3. Training:KL 约束 prior 向 posterior 逼近
4. Inference:只用 prior encoder

**和 v6 关系**:完全不同范式,不直接借鉴。但 prior-posterior 思路有启发——可以训练时让 regime encoder 同时输入 past + future 做 teacher,推理时只用 past 做 student。

#### 7.1c.7 UMI / Irrationality(KDD 2025)— 双层不理性因子

**Stock-level 机制**(技术上最精巧):
1. **Cointegration attention**:用 attention 找每只股票 u 的 cointegrating peers,产出 estimated rational price `p̃_t = Σ β_u · p_peer_t`
2. **Stationary regularization**:残差 u_t 必须是 stationary 过程
3. **u_t 作为 stock-level irrationality factor**

**Market-level 机制**(全自监督):
1. **Sub-market comparative learning**:InfoNCE-like loss
2. **Market synchronism prediction**:预测同一时刻内市场内所有股票同步涨跌

**Forecasting 头**:Transformer + graph-attention 混合 + **RankIC loss**

**和 v6 关系**:
- A13 §十二.5 把这篇归入"路径 4 加新输入维度",但 UMI 其实**不是简单加字段**
- Market-level 两个自监督任务可以和 MERA 的 contrastive pretraining 合并成一个自监督阶段
- **RankIC loss 是零架构改动、可立即叠加的改进点**——把 v6 的 MSE loss 换成 RankIC loss。ROI 极高

#### 7.1c.8 DeepSeekMoE(ACL 2024)— 已实现

A10 已实现 `NUM_SHARED_EXPERTS` 参数。A11 测过 `D_shared` 对照,routed experts 分化最强(entropy ratio 0.528 vs D 组 0.640)但 OOS IC 持平 D 组。**单层 Shared Expert Isolation 在 v6 已到极限**——如果要再进一步,需要结合 Path 2 的 regime latent z 做 hierarchical MoE。

#### 7.1c.9 DHMoE(AAAI 2025)— 未读

只看过标题"Diffusion + Hierarchical MoE"。不进路线图。

### 7.2 官方 MERA 源码结构

```
MERA/src/
├── model_moe_attn.py       # Transformer + MoE (attention gate)
├── model_moe_vote.py       # Transformer + MoE (vote gate)
├── custom_moe_layer.py     # FMoETransformerMLP (GRU experts)
├── noisy_gate.py / noisy_gate_vmoe.py
└── dataset.py
```

### 7.3 官方代码已知问题

1. **Expert 共享引用 bug**(custom_moe_layer.py L218-224):所有 expert 共享同一 GRU 实例。v5 已修复
2. **Similar 维度 mismatch**(model_moe_attn.py L428):`bmm(x[-1][B,1,H], similar_feat[B,K,351].T)` 当 H≠351 时失败。v5 通过 `similar_proj(360→H)` 修复
3. **TRA 在 num_states=1 时退化为 2 层 MLP**
4. **gate_aux_weight config vs 代码不一致**:config `lamb=1.0`,代码硬编码 `0.01`
5. **config.max_steps_per_epoch=200 在代码中未使用**

### 7.4 MERA 论文未报告的细节(A11 观察)

论文展示 MoE 版本比非 MoE 版本 IC/RankIC 更高,但**没有报告 gate usage 分布、expert 分化程度、expert 输出差异**。结合官方代码的 expert 共享引用 bug,官方实际报告的 SOTA 是"8 个事实上共享的 expert"下拿到的。追求 expert 分化是在"超越官方未实现的论文描述"。

### 7.5 MASTER 论文借鉴(A12 + ★ A13 修正)

**v6 借鉴方式**:
- **保留**:α = 2 · softmax(·) 的数学形式
- **保留**:pre-BN 应用位置
- **修改**:m_t 维度从 MASTER 原版的 63 维简化为 30 维(3 指数 × 10 统计量)。原因:v6 Universe 是国证 2000,板块内部差异小于 MASTER 原 Universe,63 维会引入噪声维度
- **不用**:MASTER 的 intra-stock + inter-stock 交替注意力架构

**★ A13 补充修正**:MASTER 原论文未明确 m_t 归一化方式。A12 v6 首轮用 expanding z-score 在 OOS 上失败。A13 改为 rolling 252 日 z-score。论文如果真用 expanding,很可能在长回测期也会遇到同样问题,只是论文报告测试期较短(通常只有 1-2 年)不明显。

### 7.6 ★ A14: MASTER 借鉴程度再评估 + Path 1 上限不确定性下调

A14 读过原文后,确认 MASTER 五步中 v6 只做了第 1 步约 30% 总贡献。inter-stock attention(第 3 步)是论文的**核心 novel 贡献**,论文 ablation 对比的是 DTML 等完全不同架构,**没有直接报告"只做 gating vs 做完五步"的分步 ablation**。因此:

**A13 原话**:"路径 1 预期 IC +0.01-0.02"——**A14 下调为 +0.005-0.015,可信度中低**。

### 7.7 ★ A14: 顶刊路径优先级重排(基于真实论文机制)

| 路径 | 来源论文 | 预期上限 | 可信度 | ★ A14+ 实际状态 | 核心依据 |
|---|---|---|---|---|---|
| ~~**MERA 自监督预训练([60, 13] 版)**~~ **MERA 自监督预训练([60, 13] 版,★ A18 已实现)** | MERA WWW 2025 | daily IC 0.023 → 0.03-0.04, Sharpe 0.9 → 1.3 | **高** | ~~未实现(用户偏好排到最后)~~ **★ A18 已实现 `mera_pretrain_v6.py`,等待实跑数据验证** | A11 在 v5 [60,6] 上实测 OOS IC 从 0.047 → 0.091 |
| **Path 2: RSAP-DFM 借鉴版** | RSAP-DFM IJCAI 2024 简化版 | +0.005-0.03 IC | 中 | ✅ **A14+ 已 deploy 至 W1**。val IC 0.065→0.069 / daily IC 0.0226→0.0232 / Sharpe 0.898→1.034 过阈值。但 **2024 daily IC 从 0.033 衰退到 0.0185** 创项目最大 DD -30.8% | 已按原文核对。**A14+ 纠正定性为"启发式借鉴"** |
| **UMI RankIC Loss** | UMI KDD 2025 | +0.005-0.015 IC | 中 | ✅ **A14+ 已实现 + A16 已并入融合版**(融合版 W1 daily IC +73% vs Path 2 W1 ②) | 零架构改动 |
| **★ A14+ 新增 — Path 3: Hierarchical MoE** | RSAP-DFM 双 regime 启发 + DeepSeekMoE | 未估,预期针对 2024 衰退场景 | 中 | ✅ **A14+ 已实现 + A16 已并入融合版**(融合版 W1 2024 daily IC 0.013→0.028 +118% 救回) | Path 2 W1 发现 E4=0.009 死 expert |
| **UMI Stock-level irrationality factor** | UMI KDD 2025 | +0.005-0.02 IC | 中 | 未实现。工程风险:cointegration attention 跨 ~2000 股票显存 ~60GB | 需要独立的自监督前置训练 |
| **UMI Market-level 自监督** | UMI KDD 2025 | 和 MERA pretraining 合并评估 | 中 | 未实现 | sub-market comparative + synchronism prediction |
| **Path 1: MASTER inter-stock attention** | MASTER AAAI 2024 第 3 步 | +0.005-0.015 IC | 中低 | 未实现。工程风险:显存 ~60GB,batch 结构 1 天 × ~2000 与 Path 2 冲突 | 论文 ablation 未报告单项贡献 |
| **Path 4: 加行业 one-hot / 北向资金 / 换手异动** | KDD 2025 输入维度扩展 | +0.01-0.02 IC | 中 | 未实现 | ROI 高但不 novel |
| **HireVAE 借鉴** | HireVAE IJCAI 2023 | 未估 | 低 | 未实现 | DFM/VAE 范式与 v6 MoE 不兼容 |
| **FactorVAE 借鉴** | FactorVAE AAAI 2022 | 未估 | 低 | 未实现 | prior-posterior 思路对 regime encoder 有启发但工程量大 |
| **StockMixer backbone 替换** | StockMixer AAAI 2024 | 未估,不推荐 | 低 | 未实现 | 工程量大收益不明确 |
| **DHMoE** | AAAI 2025 | 未读原文 | 极低 | 未实现 | 不进路线图 |

**A14 关键决策**(A14+ 延续):

1. **MERA 自监督预训练**:A14+ 调整为用户偏好把预训练排最后,先做其他组件
2. **UMI 拆成两部分分别评估**:RankIC loss(零架构改动)已实现待测;stock-level cointegration 因显存墙暂缓
3. **Path 1 MASTER inter-stock 降级 + 暂缓**:不确定性高 + 和 Path 2 batch 结构冲突
4. **★ A14+ 新增决策 — Path 3 Hierarchical MoE 的引入**:W1 Path 2 部署后 2024 衰退 + 死 expert,直接催生 Hierarchical MoE
5. **★ A14+ 新增决策 — 融合版优先于单变量 ablation**

**★ A14+ 实验矩阵**(A16 全部改 Path X):

| ~~A14+ 旧 tag~~ | **A16 Path X tag** | USE_HIERARCHICAL_MOE | USE_RANK_IC_LOSS | REGIME_Z_TO_GATE | 测试目的 | A16 实测状态 |
|---|---|:---:|:---:|:---:|---|:---:|
| ~~D_m13_rsadv~~ | **D_m14_p2** | F | F | "film" | Path 2 baseline | ✅ W1 跑两次 |
| ~~D_m14_rsadv_ric~~ | **D_m14_p2_p4** | F | **T** | "film" | RankIC 单独增益 | 未测 |
| ~~D_m14_rsadv_h2x4~~ | **D_m14_p2_p3**(z 单通道) | **T** | F | "off" | Path 3 纯净版 | 未测 |
| ~~D_m14_rsadv_h2x4_film~~ | **D_m14_p2_p3**(z 双通道) | **T** | F | "film" | Path 3 + z 双通道 | 未测 |
| ~~**D_m14_rsadv_h2x4_ric**~~ | **D_m14_p2_p3_p4** | **T** | **T** | "film" | **融合版(首测推荐)** | ✅ **A16 W1 已跑** |

---

## 八、Preprocessing Pipeline

### 8.1 Feature Preprocessing — v5 原版

1. 中位数填充 NaN
2. inf 检测与替换(fit 填完 NaN 后用 `~np.isfinite()`)
3. z-score 标准化
4. Clip ±3σ
5. 转 float32

### 8.1.X ★ A13 修正后的 Feature Preprocessing(v6)

**阶段 A(A13 修正版,共 7 项变换)**:

1. **`volume`(★ A13 新增)**:`log1p(clip(x, 0, None))`。原 ratio 不做 log 压缩直接 z-score,std 撑到 ~200,99% 正常样本 z-score 后挤在 ±0.08。`log1p` 压缩后 ratio=1 → 0.69,ratio=10 → 2.4,ratio=100 → 4.6
2. **`log_mv`(继承 A12)**:`log(clip(x, 0, None) + 1)`
3. **`pe_signed_log` / `pb_signed_log`(继承 A12)**:`sign(x) * log(1 + |x|)`
4. **`pe_signed_log` / `pb_signed_log`(继承 A12)**:行业中性化(industry median 减 + MAD 除)
5. **`roe`(★ A13 新增)**:`signed_log(x)`。原版 raw max=3230 / min=-399 撑大 std 到 42,99% 正常 ROE 样本 z-score 后挤在 ±0.5
6. **`turnover_log`(★ A13 修正)**:`log1p(clip(x, 0, None))`(**去掉原 `* 100`**)。Qlib `$turnover_rate` 实际是百分比形式 3.665
7. **`mtss_*`(继承 A12)**:NaN → 0 填充

**阶段 B(全局 z-score + clip,★ A13 实现改 in-place)**:

```python
def transform(self, x_flat, instruments=None, inplace=False):
    x_stage_a = self._transform_stage_a(x_flat, instruments=instruments, ..., inplace=inplace)
    if inplace and x_stage_a.dtype == np.float32:
        vals = x_stage_a.reshape(x_flat.shape)  # view, 不新分配
    else:
        vals = x_stage_a.reshape(x_flat.shape).astype(np.float32, copy=True)
    bad_mask = ~np.isfinite(vals)
    if bad_mask.any():
        vals[bad_mask] = np.take(self.med, np.where(bad_mask)[1])
    np.subtract(vals, self.mu, out=vals)      # ★ A13 原地操作
    np.divide(vals, self.std, out=vals)        # ★ A13 原地操作
    np.clip(vals, -3.0, 3.0, out=vals)         # ★ A13 原地操作
    return vals
```

**pipeline 调用方式**:

```python
# train 阶段
pre = FeaturePreprocessor().fit(train_flat_raw[tr_slice], instruments=tr_instruments)
x_all = pre.transform(train_flat_raw, instruments=train_instruments_all, inplace=True)
# train_flat_raw 原地被改写,raw 值不再可用

# test 阶段
pre = FeaturePreprocessor(**art.preprocessor_state)
x_test = pre.transform(test_flat_raw, instruments=test_instruments, inplace=True)
```

**节省内存量**:一次 transform 原本产生 2× 输入体积的临时数组。`inplace=True` 节省约 5-8 GB RAM。

**注意事项**:
- `inplace=True` 要求输入必须是 `float32` 且 `c_contiguous`
- 调用方承担"raw 值不再使用"的语义责任
- `inplace=False`(默认)保留原版行为,兼容诊断脚本里的多次 transform 调用


### 8.2 Label Preprocessing

1. z-score 标准化
2. Clip ±3σ
3. NaN 置零

### 8.3 Similar Retrieval(★ A13 FP16 加速)

1. 训练集特征 L2 归一化,构建 bank
2. **★ A13 修正**:Bank 一次性上传 GPU,**用 FP16(`torch.float16`)而非 FP32**
3. Query 分批 1024-2048 个上传,GPU 矩阵乘法(FP16 matmul 吞吐翻倍)+ `torch.topk(k=50)`
4. 只返回索引 [N, 50] int64
5. `collate_fn` 批量查表拼接
6. 训练时排除自身(**★ A13: FP16 下用 `-1e4` 而非 `-1e9` 标记排除,防 overflow**)

**v6 差异**(A12):bank 只存 360 维量价(前 6 维)。query 用 `extract_price_flat(x_all)` 提取前 6 维。13 维中的基本面/两融部分(后 7 维)不参与检索但参与模型 forward。

**★ A13 性能收益**:OOS 推理阶段的检索步骤从 3:33 缩短到 ~1:30,提速约 2.4 倍。精度损失可忽略(top-50 邻居中 >95% 与 FP32 版本一致)。

---

## 九、运行环境

### 9.1 硬件

| 组件 | 规格 |
|------|------|
| OS | Windows 11 (10.0.26200) |
| CPU | Intel Core (14th gen), 24 核 |
| RAM | 128 GB DDR5 |
| GPU | NVIDIA GeForce RTX 4090 24GB |

### 9.2 软件栈

| 软件 | 版本 |
|------|------|
| Python | 3.13.11 |
| PyTorch | 2.6.0+cu124 |
| CUDA | 12.4 |
| cuDNN | 9.1.0 |
| Qlib | pip install qlib |
| IDE | PyCharm |

### 9.3 注意事项

- 必须禁用 torch.compile/dynamo
- gymnasium 替代 gym
- `C["kernels"] = min(cpu_count, STOCK_BATCH, 16)`
- Pipeline 保留 `cudnn.benchmark=True`;诊断脚本 False
- **★ A13: 启用 TF32**(`torch.backends.cuda.matmul.allow_tf32 = True` + `torch.backends.cudnn.allow_tf32 = True`),4090 训练 +20-40%,AMP 兼容
- **★ A13: 训练期 RAM 峰值**:修复前 ~100 GB(含 Memory Compression 触发),修复后 ~70 GB

---

## 十、项目目录结构(★ A13 更新)

```
PROJECT_ROOT/
├── project_config.py                    # 项目级路径
├── qlib_pipeline_mera_v5.py            # v5 研究入口(保留不动)
├── qlib_pipeline_mera_v6.py            # ★★ A12: v6 研究入口(★ A13 + A14+ Path 2/3/4 + ★ A17 helper 抽取 + ★ A18 _load_pretrained_encoder_into 真实加载,~3340 行)
├── mera_single_window_diag_v2.py       # v5 单窗口诊断(A11)
├── mera_single_window_diag_v6.py       # ★★ A12: v6 单窗口诊断(★ A13 + A14+)
├── pretrain_mera.py                     # v5 自监督对比学习预训练([60, 6])保留不动
├── ~~pretrain_mera_v6.py~~ **mera_pretrain_v6.py**  # ★★ A18: v6 自监督对比学习预训练([60, 13],1071 行,SimCLR + cross-space align)
├── mera_hb_grid_search.py               # HB Grid Search
├── hb_sweep_v2.py                      # ★ A14: HB Sweep v2(复用 pipeline run_qlib_backtest)
├── qlib_pipeline_native.py              # XGB/LGB 主策略(共享函数来源,★ A16: run_qlib_backtest 加 benchmark 参数)
├── qlib_trading.py                      # 主策略交易管线
├── qlib_trading_mera.py                 # v5 MERA 交易管线
├── gm_strategy.py                       # 掘金执行策略
├── inspect_mera_checkpoint.py           # checkpoint 诊断
├── exposure_constraint.py               # Momentum 漂移约束
├── etl/
│   ├── jq_config.py                     # 数据同步配置(A12: MTSS_FIELDS)
│   ├── jq_sync.py                       # A12: jq_sync run_qlib_conversion 自动 hook rebuild_market_features_v6
│   ├── qlib_converter.py                # A12: JQ CSV → Qlib .bin + rebuild_market_features_v6
│   └── qlib_factors.py                  # 手工因子定义
├── MERA/src/                            # 官方 MERA 源码参考
├── qlib_data/cn_jq/                     # Qlib .bin 数据(A12 新增 mtss_fin_value/mtss_fin_buy)
├── tools/                               # ★ A13 新增: 诊断与验证脚本目录
│   ├── align_check.py                   # 数据对齐验证
│   ├── verify_split.py                  # split_train_val 切分方式验证
│   ├── verify_feature_scale.py          # 13 维 feature scale 诊断
│   └── ram_probe.py                     # RAM 分阶段探测
├── Output/
│   ├── Qlib_DL/                         # v5 输出(保留不动)
│   │   ├── mera_v5_checkpoints/
│   │   ├── mera_v5_pretrain/             # v5 pretrain ckpt(`pretrain_mera.py` 写入)
│   │   ├── alpha360cache/
│   │   ├── MERA_V5_ret2d_*_{TAG}/
│   │   └── diag_W{ID}_{TAG}/
│   ├── Mera_Output/                     # ★★ A12: v6 输出(独立命名)
│   │   ├── mera_v6_checkpoints/{TAG}/ret2d/   # 训练 ckpt
│   │   ├── **mera_v6_pretrain/**             # **★ A18: 预训练 ckpt(`mera_pretrain_v6.py` 写入)**
│   │   │   └── pretrain_encoder_v6_window_NN.pt
│   │   ├── alpha_cache_v6/                    # pretrain 与 pipeline 共享(p2_d13.pkl)
│   │   ├── market_features_v6.pkl       # ★ A12: ETL 自动生成
│   │   ├── MERA_V6_ret2d_*_{TAG}/
│   │   └── diag_v6_W{ID}_{TAG}/
│   └── ...                              # 其他主策略输出
└── Dataset2006JQ/
    ├── MarketData/{code6}.{SZ|SH}.CSV   # JQ 原始 CSV(★ A13 大写扩展)
    ├── cons_gz2000.csv                  # 国证 2000 成分股
    ├── cons_csi1000.csv                 # 中证 1000 成分股
    ├── benchmark_gz2000.csv
    ├── industry_map.pkl                 # ★ A12: 行业静态快照(pretrain 也读它做行业中性化)
    └── ...
```

---

## 十一、关键设计决策与教训

### 11.1 v5 阶段累积教训(精简,详见 A1-A11 历史文档)

**★ MERA 三大贡献(A1)**:MoE+GateNet 动态路由 / Retrieval 用 representation 找 pattern 邻居 / 自监督预训练。本项目 2/3 实现。

**★ Expert 共享引用 bug(A1)**:官方 `custom_moe_layer.py L218-224` 所有 expert 共享同一 GRU 实例。v5 已修复(独立实例化)。

**★ Similar 维度 mismatch(A1)**:官方 `model_moe_attn.py L428` H≠351 时失败。v5 通过 `similar_proj(360→128)` 修复。

**★ Universe 大小决定训练成本(A3)**:全市场 → 成分股,样本 330 万 → 100 万,每 epoch 7-8 min → 2 min。

**★ Instrument 大小写敏感(A4)**:`D.features()` MultiIndex 的 instrument 是大写(`SH600000`/`SZ000001`)。frozenset 查找大小写敏感。

**★ AMP + collate_fn 加速训练(A4)**:每 epoch 299s → 120s。

**★ TORCHDYNAMO_DISABLE 必须在 import torch 之前设(A4)**:Windows 无 Triton。

**★ gate_aux_weight 0.01 是官方硬编码(A5)**。A11: 仍是坍缩元凶。

**★ SimCLR 预训练 loss 正常收敛范围(A6)**:batch=8192 时随机 baseline = log(2B-1) ≈ 9.70,Window 1 收敛到 ~3.8。

**★ load_model_from_artifacts 必须 strict=True(A6)**。

**★★ 训练结束后必须释放 GPU 模型(A7)**。

**★ BankCollator 解决 num_workers>0 + 大 bank 矛盾(A7)**:Dataset 只存轻量索引,bank 在 collate_fn 里查表。

**★ MAX_STEPS_PER_EPOCH=500 对后期大窗口加速 ~2.5x(A7)**。

**★ EXPERIMENT_TAG 实现实验隔离(A7)**。A11: 做对照实验时必须用新 TAG。

**★ Qlib 并行加载 kernels 必须限制上限(A7)**:`C["kernels"] = min(cpu_count, STOCK_BATCH, 16)`。

**★★ 标签公式 `Ref($open, -N)/Ref($open, 0)-1` 存在前瞻偏差(A8)**。修复:`Ref($open, -(N+1))/Ref($open, -1)-1`。

**★★ DEFAULT_PERIOD 必须对齐官方的短期标签(A8)**。改 ret_2d 后年化 +34.8%,Sharpe 1.30。

**★ 单窗口诊断脚本大幅加速实验迭代(A8 / A11 / A12 / ★ A13 重写)**。

**★ DL_TAG 是冗余变量(A8)**。

**★ Alpha360 缓存目录重组(A8)**。

**★ Checkpoint 按 period 子目录隔离(A8)**。

**★ volume 归一化表达式避免 Ref(ATTR,0)(A8)**。

**★★ FeaturePreprocessor 的 inf 是 NaN 的根因(A9)**。

**★ Windows Memory Compression 导致 DataLoader 变慢(A9,★ A13 再次确认)**。A13: FeaturePreprocessor.transform 的 out-of-place 副本是另一个常见触发源。

**★ ContrastiveHead 的 BatchNorm 在 AMP 下产生 NaN(A9)**:改 LayerNorm。

**★ Conv1d(6→6) 在 encoder 前端不值得加(A9)**。

**★ SMA deepcopy 开销在大模型下不可忽视(A9)**:改 EMA。

**★ 验证每 VAL_EVERY=2 个 epoch 跑一次省 ~30% 时间(A9)**。

**★ cudnn.benchmark=True 对固定输入尺寸有效加速(A9)**。A11: 诊断脚本关掉(防 GRU+AMP CUDNN_STATUS_INTERNAL_ERROR)。

**★ simclr_loss 和 cross_space_align_loss 必须强制 FP32(A9)**。

**★ Pretrain WIP / 动态 worker 数 + PT_MAX_STEPS(A9)**。

**★ Pipeline WIP epoch 级断点续跑(A10)**。

**★ Pipeline 动态 worker 数防 Memory Compression(A10)**。

**★★ SMA 的 copy.deepcopy(model.state_dict()) 在 GPU 上累积内存碎片(A10)**:改 cpu().clone()。

**★★ ~~预训练提升检索质量但可能削弱 Gate 路由多样性(A10)~~ A11 修正:这个判断是错的**。aux loss 在所有配置下都会持续抹平初始分化,和检索空间无关。

**★★ 纯量价 Alpha360 信号在 2023-2024 系统性失效(A10)**。A12 对策:13 维扩展加入基本面/两融,配合 MASTER gate。

**★★ Expert 分化是多层次概念(A11)**:6 项指标组合看。

**★★ aux loss 是 expert 坍缩的元凶,即使在官方默认值 0.01 下(A11)**。

**★★ MERA 论文没有验证 expert 分化(A11)**。

**★★ 早停 min_delta 必须匹配 val_IC 实际噪声幅度(A11)**:0.001→0.005。

**★★ EXPERIMENT_TAG 不能和 DIAG_TAG 混用做实验分隔(A11)**。

**★ 共享 Expert 对照实验工具:SharedGRUMoE(A11)**。

### 11.2 A12 阶段教训

**★★ pickle 不能序列化函数内定义的 dataclass(A12)**:最初让 `qlib_converter.rebuild_market_features_v6` 在函数内 `@dataclass class MarketFeatureProvider`,pickle 时报 `AttributeError: Can't pickle local object`。修法:converter 只写 plain dict 带 `__format__` 标记;pipeline 加载时识别 dict 格式并 re-wrap 成自己内联的 dataclass 实例。避免了 ETL 和 pipeline 两边都需要相同的类定义。

**★★ ETL 自动 hook 必须加在 `jq_sync.run_qlib_conversion()` 而不是 `qlib_converter.main()`(A12)**:`qlib_trading.py` 调的是 `jq_sync.main()`,jq_sync 内部调 qlib_converter 的单个函数(staging / dump / verify),**不走 main()**。最初把 rebuild hook 加在 `qlib_converter.main()` 末尾,在 trading 流程里完全不会触发。正确位置:`jq_sync.run_qlib_conversion()` 末尾的 `qlib_rebuild_market_features()` 调用 + `qlib_converter.main()` 末尾的调用,两处都加。

**★★ MarketFeatureProvider 应内联 pipeline 而不是独立文件(A12 架构决策)**:最终方案:
- ETL(`qlib_converter.py`)独占 build 职责
- Pipeline(`qlib_pipeline_mera_v6.py`)独占 load + query 职责:`MarketFeatureProvider` 内联 80 行
- `market_features.py` 删除,避免冗余和漂移风险

**★★ 输出目录命名应和业务模块对齐而不是"qlib 前缀"(A12 架构决策)**:v6 改为 `Output/Mera_Output/` 对齐 Ensemble 系列命名风格。


<!-- ░░░ A16 原文行 1812-1888 在 v-A17 重建过程中暂缺 — 中后段(疑似 §10/§11 工程细则 / 阈值优先级表 §七 部分) ░░░ -->

*<small>v-A17 重建说明:此处 A16 原文 1812-1888 行内容因长上下文压缩缺失,如需追溯请参考 `A16_Fusion_Triple_Stack_v2.md` 原始备份。涉及主题:中后段(疑似 §10/§11 工程细则 / 阈值优先级表 §七 部分)。</small>*


83-95. ETL 扩展两融字段 / Qlib .bin 新增 7 字段 / industry_map.pkl 生成 / MARKET_DIM=30 设计 / qlib_pipeline_mera_v6.py / mera_single_window_diag_v6.py / rebuild_market_features_v6 内联 qlib_converter / jq_sync.run_qlib_conversion hook / pickle plain dict 格式 / market_features.py 删除 / 输出目录 Qlib_DL → Mera_Output / 端到端 pickle round-trip 测试 / v5/v6 完全隔离 ✅

### ★ 已完成(A13)

96-112. v6 首轮诊断 D_m13 完成(发现 17x 差距)/ align_check.py / verify_split.py / split_train_val 修复(cross-sectional → time-series)/ verify_feature_scale.py / Feature scale 三项修复(volume log1p / roe signed_log / turnover ×100 去除)/ MASTER α 衰减诊断 / expanding → rolling 修复 / MarketFeatureProvider rolling 模式开关 / FeaturePreprocessor.transform in-place 省 5GB / 检索 bank FP16(3:33→1:30)/ 诊断 del DataFrames 释放 16GB / 训练完 del train_flat_raw / TF32 启用 / ram_probe.py / `stage_b_clip_ratios` 入 JSON / `val_oos_ratio` 入 JSON ✅

### 进行中(★ A13 修复后)

113. v6 A13 单窗口诊断 D_m13 重跑 — A14 完成,**结果部分达标**:val IC=0.065 ✅ / daily IC=0.0226(接近但未达 0.04 起步)/ MASTER α OOS σ=0.008(未达 0.03,仍衰减 3.5×)/ clip 率 ≤ 5% ✅ / 分年 2024 IC 突破 0.033 是 A12+A13 最大胜利
114. ~~对照:v6 A13 关 rolling z-score~~ — A14 搁置

### ★ 已完成(A14)

131-142. A13 单窗口基线确认 / 单窗口诊断扩展内置回测 / HB Sweep v2 / HB Sweep 信号层天花板证实 / HB Sweep v1 废弃 / 九篇顶刊论文原文精读 / §七 大幅重写 / RSAP-DFM 机制纠错(DANN → FGSM + bilevel)/ Path 2 RSAP-DFM 借鉴版实现:
- stock-hidden regime extractor φ_DM
- FiLM 调制 gate_in:γ=1, β=0 初始化
- FGSM 对抗扰动 ε=0.01
- Joint loss `ℓ = ℓ_main + 0.3·ℓ_adv`
- ADV_WARMUP_EPOCHS=5

diag 脚本加 Path 2 参数同步 / diag 输出目录 tag 自动加 `_rsadv` 后缀 / A14 工程方法论形成(三层验证 / Claude 自查可信度分级 / 论文读过等级标注)✅

### 进行中(★ A14)

143. ~~Path 2 单窗口诊断 D_m13_rsadv 运行中~~ → **★ A15 已完成**(W1 ① val 0.069 daily 0.023 Sharpe 1.034)
144. ~~MERA 自监督预训练 v6~~ → **A15+ 用户决策推迟**(先做融合版三件套验证 Path 3 + Path 4 联动)

### ★ 已完成(A15)

163. A14+ Path 2 W1 第二次复现(D_m13_rsadv 同 config,gz2000 + SZ399303)✅
- val IC 0.0811(+18%),Sharpe 0.910(**跌破阈值**),Max DD -33.8%(创最差)
- **2024 daily IC 进一步退化到 0.0131**(vs ① 0.0185)
- 死 expert 现象消失,但 val/OOS gap 扩大
- **两次复现一致结论**:Path 2 在 2024 信用收缩+轮动是负贡献,催生融合版三件套

### ★ 已完成(A16)

164. 融合版三件套 D_m14_p2_p3_p4 W1 实测(csi1000 + SH000852)✅
- val IC 0.1087 / OOS daily IC 0.0358(**+79%**)/ daily IR 6.94 / Sharpe 1.084
- **2024 daily IC 救回 0.0284(+118%)**,所有分年改善
- Macro gate 真分化 0.68/0.20
- Max DD -36.2% 创项目最差,2023 Sharpe 0.49→-0.05 退步
- 累计超额 +152% vs CSI1000

165. 主 pipeline BENCHMARK 切到 SH000852 ✅
166. native::run_qlib_backtest() 加 benchmark 参数(不破坏自身行为)✅
167. diag 全面切换 csi1000 + SH000852 + 配对 assert ✅
168. Tag 命名规范重写为 Path X ✅
169. 参数区分层整合(9 段)✅
170. Diag 加 resume 逻辑(与 pipeline L2941 同款)✅
171. 诊断函数 `_get_expert(moe, e_idx)` helper 兼容 HierarchicalMoE ✅
172. Sharpe 计算口径不一致问题定位(暂未修复)🟡
173. beta/alpha/IR 分析(融合版 corr=0.976 / beta=0.94 / 残差 IR=1.93)✅

### ★ 已完成(A17)

180. **`qlib_pipeline_mera_v6.py` 真重复抽公因子 + 长函数瘦身** ✅:`train_mera_window_v5` 560→456 行 / `main()` 160→31 行 / `build_alpha360_expressions` 140→69 行 / `HierarchicalMoE` 164→138 行 / `SparseSequenceMoE` 104→83 行
181. **新增 7 个顶层 helper** ✅:`_build_gate_in_modulated` / `_moe_dispatch_topk` / `_load_pretrained_encoder_into` / `_resume_wip_or_init` / `_save_window_checkpoint` / `_print_config_summary` / `_print_backtest_stats` / `_plot_five_panels`
182. **零风险冗余清理** ✅:重复 `import torch` 移除;`from qlib.data import D` 三处内层 import 上提模块顶部
183. **`build_alpha360_expressions` 等价性验证** ✅:`/home/claude/test_equiv.py` 跑 780 表达式 + 780 名字字符串级逐字节比对,与原版完全一致
184. **47 个诊断脚本 public 名字保留** ✅:`grep` 验证全部 setattr 同步点不受重构影响
185. **A17 工程方法论 §15.17-15.23 七条** ✅

### 进行中(★ A17)

186. **完整训练-推理路径数值等价验证** ★:实跑一次单 window(D_m14_p2_p3_p4 W1)对照 A16 baseline(val IC 0.1087 / OOS daily IC 0.0358 / Sharpe 1.084),确认重构未引入数值差异。**A17 PENDING 验收项** — A18 user 日志显示 W1 已 resume 老训练 ckpt(val_loss=0.60260 / valIC=0.1087 与 A16 数值一致),侧面验证 A17 重构未引入 train-time 数值偏移
187. **`_print_backtest_stats` 改 IR + beta 口径**(A16 §15.13 已计划,A17 helper 注释里已标 TODO,实际未改) — **★ A18 仍未做,继承到 A18 待办**

### ★ 已完成(A18)

190. **`mera_pretrain_v6.py` 实质实现**(13 维输入版,A11/A16/A17 三轮列首要任务终于落地)✅:1071 行,SimCLR 双视图增强 + cross-space 360↔[60,13] align 双任务联合训练 + 行业中性化 + WIP 断点续跑 + 4 重 ckpt 校验。与 v5 `pretrain_mera.py` 完全独立并存
191. **Pipeline `_load_pretrained_encoder_into` 真实加载** ✅:由 A17 "警告 stub" 升级为实质实现,4 重校验:存在性 / `version==6` / `input_size==13` 与 `retrieval_flat_dim==360` / 关键 key 真的不在 missing 列表(防 strict=False 静默全 missing 隐藏 bug)
192. **`USE_PRETRAINED_ENCODER` 默认 True** ✅:不再被 A12-A17 的"v6 第一轮强制 False"覆盖
193. **`PRETRAIN_DIR_V6` 新常量** ✅:`OUTPUT_DIR / "mera_v6_pretrain"`,与 pretrain 脚本共享同一物理路径
194. **文件命名规范统一** ✅:`pretrain_mera_v6.py` → `mera_pretrain_v6.py`,与 `qlib_pipeline_mera_*` / `mera_single_window_diag_*` / `mera_hb_grid_search.py` 系列对应,与融合版 `ensemble_*.py` 系列对应。pipeline 5 处错误信息引用同步
195. **OUTPUT_DIR 错配工程教训沉淀** ✅:第一版 pretrain 误用 `Output/Qlib_DL`(v5 老路径)→ user 提示 → 改为 `Output/Mera_Output`,加注释 `# ★ 必须与 pipeline OUTPUT_DIR 一致`
196. **`SimilarProjector` 维度修正** ✅:第一版误用 `flat_dim=FLAT_DIM=780` → user 截图警告 v6 [M3] 检索 bank 是 360 维量价的设计选择 → 改为 `RETRIEVAL_FLAT_DIM=360`,`PretrainDataset.flat_x_price` 切出量价 6 字段
197. **A18 工程方法论 §15.24-15.26 三条** ✅
198. **`alpha_cache_v6` 路径归一** ✅:删除 `Output/Qlib_DL/alpha_cache_v6/` 重复副本(36GB),pretrain 与 pipeline 共享 `Output/Mera_Output/alpha_cache_v6/alpha_features_p2_d13.pkl`

### ★ 待办(A18,从 A17 继承 + 本阶段新增)

200. **`_print_backtest_stats` 改 IR + beta 口径**(A16 §15.13 / A17 § 待办 #187 继承)— 仍未做
201. **完整 expanding window 系列实跑验证 pretrain 效果** ★★★:把 `mera_v6_checkpoints/D_m14*/ ` 全部清空,重训 7-8 windows + 完整回测,对照 A16 融合版 baseline(daily IC 0.0358 / Sharpe 1.084),验证 A11 在 v5 上观察到的 "0.047 → 0.091" 翻倍效果是否在 v6 13 维上重现
202. **2024 daily IC 受预训练影响如何** ★★:A16 融合版救回 2024 daily IC 到 0.0284,需要看 pretrain 是否进一步改善 2024(融合版 + pretrain 双叠加)
203. **`mera_single_window_diag_v6.py` 同步 pretrain 加载逻辑**:diag 也走 `_load_pretrained_encoder_into` 才能在快迭代时复现 pipeline 行为
204. **pretrain 单 window 实测 loss 曲线收集**:确认 SimCLR + align 收敛形态(simclr 应从 baseline log(2B-1)≈9.70 收敛到 3.x,align 从 log(B)≈9.0 收敛到 4-6)
205. **是否考虑把 pretrain 也加进 Ensemble 信号源**:中期议题,看 A18 实测数据决定

### 进行中(★ A16,A17 仍承接)

174. ~~**MERA 自监督预训练 v6([60, 13] 版)** ★★★ :写 `pretrain_mera_v6.py`,A11 v5 [60,6] 实测 OOS IC 从 0.047 → 0.091 翻倍。**当前融合版 OOS daily IC 0.0358,预训练后预期 0.06-0.08,Sharpe 1.08 → 1.4+**。这是路线图上**上限最高且可信度最强**的一项,且和路径 1/2/3/4 全部正交可叠加。**A16 用户决策为下一步首要任务**~~ **★ A18 已实现**(脚本名按命名规范定为 `mera_pretrain_v6.py`),实跑数据待验证

### 近期(★ A16,按 daily IC 边际改善优先)

175. **Sharpe 计算口径统一**(diag L680-684 改算术口径)+ 加 `beta_to_benchmark` / `info_ratio` / `alpha_annual_pct` / `residual_std_annual_pct` 四个字段
176. **§7.7 阈值表补充 IR 判据**:从 "Sharpe ≥ 1.0" 扩展为 "Sharpe ≥ 1.0 或 IR ≥ 1.5"
177. **融合版 W2-W7 ensemble**:W1 跑通后扩展到 7 窗口,预期 Sharpe 1.08 → 1.2+。约 8h × 7 = 56h
178. **2024 / 2023 失败年份归因**:融合版 2024 救回但 2023 退步,可选 ablation:D_m14_p2_p3 / D_m14_p2_p4
179. ~~UMI RankIC Loss 改写~~ → **A16 已并入融合版,daily IC +73% 证明 work**
146. **Path 1 MASTER inter-stock attention**(下调):MERA pretrain 之后再考虑
147. **Path 4 UMI Stock-level cointegration + Market-level 自监督**:作为 MERA pretrain 的扩展 pretext task
148. **v6 消融 D_m14_p2reg**:关闭 FGSM 仅留 regime extractor
149. **v6 消融 D_m14_base + 其他子组件**
150. **v6 Shared Expert + 融合版叠加**
151. **qlib_trading_mera_v6.py**(融合版稳定后):支持 13 维 + 市场特征加载

### 中期

152. **Path 3 hierarchical MoE**(已完成在融合版里,此项历史)
153. **MERA v6 融入主策略 Ensemble 框架**:把 v6 pred_score 作为第三个信号源注入 XGB/LGB
154. **Expert 分化的语义验证**:gate top-1 和宏观 regime 标签做 correlation
155. **MASTER α 分化的语义验证**
156. **风格暴露控制**:扩展行业中性化到其他维度
157. **★ A13: mtss NaN 歧义修复**:43% 非两融股的 mtss 维度被 median 填充,添加 `is_mtss_eligible` 二元标志字段
158. **MASTER α OOS σ 衰减 3.5× 深度诊断**(A14):window 改 126/504 对比

### 长期

159. **分钟数据整合**
160. **多周期标签**(ret_2d / ret_5d / ret_20d 多标签预测头)
161. **再扩展输入维度**:北向资金流、行业动量、量价高阶统计量
162. **HireVAE / FactorVAE 深度借鉴**(仅在前面所有路径都做完后)

---

## 十二.5、★ v7+ 战略架构演进路线(A14 基于真实论文理解重排)

**★ A14 关键变化**:Claude 在 A14 阶段实际读过了 §七 列的九篇论文原文(除 DHMoE 外),对每条路径的机制和上限估计做了重新评估。**A13 原路径排序已不再适用**,新排序见 §7.7。

**★ A14 决策要点**:
- MERA 自监督预训练**升级为"下一步必做"**(A11 对照实验数据 0.047→0.091)
- Path 2 RSAP-DFM 借鉴版正在跑,结果待评估
- Path 1 MASTER inter-stock 优先级**下调**
- UMI 拆分成 stock-level cointegration / market-level 自监督 / RankIC loss 三部分,RankIC loss 是零架构改动可并行启动

**A14 结果**:A13 修复后 D_m13 daily IC=0.0226 / Sharpe=0.898 / 2024 分年 IC=0.033 突破,**值得继续投入**,但上限可能在 Sharpe 1.3-1.5 量级。

### v7+ 四条值得融的顶刊路径

#### 路径 1:MASTER (AAAI 2024) 完整版 — 补截面维

**当前状态**:v6 只实现了 market-guided gating(`α = 2·softmax(W·m)`),论文贡献的**一小部分**。

**未实现**:
- intra-stock attention(已有 Transformer backbone 近似)
- **★ inter-stock attention / cross-stock attention**:同一天所有股票的 last_hidden 做 self-attention(论文核心贡献之一)
- 市场引导控制时序/截面融合权重

**风险**:截面 attention 的 batch 组织必须按"交易日"。当前 DailyBatchSampler 每步 2 天约 4000 样本,但 cross-stock 需要一次看完一天的所有股票(~2000 只)。显存和 batch 组织都要重构。

#### 路径 2:RSAP-DFM (IJCAI 2024) — 已部署 ✅

详见 §3.1 / §7.1c.4。Path 2 W1 已实测 Sharpe 1.034 过阈值但 2024 衰退,W2 第二次复现 Sharpe 0.910 跌破阈值。融合版三件套 D_m14_p2_p3_p4 W1 已 deploy。

#### 路径 3:DeepSeekMoE hierarchical (ACL 2024) — 已部署 ✅

详见 §3.2 / §7.1c.8。HierarchicalMoE 已实现,2 macro × 4 micro。融合版 W1 实测 macro_gate 真分化 0.68/0.20。

**★ 严重陷阱 — 双层 MoE 的 gate aux loss 会叠加坍缩**:
A11 已发现单层 aux=0.01 都会坍缩。两层串联,第一层坍缩 → z 聚类失效 → 第二层输入退化 → 第二层也坍缩。**必须重新设计 aux loss**:
- 只惩罚 dead expert(usage<阈值,比如 1/NUM_EXPERTS × 0.1)
- **不惩罚 usage 不均**(A11 的核心教训)

#### 路径 4:KDD 2025 Irrationality + 基本面 — 补输入维度

**v6 已覆盖**:
- ✅ PE / PB / ROE(加了行业中性化 + ★ A13 signed_log 压缩)
- ✅ 市值(log_mv)
- ✅ 换手率(turnover_log,★ A13 修复公式)
- ✅ 两融(mtss_stock_rel / mtss_flow_rel)

**v7 仍需扩展**:
- 行业 one-hot
- 北向资金流(陆股通)
- 情绪因子(新闻情绪、研报情绪)
- 资金流异动(龙虎榜、大单净额)
- 换手异动(当日换手 / 60 日均换手的 z-score)

**预期**:这一步做完 OOS IC 大概率直接涨,**所有路径里 ROI 最高、风险最低**。

### 实施优先级(★ 严格按顺序,不可跳过)

| 阶段 | 工作内容 | 预估时间 | 风险 | ROI |
|---|---|---|---|---|
| 第 0 步(★ A13 完成) | v6 数据管线 bug 修复 | 已做 | 零 | **前提条件** |
| 第 1 步 | 路径 4:加行业 one-hot + 北向资金 + 换手异动 | 2 周 | 极低 | **最高** |
| 第 2 步 | 路径 1:MASTER cross-stock attention | 2-3 周 | 中 | 中高 |
| 第 3 步 | 多标签预测头(ret_2d / ret_5d / ret_20d) | 1 周 | 几乎零 | 中(对回撤直接有效) |
| 第 4 步 | 路径 2 + 路径 3:RSAP-DFM regime encoder + hierarchical MoE | 1+ 月 | **高** | 高但不确定 |

**触发条件**:每步做完跑一次全 pipeline 诊断。如果 IC 不涨或下降,**立即止损回退**。

### 三个绕不开的陷阱

#### 陷阱 1:Regime encoder 的 look-ahead

A 股历史数据上 regime label 很容易用 look-ahead 凑出来。RSAP-DFM 的 adversarial 设计是为了防这个,但**实现时要严格保证 z 的计算只用 t 之前的市场序列**。

**检查清单**:
- [ ] Market encoder 的输入只包含 [t-T, t-1] 的 m_t
- [ ] Regime label 只在 training 阶段用
- [ ] Expanding → rolling z-score(v6 A13 已实现)

#### 陷阱 2:双层 MoE 的 gate aux loss 叠加坍缩

```python
# ❌ 旧设计(会坍缩)
aux = cv_squared(importance) + cv_squared(load)

# ✅ 新设计(只防 dead expert)
threshold = 1 / NUM_EXPERTS * 0.1   # 使用率低于 10% 平均值判为 dead
dead_mask = (usage < threshold)
aux = dead_mask.float().sum() * penalty_weight
```

**核心原则**:只防止 expert 完全不被路由,**不强制 usage 均匀**。

#### 陷阱 3:Multi-task loss 权重不能固定

三个标签(ret_2d / ret_5d / ret_20d)的 loss scale 差很多。固定权重会让训练被 scale 大的任务主导。

**解决方案**:
- **GradNorm**(ICML 2018):根据各任务 gradient norm 动态调整权重
- **Uncertainty weighting**(Kendall CVPR 2018):每个任务学一个可训练的 log_variance

**简化方案**:在训练早期(前 5 epoch)记录每个任务 loss 的初始值,之后用 `loss_i / initial_loss_i` 归一。

### 战略定位

**重构 MERA 的价值定位**:做成一路**独立、高质量、低相关性**的信号源,融进现有的 Ensemble 框架,而不是指望它单独变成主力策略。**[★ A20]** 此定位现已有主线数据背书:mera 是 short-horizon 信号(period≤10 加分、长 period 减分),与长 period 主策略 horizon 互补、相关性低——正是 ensemble 该利用的结构,明确为"短 horizon 子策略"而非泛泛信号源。详 §十六。

**乐观估计**:全做完 OOS IC 从 0.09 拉到 0.13-0.15,Sharpe 从 1.09 到 1.5-1.8。但本项目已有的 XGB/LGB 主策略 2023-2024 超额 +12/+18%,另一条路线已经拿到了更好的结果。

**★ A13 补充**:在得出"架构欠拟合"结论前,必须先确认数据管线本身没 bug。A13 揭示的四个 bug(split / volume / roe / turnover) + MASTER z-score 模式,都不是"架构欠拟合",是**数据管线本身欠正确**。

**★ A14 补充**:A13 修复后 D_m13 首轮诊断 daily IC=0.023 / Sharpe=0.898 + HB Sweep v2 全档几乎没有差异,共同指出:
1. 数据管线本身目前是干净的
2. 当前信号质量正处于"架构欠拟合"的上限
3. 下一步应该是信号增强,不是组合构建优化

A14 结论:**架构欠拟合空间确实存在**(预训练在 v5 上 OOS IC 翻倍有对照数据,Path 2 和 inter-stock 虽然上限不确定但有机制支持)。~~**v7 重构值得投入**,但必须按 §7.7 优先级表逐项验证。~~ **[★ A20 重排]** v7 仍值得投入,但顺序上 **ensemble-first**:先用 v6 现状做零成本正交化诊断 + 短×长 horizon ensemble 验证(详 §十六),v7 信号增强降为"ensemble 验证后条件触发"。理由:主线数据显示 v6 短 period 价值未榨干,且 ensemble 不要求子策略单独强、只要求与主策略低相关。

---

## 十三、FAQ

### 数据与环境

- **gymnasium 警告** → `sys.modules["gym"] = gymnasium` 垫片
- **财报数据怎么更新** → `jq_sync.py` daily 模式自动采集
- **qlib_converter.py 怎么跑** → 不用单独跑,jq_sync daily/full 模式末尾自动调用(末尾还会自动调 rebuild_market_features_v6)
- **Qlib .bin 存在哪** → `PROJECT_ROOT/qlib_data/cn_jq/`
- **CSV 编码错误** → 默认 `gbk`,依次 `gb18030` → `utf-8-sig`
- **stock_names.pkl 过时** → 30 天自动刷新
- **vwap.day.bin 不存在** → 删除 `qlib_data/cn_jq/features/` 整个目录重跑
- **mtss_fin_value.day.bin 不存在** → 同上,删 features 重跑 ETL
- **industry_map.pkl 不存在** → 手动跑 `generate_industry_map.py`
- **Ref(ATTR, 0) WARNING 刷屏** → 修改 `site-packages/qlib/data/ops.py`
- **★ A13: `$turnover_rate` 单位** → 百分比(3.665 表示 3.665%)。Python 层不要再乘 100
- **★ A13: JQ CSV 文件在哪** → `Dataset2006JQ/MarketData/`,文件名格式 `{code6}.{SZ|SH}.CSV`(大写扩展)
- **★ A13: 成分股 CSV 的 code 是数字还是字符串** → 整数。例如 `2222` 表示 `002222`。读取时需 `str(int(code)).zfill(6)`

### MERA 模型(v5 / v6)

- **v5 和 v6 能同时跑吗** → 能。完全隔离
- **v6 和 v5 怎么选** → v6 是升级版(A12 架构 + A13 bug 修复),v5 是稳定基线
- **v6 参数和 v5 一致吗** → 大部分一致。新增(A12):INPUT_SIZE 6→13, MARKET_DIM=30, USE_MASTER_GATE, USE_INDUSTRY_NEUTRALIZE。变更:RETRIEVAL_INPUT_SIZE 保持 6, USE_PRETRAINED_ENCODER=False, RETRIEVAL_SPACE="raw"。★ A13:MARKET_ZSCORE_MODE="rolling", MARKET_ROLLING_WINDOW=252
- **Gate 用什么信息** → 仅用检索邻居的离散标签 bucket embedding
- **MASTER gate 是什么** → `α = 2·softmax(W·m)`,m 是 30 维市场向量
- **MASTER gate 怎么初始化** → `W` 和 `bias` 都 zeros init
- **★ A13: MASTER gate 在 OOS 上为什么瘫痪** → 市场向量 m 用 expanding z-score 归一化,std 随时间衰减
- **★ A13: 修复方法** → `MarketFeatureProvider.load_or_build()` 加载 pkl 后用 rolling 252 日 z-score 重算 features
- **gate_aux_weight 为什么是 0.01** → 官方代码硬编码。A11: 仍是坍缩元凶
- **MoE 有残差连接吗** → 没有
- **SMA 怎么工作** → EMA 就地更新 decay=0.2
- **checkpoint 存在哪** → v5: `Output/Qlib_DL/mera_v5_checkpoints/`;v6: `Output/Mera_Output/mera_v6_checkpoints/`
- **★ A13: 从 A12 升级到 A13 要删 checkpoint 吗** → 必须删。A12 的 best_val_ic 是按股票切的虚假值
- **可以在 CPU 上跑吗** → 检索模块依赖 GPU
- **跑一次要多久** → RTX 4090,~10-14 小时 / 7 窗口。**★ A13 预期 ~7-10 小时**
- **内存不够怎么办** → ★ A13:修复前 ~100 GB,修复后 ~70 GB。如果还吃满,缩 DataLoader num_workers
- **UNIVERSE_CSV 怎么切换** → 文件顶部改
- **v6 第一次跑报 FileNotFoundError: market_features_v6.pkl** → 先跑 `python etl/qlib_converter.py`
- **v6 的 market_features 多久更新一次** → 每次 ETL 自动更新

### Expert 分化(A11)

- **expert 分化怎么判断,只看 gate_usage 够吗** → 不够。6 项指标组合看
- **MERA 论文报告 gate 分化数据了吗** → 没有
- **D 组(encoder 检索)和 A 组(raw 检索)分化差异大吗** → A11: D 组 gate_variance_mean=0.011 略高于 A 组 0.009
- **为什么 gate 初期有分化后来变均匀** → aux loss 持续施压
- **如何打破这个坍缩** → GATE_AUX_WEIGHT 0.01→0.001 / TOP_K_EXPERTS 4→2 / 丰富 gate input / GATE_AUX_WEIGHT=0
- **D_shared 到底有没有用** → A11: 改变 MoE 内部分工但 OOS IC 不提升
- **共享 expert 对照实验怎么跑** → `SHARE_ALL_EXPERTS=True`

### MASTER α 诊断(A12 + ★ A13)

- **alpha_sum_of_mean 应该是多少** → 理论 2.0,实测应在 1.8~2.2 内
- **alpha_sum_of_mean 偏离 2.0 意味着什么** → Gate 初始化或公式 bug
- **alpha_std_overall 应该是多少** → 0 = gate 没学东西,>0.05 = 正常,>0.15 = 强分化
- **alpha_std_overall=0 是什么意思** → gate 对所有市场状态都输出同样的 α
- **alpha_mean_per_field 某维度 > 1.0 意味着什么** → 那个维度被 gate 平均放大了
- **MASTER α 分化 = regime 识别吗** → 不等于
- **★ A13: alpha_std_train vs alpha_std_oos 差距多大正常** → rolling z-score 下应接近(1.0-1.5× 以内)。如果 OOS σ 比训练期 σ 小 3 倍以上,查 ETL 输出

### 市场特征(A12 + ★ A13 rolling z-score 修正)

- **market_features_v6.pkl 在哪** → `Output/Mera_Output/market_features_v6.pkl`
- **谁负责生成这个 pkl** → `qlib_converter.rebuild_market_features_v6`
- **pipeline 会不会自动生成** → 不会
- **pkl 多大** → 几百 KB
- **pkl 多久过期** → 缓存末日期 >= 请求日期就用
- **为什么 30 维不是 63 维** → v6 Universe 是国证 2000 小盘,板块指数无用
- **为什么 pipeline 要内联 MarketFeatureProvider 而不是从 market_features.py import** → 避免 ETL/pipeline 两处真相源
- **第一次跑 qlib_converter 输出 3557 dates 但合并后只有 2873 dates 正常吗** → 正常。三个指数的起始日期不同,inner join 后取交集
- **★ A13: rolling 和 expanding 的区别** → expanding 每步用 `[0..t]` 累积 mean/std,OOS 期 z-score 幅度衰减;rolling 用 `[t-252..t]` 滚动窗口,历史长度恒定
- **★ A13: 改 rolling 要重建 pkl 吗** → 不用。pipeline 加载 pkl 后在内存里对 raw_features 重算 z-score
- **★ A13: 想用 expanding 对照怎么办** → 参数区改 `MARKET_ZSCORE_MODE = "expanding"`
- **★ A13: 为什么 window=252** → 1 年交易日,匹配 A 股年度 regime 变化周期

### 单窗口诊断

- **v1/v2/v6 诊断的区别** → v1 (A8, 已废) → v2 (A11) → v6 (A12,★ A13: 增加 TF32 + del DataFrames + stage_b_clip_ratios / val_oos_ratio)
- **什么时候该用诊断脚本** → 验证架构/参数改动对 OOS IC + expert 分化 + MASTER α 的影响
- **诊断脚本要改 pipeline 代码吗** → 不用。通过 `setattr(_pipe, ...)` 同步
- **EXPERIMENT_TAG 冲突怎么办** → 用独立 TAG,不能只靠 DIAG_TAG
- **为什么 cudnn.benchmark 诊断脚本要关** → 防 GRU+AMP CUDNN_STATUS_INTERNAL_ERROR
- **早停为什么不触发** → A11: `EARLY_STOP_MIN_DELTA=0.001` 太小,提到 0.005
- **v6 诊断输出在哪** → `Output/Mera_Output/diag_v6_W{WINDOW_ID}_{TAG}/`
- **★ A13: 诊断跑一次要多久** → A12 ~40-60 分钟。A13 修复版 ~20-30 分钟
- **★ A13: 诊断 RAM 峰值多少** → A12 ~100 GB,A13 ~70 GB

### ★ A13 新增:数据对齐与 bug 定位工具

- **align_check.py 做什么** → 三源对齐验证。选一只股票在三段时间窗口,手算 label 和所有 feature
- **align_check.py 输入什么** → 无参数。顶部硬编码 `WINDOWS`
- **align_check.py 输出什么** → 三段 CSV + 控制台打印每行的 open/close/turnover/mtss/label
- **verify_split.py 做什么** → 打印 train/val 切分后 tr_slice 和 va_slice 的日期范围和股票集合重叠率
- **verify_feature_scale.py 做什么** → 对所有输入字段打印 Stage 0/A/B 的 p01/p50/p99/NaN 占比 + clip 触发率
- **ram_probe.py 做什么** → 在 pipeline 每个关键阶段打印 RSS + 系统内存
- **什么时候应该用这些工具** → 任何 "IC 不对"、"RAM 爆满"、"val/OOS 差距大" 的时刻

### 预训练(v5 / ★ A18 v6)

**v5(`pretrain_mera.py`)**:
- **预训练脚本怎么跑** → PyCharm 打开 `pretrain_mera.py` Run
- **预训练跑几次** → 只跑一次
- **预训练 checkpoint 存在哪** → `Output/Qlib_DL/mera_v5_pretrain/`
- **怎么启用预训练初始化** → `qlib_pipeline_mera_v5.py` 参数区 `USE_PRETRAINED_ENCODER = True`
- **预训练 loss 正常范围** → SimCLR 100 epoch 后 simclr ~3.3-3.8,align ~3.7-3.9
- ~~**v6 也有预训练吗** → 首轮没有。v5 pretrain 是 [60, 6],v6 [60, 13] 不兼容~~ **★ A18 已实现**:见下

**★ A18 v6(`mera_pretrain_v6.py`)**:
- **预训练脚本怎么跑** → PyCharm 打开 `mera_pretrain_v6.py` Run
- **预训练跑几次** → 只跑一次,除非 expanding window 配置改了或想用新数据重训。已存在的 window ckpt 自动跳过
- **预训练 checkpoint 存在哪** → `Output/Mera_Output/mera_v6_pretrain/pretrain_encoder_v6_window_NN.pt`(每个 window 一个)
- **怎么启用预训练初始化** → `qlib_pipeline_mera_v6.py` 参数区 `USE_PRETRAINED_ENCODER = True`(A18 默认 True 不再被强制 False 覆盖)
- **预训练 loss 正常范围** → SimCLR baseline log(2B-1)=log(16383)≈9.70,收敛到 3-4;Align baseline log(B)=log(8192)≈9.0,收敛到 4-6;PT_ALIGN_WEIGHT=0.5 总 loss 范围约 5-7
- **预训练单 window 时长** → RTX 4090 + PT_BATCH_SIZE=8192 + PT_MAX_STEPS=200 + PT_EPOCHS=100 → 约 30-60 分钟 / window
- **8 windows 全跑总时长** → 约 4-8 小时,可中途停掉,WIP 文件 `*_wip.pt` 每 10 epoch 保存一次
- **加载日志怎么看** → `[Pretrain] ✓ Loaded v6 ckpt: pretrain_encoder_v6_window_02.pt | best_loss=4.2317 | loaded=XX/YY keys | missing=N unexpected=0`。**`unexpected=0` 必须**(>0 = pretrain 端 key 命名 bug);`missing>0` 正常(market_gate / moe / pred_head / regime_extractor 等下游模块本来就不在 ckpt 里)
- **如果加载报关键 key 在 missing 里** → A18 加载逻辑会 raise,提示去检查 `mera_pretrain_v6.py` 里 `combined_state` 拼装的 key 命名映射(`SimilarProjector.proj.0.weight → similar_proj.0.weight`)
- **什么时候 v5 / v6 ckpt 互不兼容** → 完全不兼容。v5 input_proj 是 `Linear(6, 128)`,v6 是 `Linear(13, 128)` shape 直接对不上;v5 ckpt 没有 `version` 字段,A18 加载会 raise `"v6 model 需要 v6 ckpt, 当前 ckpt version=-1"`
- **v6 pretrain 也用行业中性化吗** → 用,与 pipeline 完全一致。pretrain `FeaturePreprocessor` 两阶段(Stage A 含 PE/PB 行业中性化 + Stage B 全局 z-score)与 pipeline 同一份代码逻辑
- **temp 关闭 v6 pretrain 做 baseline 对照** → pipeline 参数区 `USE_PRETRAINED_ENCODER = False`

### 回测与交易

- **怎么跑回测** → v5: `qlib_pipeline_mera_v5.py`;v6: `qlib_pipeline_mera_v6.py`
- **怎么跑 MERA 交易** → v5: `qlib_trading_mera.py`;v6: 暂未实现
- **怎么跑主策略交易** → `qlib_trading.py`
- **qlib_trading.py 用改吗** → 不用。内部调 jq_sync.main(),自动包含 market_features 重建
- **基准指数怎么加** → `qlib_converter.py` 顶部 `BENCHMARK_INDEXES` 字典加一行
- **回测图在哪** → v5: `Output/Qlib_DL/MERA_V5_ret2d_*_{TAG}/backtest_nav.png`;v6: `Output/Mera_Output/MERA_V6_ret2d_*_{TAG}/backtest_nav.png`

### 回滚

- **v6 全线失败怎么回退 v5** → v5 文件完全不动。直接跑 `python qlib_pipeline_mera_v5.py`。v6 独立占用 `Output/Mera_Output/`、`mera_v6_checkpoints/`、`market_features_v6.pkl`
- **回退后 ETL 还会跑 rebuild_market_features_v6 吗** → 会,但只是多算 0.1 秒。v5 不碰这个文件
- **★ A13: 改动想回退怎么办** → 每项修正都在参数区有开关:(1) `MARKET_ZSCORE_MODE="expanding"` 回退到 A12;(2) `split_train_val` 改回 `slice(0, n_train), slice(n_train, n)`;(3) `_transform_stage_a` 的 volume/roe/turnover 修正注释掉;(4) `transform(inplace=False)` 回退;(5) 检索 bank dtype 改回 `torch.float32`

---

## 十四、★ A13 工程方法论

本节总结 A13 阶段形成的、适合迁移到其他 project 的方法论。不特定于 MERA 或量化策略。

### 14.1 核心原则:渐进式 >> 一次堆 5 个功能

**为什么渐进式看起来慢,实际快**:
- 每个改动独立验证,失败定位精准,不会 14 小时打水漂
- 单窗口诊断 40 分钟 vs 全量 7 窗口 10-14 小时,试错成本差 20 倍
- 归因清晰:IC 提升或下降都能说出"因为改了 X"
- 5 个改动组合 = 2^5 = 32 种可能,完整消融成本巨大

**为什么"一次堆 5 个功能"更慢**:
- 组合改动后 OOS IC 不管涨跌,都不知道**哪一个**是元凶
- 引入 NaN / OOM / 训练崩溃的风险指数级上升
- 14 小时跑失败后回退哪一个改动?全部回退重来
- 早期错误在后续 phase 继续叠加,越往后越难拆

**实操建议**:
1. **独立模块可以并行**(比如 ETL 重构和模型架构改动互不影响)
2. **影响同一最终指标的改动必须串行**
3. **改动之间有传递依赖时优先改上游**(A13 先改 split,再改 feature scale,再改 MASTER z-score)
4. **每个改动跑单窗口诊断 40 min 通过后再动下一个**

### 14.2 Bug 定位顺序:数据对齐 → Split → Feature → Gate → 架构

从最便宜、最容易验证的开始。跳过任何一步都容易误诊。

**Level 1: 数据对齐**(10 分钟)
- 验证 label 公式无前瞻
- 验证每个 Qlib 字段单位和假设一致
- 验证 index 对齐(MultiIndex datetime / instrument 顺序)

**Level 2: Train/Val/Test Split**(5 分钟)
- 打印 tr / va / te 的日期范围、股票集合、样本重叠率
- 时序数据必须按 datetime 切,不能按行序
- val 集必须在 train 之后(时间上)

**Level 3: Feature Scale**(30 秒)
- 每维度打印 Stage 0/A/B 的 p01/p50/p99/NaN/clip 率
- 重尾必须压缩(log / signed_log / winsorize),z-score 不会自动修复
- 每维度 Stage B 值域应在 ±2 附近,clip 率 < 5%

**Level 4: Gate / Attention 输入归一化**(1 小时)
- Gate 输入的跨时间 std 在训练期和 OOS 期是否一致
- Expanding z-score 在长回测期会衰减,考虑 rolling
- Attention weight / softmax 输出的熵在 OOS 上是否退化

**Level 5: 架构调整**(数周)
- 只有前 4 级都干净了,才考虑换 backbone、加分支、改 MoE 结构
- 如果架构改动下 IC 变化小,先怀疑前 4 级有漏网 bug,再怀疑架构本身

### 14.3 RAM 限制环境下的数据流设计

**核心原则:所有中间产物都要明确生命周期**

**禁止同时持有 raw + transformed 的同一份数据**:
- pandas DataFrame → numpy array 提取后立即 del DataFrame
- pre.transform(raw_array) 设计成 `inplace=True` 模式
- 下游如果还需要 raw 值,必须显式复制

**禁止 DataLoader num_workers 自动 × 数据副本**:
- BankCollator 思想:大对象从 Dataset 分离到 collate_fn
- Dataset 只存轻量 index,worker 复制成本可控
- num_workers 数量必须和 (RAM budget / Dataset size) 挂钩

**Windows Memory Compression 是信号**:
- RAM 显示"未用满"但触发压缩 = 实际已经饱和
- 每次访问压缩页 CPU 解压,DataLoader 速度断崖
- 监测:任务管理器 → 性能 → 已压缩的那行 > 5 GB 就警告

### 14.4 诊断工具的写作规范

**四个必备工具**:

<!-- ░░░ A16 原文行 2297-2319 在 v-A17 重建过程中暂缺 — FAQ 中段或 §15.x 早期工程方法论小节 ░░░ -->

*<small>v-A17 重建说明:此处 A16 原文 2297-2319 行内容因长上下文压缩缺失,如需追溯请参考 `A16_Fusion_Triple_Stack_v2.md` 原始备份。涉及主题:FAQ 中段或 §15.x 早期工程方法论小节。</small>*

- 顶部参数区硬编码(PyCharm 一键 Run)
- 输出分段打印 + CSV 落盘,便于粘贴给协作者
- 运行时间 < 5 分钟

### 14.5 A13 总结性教训

- **任何 split 必须手动验证 tr/va 的日期和样本集合**
- **新字段必须 scale 扫描后才能 z-score**
- **Qlib / 商业 API 字段单位没有文档,必须 p50 验证**
- **MASTER / Gate / Attention 的 softmax 输入必须检查跨时间稳定性**
- **Windows Memory Compression 不是"缓冲区",是性能悬崖**
- **pre.transform 默认 out-of-place 是常见 RAM 杀手**
- **bug 定位顺序:数据对齐 → Split → Feature → Gate → 架构**
- **渐进式验证是最便宜的消融;一次堆功能是最贵的赌博**
- **v6 首轮 IC 0.0218 看起来是"架构不 work",实际是 4 个 bug 叠加**

---

## 十五、★ A14 工程方法论

§十四 是 A13 形成的数据管线和 bug 定位方法论,§十五 聚焦"顶刊机制借鉴"和"Claude 自查"两件事。

### 15.1 顶刊机制借鉴的三层验证原则

**A14 的 RSAP-DFM 事故揭示的系统性风险**:Claude 在 A13 阶段根据 §七 "参考文献一句话概括"写了 Path 2 实现建议,说"用 gradient reversal layer 防止 regime encoder 过拟合训练期 regime label"。A14 用户质疑后 Claude 去读 RSAP-DFM 原文,发现真实机制是 **FGSM 梯度扰动 + bilevel 优化 + 双 regime shifting encoder**,和 gradient reversal 完全无关——后者是 DANN(Ganin 2016)的机制。Claude 把 DANN 的关键词糊到了 RSAP-DFM 上,产生了一段"听起来合理但完全不对"的实现建议。

**为什么会发生**:大模型从训练数据里见过很多"对抗训练"相关论文,在缺少原文输入时,倾向于把**最常见的对抗机制**(gradient reversal)拼到**任何提到"adversarial"的论文**上。

**三层验证原则**:

**Layer 1:读 abstract**(5 分钟)
- 只能知道论文"宣称做了什么",不能知道"具体怎么做"
- 产出:论文的一句话总结 + 和本项目的潜在借鉴点
- **不足以支持代码实现**

**Layer 2:读原文 + 代码仓库**(30-60 分钟)
- 具体数学公式 + 模块架构图 + 主干代码实现
- 产出:可迁移机制的精确描述("FiLM 调制不是 concat","γ=1+ε·z 不是 γ=σ(W·z)")
- **支持 Path 2 这种 "借鉴核心思想+简化实现" 类型的代码改动**

**Layer 3:兼容性评估**(另外 30 分钟)
- 原论文范式(DFM/VAE/MoE)和本项目范式是否兼容
- 不兼容 → 选择"简化借鉴"
- 简化程度和论文 ablation 的覆盖度成正比

**规则**:跳过 Layer 2 直接从 Layer 1 的 abstract 拼实现 = 你的"RSAP-DFM 借鉴"可能变成 DANN。Layer 2 是唯一能保证机制正确的环节。

### 15.2 Claude 自查可信度分级

| 等级 | 依据 | 适用场景 |
|---|---|---|
| **高** | 本项目实验数据支撑 | "MERA 预训练 v6 预期 daily IC 0.023→0.03-0.04"(依据:A11 v5 D 组实测 0.047→0.091) |
| **中** | 论文 ablation 明确报告 + Layer 2 读过机制 | "DeepSeekMoE shared expert 能缓解 expert 均匀化"(依据:A10 已实现,A11 实测) |
| **中低** | 论文总体结果但无 ablation + Layer 2 读过机制 | "MASTER inter-stock attention 预期 +0.005-0.015 IC"(论文超过 DTML 全指标 p<0.01,但 inter-stock 单独贡献未 ablation) |
| **低** | 只读过 abstract | "HireVAE 的 market encoder + stock encoder 双层 latent 在 regime shift 时可能更鲁棒" |
| **极低** | 只看过标题 | "DHMoE 可能对波动率高时的 expert 路由有帮助" —— **应拒绝给出** |

**规则**:
- Claude 向用户陈述论文理解时,必须标注等级
- 用户基于"低"等级的陈述做工程决策 = 高风险
- 极低等级的陈述应该改成"未读,无法评估"

### 15.3 论文读过等级标注规范

**适用对象**:任何涉及论文引用的知识库章节。

**标注粒度**:每篇论文必须标注 Claude 的实际读过级别:
- 读过原文:PDF / arXiv HTML / 代码 repo 至少读过一次
- 读过 abstract:只读了会议页面或 arXiv 的 abstract
- 只看过标题:不应基于此做任何判断

**标注位置**:论文 reference 表里加一列 "核实等级"。

**禁止行为**:
- ❌ 用"顶刊推荐"当挡箭牌引用没读过的论文
- ❌ 把多篇论文的关键词拼接出"综合建议"
- ❌ 向用户演绎"论文 X 可能的机制"时不标注可信度

### 15.4 诊断工具的完整性决定迭代速度

**A14 关键教训**:A13 的 diag 脚本只产出 IC/IR 等数值指标,Sharpe / 换手 / 回撤 / 分年等"真正决定策略质量"的指标要跑完 full pipeline 才能看到。A14 把 `run_qlib_backtest` 集成到 diag 脚本,单窗口 40min 就能得到完整的评估。**这让迭代速度提升至少 5 倍**。

**通用原则**:诊断工具必须覆盖 pipeline 的**全部评估步骤**:
- 回测(Sharpe / 回撤 / 换手 / 分年分解)
- 组合构建层的实际约束(HB / TOP_K / 成本)
- 持仓稳定性(日换手 / 持仓集中度)
- 模型内部诊断(expert 分化 / gate 熵 / attention σ)

**反面教训**:A14 的 HB Sweep v1 自写简化回测,close-to-close 替 open-to-open + turnover 算错。**pipeline 已经有的函数必须复用**,诊断脚本不允许"简化重写"关键指标。

### 15.5 组合构建层调参是"后段游戏"

**通用教训**:组合构建层的调参(HB / TOP_K / TURNOVER_THRESHOLD / 风险暴露约束等)是"signal 足够强之后"的精调。

**阈值经验值**(A 股日频截面选股):
- daily IC < 0.02 → 组合调参没用,必须先改 signal
- daily IC 0.02-0.04 → 组合调参可能有边际贡献(Sharpe ±0.1)
- daily IC > 0.04 → 组合调参开始有明显回报

**当前 v6 状态**:daily IC=0.023 处于"组合调参边际有用"区间下沿,继续往信号层投入的 ROI 比组合调参高得多。

### 15.6 A14 总结性教训

- **顶刊机制借鉴必须读原文,凭 abstract 拼实现必出 DANN/RSAP 混淆**
- **Claude 自查可信度分级是防止误导决策的硬约束**
- **诊断工具的完整性(内置回测)决定迭代速度**
- **pipeline 已有的函数必须复用,不是"可以重写"的选择题**
- **组合构建层调参(HB / TOP_K)是"信号足够强之后"的精调,弱信号下无效**
- **信号层 IC < 0.04 daily 时,应把工程时间投到 signal 层(预训练 / regime / inter-stock)**
- **Path 2 实现三原则:FiLM γ=1 β=0 冷启动等价 / warmup 让模型先稳定 / joint loss 代替 bilevel 调参空间压缩 10 倍**
- **任何论文路径上限估计必须标可信度,极低等级应拒绝给出**
- **RSAP-DFM 真实机制是 FGSM + bilevel 双 regime shifting 编码器,不是 DANN 梯度反转**

---

## 十五.5、★ A14+ 部署期工程方法论

### 15.7 AMP 梯度路径的 FP16 精度陷阱:FGSM 三次失败的教训

**现象**:Path 2 v1 在 `adv_warmup_epochs=5` 之后的第一个对抗 step 立即在 `_prob_in_top_k` 的 `normal.cdf` 报 `ValueError: The parameter loc has invalid values`。连续三次修复才根治。

**第一次失败(v1)— `scaler.scale` 错加**:把 `scaler.scale(loss_main)` 加到 `torch.autograd.grad(loss, gate_in)` 之前。实际上 `GradScaler` 只对**参数梯度**(FP32 master copy)缩放,不对中间节点梯度(FP16)生效。用 scaler.scale 放大 loss 再求 grad,拿到的 g 是 FP16 被放大 65536× 直接 overflow `+Inf`;除以 scaler 再除以 norm → NaN → 传进 `softmax` 再进 `normal.cdf` 炸。

**第二次失败(v2)— `nan_to_num` 救不回 Inf**:移除 `scaler.scale`,直接 `autograd.grad`,在 `g` 后加 `nan_to_num(g, nan=0, posinf=0, neginf=0)`。再次同位置炸。定位:真正坑在 `g.norm(dim=-1)` 本身。`norm = sqrt(sum(x²))`,FP16 下 `sum(x²)` 对**接近 FP16 max(65504) 的有限值**会 overflow 成 `+Inf`(不是 NaN,所以 `nan_to_num` 不拦截),后续 `g / norm` 产生 NaN。

**第三次(v3)— FP32 隔离根治**:整段 FGSM 数学全放在 `with torch.amp.autocast("cuda", enabled=False)` 里强制 FP32:

```python
g, = torch.autograd.grad(loss_main, gate_in, retain_graph=True)
with torch.amp.autocast("cuda", enabled=False):
    g_fp32 = torch.nan_to_num(g.detach().float(), nan=0, posinf=0, neginf=0)
    g_fp32 = torch.clamp(g_fp32, min=-1e4, max=1e4)  # 防御性上界
    g_norm = g_fp32.norm(dim=-1, keepdim=True).clamp(min=1e-8)  # 防零除
    gate_in_fp32 = gate_in.detach().float()
    gate_in_adv = gate_in_fp32 + ADV_EPSILON * (g_fp32 / g_norm)
    gate_in_adv = torch.nan_to_num(gate_in_adv, nan=0, posinf=0, neginf=0)
# gate_in_adv 保持 FP32 送进后续 MoE autocast 块,内部 matmul 自动降级 FP16
```

**通用教训**:
1. **AMP 的 `GradScaler` 只管参数梯度,不管中间节点梯度**
2. **看症状(NaN)找不到源头(Inf)**。必须顺梯度链反推到第一个精度不足的算子
3. **FP16 max ~65504,任何 `sum(x²)`, `log(x)`, `exp(x)` 这种非线性累加算子对接近上限的正常有限输入都可能悄悄 overflow**
4. **`nan_to_num` 只拦截 NaN/Inf 输出,拦不住"有限但即将参与 overflow 计算"的输入**。clamp 才是防御性上界
5. **在 autocast 块内手动 `enabled=False` 嵌套是合法且推荐的做法**

### 15.8 改 Dataset signature 的"间接消费者"全局追踪原则

**现象**:RankIC loss 需要 date_id 做截面分组,Dataset 从 4 元组改为 5 元组,改完三处直接 for-loop 解包(train/val/predict)。第一个 batch 立即崩:`ValueError: too many values to unpack (expected 4) in DataLoader worker process 0 → fetcher.fetch → self.collate_fn(data) → BankCollator.__call__`。

**根因**:BankCollator 的 `__call__(self, batch)` 内 `seq_list, nidx_list, y_list, m_list = zip(*batch)` 仍解 4 元素。num_workers>0 时 collate_fn 跑在子进程,`ValueError` 穿过 `_worker_loop` 的 traceback,初看像是 DataLoader 问题而不是 Dataset 问题。

**教训**:改 Dataset `__getitem__` 签名必须同步 grep 三类 site:
1. **直接 unpack site**:`for ... in dl` / `for ... in enumerate(dl_*)`
2. **间接 unpack site**:`collate_fn` 类的 `__call__`、任何 `zip(*batch)` 模式
3. **返回类型依赖**:`return seq_batch, ... , date_batch` 的 tuple 长度也要改

预防模板:

```bash
grep -n "zip(\*batch)\|collate_fn\|__call__.*batch\|for.*in dl\|for.*in enumerate.*dl" pipeline.py
```

### 15.9 "融合版 vs 单变量 ablation" 的决策框架

**A14+ 决策背景**:Path 2 W1 部署成功(Sharpe 过阈值)但 2024 衰退。两种下一步策略:
- **单变量 ablation**:下一次只开一个新组件
- **融合版**:Path 2 + Path 3 + RankIC 同时开,一次跑拿最强结果

实际决策选融合版。理由:
1. **目标不是论文归因,是系统性能最大化**。融合版若比 Path 2 好,直接上,不需要知道每个组件几分贡献
2. **期望值更高**:三个组件都是可靠论文支撑,独立 work 概率 70%+,叠加后变差概率只有 20-30%。**80% 场景下融合版是最优**
3. **失败成本低**:若融合版变差,再做两次单变量 ablation 即可定位

**通用原则**:
- 归因实验(学术):单变量 ablation 优先
- 性能实验(工程):融合版优先,失败才 ablation
- 两种组件都有论文支撑 + 不冲突 → 融合版;其中一个不确定 → 先 ablation

### 15.10 Path 3 Hierarchical MoE 的设计决策记录

**决策 1:2 macro × 4 micro 而不是 4×2 或 3×3**
- 2 macro 对应"宏观 regime 的极简分类";4×2 让 macro 过细 micro 过少失去 expert 容量
- 3×3=9 和 baseline NUM_EXPERTS=8 不符

**决策 2:L1 top-1 硬路由 + L2 top-2 软路由**
- L1 硬路由:每个样本只进一个 macro,macro_gate 学"regime 硬分类"
- L2 top-2 软:每个 macro 内 4 个 micro,top-2 给 soft mixture 保持足够容量

**决策 3:Micro experts 每 macro 独立不共享**
- 共享的话 macro 之间的 expert 退化成同样东西,L1 路由变成"纯 aux loss balance"无语义
- 独立的话每 macro 的 4 个 expert 专注自己 regime 下的 pattern

**决策 4:两级 aux loss 独立 cv_squared 而不是联合**
- 独立:macro_aux 平衡 2 个 macro,micro_aux[m] 各自平衡 macro m 内的 4 个 micro
- 联合:把 macro 和 micro 的 importance/load 拍扁成 8 维再算 cv_squared,失去层级感

**决策 5:L1 输入纯 z,不加 market vector m 或 gate_inp**
- 纯 z:L1 macro 只依赖"模型眼中的 regime",和 L2 的 gate_inp(stock 特征)正交
- z + market:冗余(MASTER gate 已经用过 market 一次)
- z + gate_inp:破坏了"L1 做 regime、L2 做 stock"的层级语义

**决策 6:FGSM 扰动对象仍是 gate_inp,不扰动 z**
- 语义:FGSM 训的是"micro-level 对 stock 噪声鲁棒",regime 信号本身保持原样
- 如果扰动 z,就是训"L1 路由对 regime 噪声鲁棒",风险是 L1 学不到稳定分类
- 对应改 `forward_moe_only(seq_hidden, gate_in_adv, z=z.detach())`

### 15.11 A14+ 总结性教训(在 A14 基础上新增)

- **AMP 环境下任何 autograd.grad 拿中间节点梯度的操作必须 FP32 隔离**,不能指望 scaler 或 nan_to_num 救场
- **改 Dataset __getitem__ 签名必须搜索 `zip(*batch)` 和 collate_fn 的 __call__**,不只 for-loop
- **融合版优先于单变量 ablation**(性能实验语境),除非变差才回头做归因
- **文献"无开源代码"不是小问题**,意味着实现完全基于描述重建,必须定性为"启发式借鉴"而非"论文复现"
- **顶刊架构的两级/多级结构设计必须显式记录每级输入、每级 aux loss、是否共享权重**
- **诊断日志的完整度决定归因速度**:Path 2 W1 结果同时含 gate_usage + expert_output_cosine + α σ + 分年 daily IC,让 "2024 衰退归因 E4 死 expert" 的判断 10 分钟完成

---

## 十五.6、★ A16 验证期工程方法论

### 15.12 同口径多次复现是判断"特征性退化 vs 随机噪声"的唯一手段

**A14+ 误判**:Path 2 W1 第一次跑出 val 0.069 / Sharpe 1.034 / 2024 daily IC 0.0185。当时把 Path 2 标为"部署成功"。

**A16 修正**:第二次同 config 跑出 val 0.081(涨 18%)/ Sharpe 0.910(**跌 12% 跌破阈值**)/ 2024 daily IC 0.0131(**比第一次更差**)。两次复现的差异 Sharpe 0.124 是单窗口随机性常态(80 epoch 早停后 best_state 的随机种子敏感性),但**2024 持续退化在两次都出现,证明这是 Path 2 设计本身的特征,不是噪声**。

**通用原则**:
1. 单窗口实验**至少跑 2 次**才能区分"特征 vs 噪声"。整体 Sharpe ±0.1 是常态,单年 daily IC 一致方向偏移才是信号
2. 跑了 1 次的结果不能进入"已成功"状态机,只能进入"已观察"
3. ensemble W1-W7 是另一种降噪手段,但成本是 ~7×,复现是 1.5× 成本 + 完全独立验证

### 15.13 evaluation 口径要看清楚分子分母

**A16 发现**:Sharpe 0.91 看着平庸,但同时整体年化收益 22% / 累计超额 152%,看似矛盾。深查发现:
1. `_run_single_window_backtest::sharpe = (cum^(1/years) - 1) / (std × √252)` 是**几何年化分子 + 算术年化分母**的混合口径,分子带 volatility drag 折扣,分母不带
2. 分年 sharpe 用 `mean × √252 / std` 是标准算术口径
3. 两口径不一致,整体被人为压低 ~5%

**更深层**:Sharpe 是绝对收益口径(vs 无风险利率),策略 corr=0.976 / beta=0.94 时分母里塞了大量基准 beta 波动。剥 beta 后残差 IR=1.93 才是真实 alpha 能力。

**通用原则**:
1. 看 Sharpe 时同时算 IR(残差 std 算分母),两者差距大说明策略 beta 暴露重
2. backtest 输出必须包含 `beta_to_benchmark` / `info_ratio` / `alpha_annual_pct` / `residual_std_annual_pct` 四个字段
3. 项目阈值表 "Sharpe ≥ 1.0" 必须扩展为 "Sharpe ≥ 1.0 **或** IR ≥ 1.5",否则策略提升分散度(降 beta)会反而看起来 Sharpe 退步

### 15.14 跨模块全局变量同步:universe ↔ benchmark 配对的隐性 bug

**现象**:mera_v6 主 pipeline 默认 `UNIVERSE_CSV = "cons_csi1000.csv"`,BENCHMARK 通过 `from native import BENCHMARK as NATIVE_BENCHMARK; BENCHMARK = NATIVE_BENCHMARK` 继承 native 默认 SZ399303(国证 2000)。**主 pipeline 实际跑出的回测一直在用错基准**,但因为 daily_detail.csv 的 mkt_ret 是 SZ399303 收盘价、cum_excess 是策略 - SZ399303,三者**自洽**所以从未触发任何错误。

**diag 脚本另一种自洽错配**:diag 的 `_PARAMS_TO_SYNC` 把自己定义的 `UNIVERSE_CSV = "cons_gz2000.csv"` 覆盖到 pipeline 模块,所以 diag 实际跑的 universe 是 gz2000;这套 universe 与 SZ399303 配对**也是自洽**。所以 §15.9 文档里所有数字(A13 baseline / Path 2 W1 ① / Path 2 W1 ②)都内部自洽,**没有错配数字,只有文档描述与实际口径不符**。

**A16 发现链**:
1. 用户问"我选股域不是 1000 吗,对标超额应该是中证 1000 啊?" → 触发查 UNIVERSE_CSV
2. grep `UNIVERSE_CSV` 发现 mera_v6 的是 csi1000,diag 的是 gz2000,**两个文件不一致**
3. 进一步发现 diag 通过 `_PARAMS_TO_SYNC` 覆盖 mera_v6 的设置
4. 主 pipeline 的 `BENCHMARK = NATIVE_BENCHMARK` 是另一道隐性问题
5. 决策切到 csi1000 + SH000852 全面对齐

**通用原则**:
1. **跨模块全局变量同步必须显式声明配对约束**。`_PARAMS_TO_SYNC` 这种"批量 setattr"模式有隐藏风险,因为新加变量后忘了纳入同步会被默默忽略
2. **强配对参数(universe ↔ benchmark / hidden_size ↔ pred_head_dim / N_MACRO×N_MICRO == NUM_EXPERTS)必须运行时 assert**
3. **跨文件常量继承(`BENCHMARK = NATIVE_BENCHMARK`)危险**,应改为显式赋值或参数传入

### 15.15 Resume 逻辑必须无差别覆盖所有训练入口

**现象**:pipeline main loop L2941 早就有 resume 逻辑(ckpt 存在 → 加载 art → 跳过训练)。但 `mera_single_window_diag_v6.py::train_single_window` **没有同款逻辑**。融合版 W1 训完 154 epoch (~7h)、checkpoint 存好后,诊断阶段崩溃(HierarchicalMoE 嵌套 ModuleList 不兼容旧诊断函数),整个训练成果差点报废。

**为什么差点丢**:第一反应是写 `resume_from_checkpoint.py` 独立脚本,复刻 art 重建 + 诊断 + 回测三步。这个方案的工程量大且和 diag 主路径分叉。用户提醒"diag 不能直接复用 pipeline 的 resume 逻辑吗?"才发现 pipeline L2941 现成的 25 行代码完全可以套用进 diag。

**通用原则**:
1. **训练入口(`train_*`)必须无差别支持 resume**。任何只在 main loop 而不在 diag 入口的 resume 逻辑都是 bug
2. **崩溃恢复脚本应是 diag/pipeline 内置能力,不应是独立救火脚本**
3. **Checkpoint 设计要兼容多入口**:fields 用 dict + version 字段,新版加字段不破坏旧版加载

### 15.16 A16 总结性教训

- **单次实验只是"已观察",复现 2 次后才能进入"已成功"状态机**(A15 Path 2 误判教训)
- **Sharpe 是绝对收益口径,IR 才是 alpha 能力口径**。多头 only 高 beta 策略的 Sharpe 会被 beta 稀释,必须并报
- **强配对参数必须运行时 assert**,跨模块全局变量同步必须显式声明配对约束
- **训练入口必须无差别支持 resume**。任何只在主 pipeline 而不在 diag 入口的 resume 逻辑都是 bug
- **基准切换是不可逆的口径变化**,所有历史数字必须标注口径才能比较;不要为了"对齐 universe"重跑历史,但要清晰标注
- **接续设计的诊断函数(如 `_get_expert(moe, e_idx)` 兼容 flat 和 hierarchical MoE)比硬分支更可维护**:单点函数 + dispatcher 模式
- **融合版三件套(Path 2+3+4)首战证实"性能实验融合版优先"路线正确**:成功时直接拿到合并增益

---

## 十五.7、★ A17 代码精简重构与 Helper 抽取期工程方法论

### 15.17 代码冗余的真假之分

**A17 经验**:`qlib_pipeline_mera_v6.py` 在 A16 末尾达到 3235 行后,看起来很长,本能想"全面重构"。但逐行甄别后发现冗余分两类:

1. **真重复(可抽公因子)**:
   - `OfficialAlignedMERA.forward` / `.forward_train` / `.build_gate_input_with_regime` 三处都有同一段 `RETRIEVAL_ABLATION` 三分支(`"none"` / `"random"` / `"replace"`)+ 可选 regime z 调制(film/concat/off 三模式)的代码,共 ~90 行重复 → 抽 `_build_gate_in_modulated`
   - `SparseSequenceMoE.forward` 与 `HierarchicalMoE.forward` 的 L2 micro 阶段都有同一段 scatter-gather(按 dispatch_idx 排序 → 逐 expert 切片 forward → inverse permutation 还原 → 按 gates 加权求和),共 ~70 行重复 → 抽 `_moe_dispatch_topk`
   - 这两处是真重复,语义零差别,抽完之后两个调用点都变成单行调用,且 docstring 能完整说清楚做什么

2. **假冗余("长"是合理代价)**:
   - `OfficialAlignedMERA` 392 行:13 维异构输入 + Path 2 / Path 3 / Path 4 三件套同文件可切换,本身就是 13×N 段逻辑,长度合理
   - `FeaturePreprocessor` 253 行:6 字段 × 3 stage(stage 0 / A 行业中性化 / B 全局 z-score)+ in-place / out-of-place 双模式,展开就这么长。强制压缩会破坏可读性
   - 这些"长"不是冗余,是项目演化的合理代价,**不抽**

**通用原则**:看到一个函数长,先问"如果我抽出来,调用方代码是会变短(真重复),还是只是把同样的代码挪了个地方(假冗余)"。后者就别碰。

### 15.18 长函数瘦身的 ROI 排序

A17 优先级排序(从高到低):

1. **`train_mera_window_v5` 560 行**:真问题。其内部 ~25 行 pretrain 加载、~20 行 WIP resume、~30 行最终 ckpt save 都是**纯 IO 块,与训练循环主体解耦**,抽出后训练逻辑(FGSM / loss / val / 早停)作为单一焦点更清晰。**ROI 高,先做**
2. **`main()` 160 行**:大部分是配置打印 + window 调度 + backtest stats 打印 + 五面板绘图。也是纯 IO + 纯组装。**ROI 中,顺手做**
3. **`build_alpha360_expressions` 140 行硬编码模板**:6 字段 × N lag 全部展开成 if-else,可改成 13 字段 spec-table + `_ref(field, lag)` helper + 单 loop。**ROI 中**(改完只是更易维护,不影响行为)
4. **OfficialAlignedMERA / FeaturePreprocessor 长**:见 15.17,假冗余,**不抽**

**反例**:有诱惑想把 `_make_loss` 里 std/adv 双路 FGSM 也合并成一段,但 std 路径 backward 一次、adv 路径需要双 forward 取中间梯度后 backward 第二次,两条路径**控制流不同**,合并会触发 §15.7 的 AMP FP32 隔离踩坑。所以**禁动**。

### 15.19 抽公因子前必须做等价性测试

**A17 实操**:`build_alpha360_expressions` 重写从硬编码模板改为 spec-table,虽然字符串拼接逻辑直观看起来一样,**但顺序、空格、引号位置任何细节差异都会导致下游 Qlib 缓存 key 失效,触发 ~3h 的全量重建**。所以重构前必须写等价性测试:

```python
# /home/claude/test_equiv.py
old_exprs, old_names = old_module.build_alpha360_expressions()
new_exprs, new_names = new_module.build_alpha360_expressions()
for i, (a, b) in enumerate(zip(old_exprs, new_exprs)):
    assert a == b, f"expr[{i}]: {a!r} vs {b!r}"
```

**通用原则**:任何"看起来等价"的字符串/表达式生成函数,重构后必须做字符串级逐字节比对,而不是"靠肉眼 review 觉得一样"。下游缓存系统对**字符串级**敏感。

### 15.20 重构时严禁动的边界(项目级承诺)

抽 helper / 拆函数时,以下边界**严禁动**(违反则视为回归 bug):

1. **顶部参数区**:项目原则,所有可配置项一律硬编码在 `.py` 顶部(§1.2 "★★ 所有脚本必须支持 PyCharm 直接 Run")
2. **std / adv 双路 FGSM FP32 训练循环主体**:A14+ §15.7 写过血泪注释("v3 FP32 隔离根治"),改一行就可能踩 FP16 g.norm overflow 的坑
3. **val loop**:早停依赖 + ckpt 选择全局口径统一
4. **`FeaturePreprocessor.transform` 阶段变换内部**:A13 in-place 修改的语义责任在调用方,内部数学算法不能动
5. **任何 public 名字**(类名、函数名、模块级变量):诊断脚本 47 个 `setattr(_pipe, name, value)` 同步点依赖
6. **`_save_window_checkpoint` 写入的 config dict 字段名**:checkpoint 反序列化向后兼容依赖

抽 helper 时优先策略:**先抽与训练循环主体解耦的纯 I/O 块**(pretrain 加载 / WIP resume / 最终 ckpt save / 配置打印 / 五面板绘图),不去碰核心训练数学。

### 15.21 Helper docstring 是防回归文档

**A17 经验**:每个新 helper 加 10-15 行 docstring,说明:
- *为什么* 抽出来(原来在哪里、与哪些函数共享)
- *做什么*(输入输出契约 + 关键不变量)
- 与原版语义对应关系(哪一段被替换)
- 任何"不能动"的坑(如 `_save_window_checkpoint` 的字段名)

这与项目历来的 `★ A12 FIX` / `★ A13 in-place 修复` / `★ A14-fix-v3 FGSM FP32 隔离` 注释风格一致。**docstring 不是装饰,是防止半年后自己回来动这个 helper 时踩老坑的防回归文档**。

### 15.22 Spec-table 重写规则

**A17 经验**:`build_alpha360_expressions` 从 60 行硬编码模板改为 13 个字段的 `(out_name, lag_offset, expr_builder lambda)` 元组 + 单 loop:

```python
# A17 spec-table 形式(伪代码,真实实现在 qlib_pipeline_mera_v6.py)
_FIELD_SPECS = [
    ("$open",   0, lambda lag: _ref("$open",   lag)),
    ("$close",  0, lambda lag: _ref("$close",  lag)),
    # ... 13 行
]

def _ref(field, lag):
    """统一 lag=0 与 lag>0 两种写法。"""
    return field if lag == 0 else f"Ref({field}, {lag})"

def build_alpha360_expressions():
    exprs, names = [], []
    for out_name, lag_offset, expr_builder in _FIELD_SPECS:
        for d in range(SEQ_LEN):
            exprs.append(expr_builder(d + lag_offset))
            names.append(f"{out_name[1:]}_d{d}")    # 与原版命名规则字符串一致
    return exprs, names
```

**通用原则**:大量结构化重复(N 字段 × M 时间步)写成 spec-table + 单 loop,比 N×M 行硬编码模板易维护。但要注意:
- spec-table 的元组顺序必须与原版输出顺序一致(下游 cache key 依赖)
- 字符串拼接 helper 内 lag=0 / lag>0 两种写法要在同一函数里(防止两处分别变更不一致)
- 必须配等价性测试(见 §15.19)

### 15.23 A17 总结性教训

- **冗余分真假**:真重复才抽公因子;假冗余(异构展开 / 项目演化合理代价)别碰
- **长函数瘦身按 ROI 排序**:560 行函数内的纯 IO 块抽出去 ROI 最高;不要为了"凑短"去拆紧耦合的核心训练逻辑
- **任何"看起来等价"的字符串生成必须做等价性测试**:Qlib 表达式 / cache key / 序列化字段名都对**字符串级**敏感
- **重构有边界,不是所有代码都该动**:顶部参数区 / FGSM FP32 训练主体 / val loop / public 名字 / ckpt config 字段名,这五样**任何阶段都不动**
- **Helper docstring 是防回归文档**,与历来 `★ A12 FIX` 等注释风格统一,半年后自己回来动这段时它救你
- **重构验证只是部分等价**:`build_alpha360_expressions` 字符串级一致只证明该函数行为不变,完整训练-推理路径(MoE 路由 / FGSM 数值 / val IC)还需要实跑一次单 window 比对 A16 baseline(val IC 0.1087 / OOS daily IC 0.0358 / Sharpe 1.084)。**这是 A17 的 PENDING 验收项,不能跳**

---

## 十五.8、★ A18 预训练适配与配套脚本规约期工程方法论

### 15.24 配套脚本必须共享 OUTPUT_DIR / CACHE_DIR / 关键路径常量

**A18 经验**:`mera_pretrain_v6.py` 第一次落地时误用了 v5 时代的老路径 `OUTPUT_DIR = PROJECT_ROOT / "Output" / "Qlib_DL"`,而 pipeline `qlib_pipeline_mera_v6.py` 是 A12 后规范的 `Output / Mera_Output`。后果两层:

1. **Cache 路径不一致** → pretrain 在 `Qlib_DL/alpha_cache_v6/` 找不到 cache → 触发完整重生成,耗时 6 分钟 + 36GB 写盘 + 一份与 pipeline 完全重复的副本
2. **ckpt 输出路径不一致** → pretrain 写到 `Qlib_DL/mera_v6_pretrain/`,pipeline 在 `Mera_Output/mera_v6_pretrain/` 找不到 → 训练时 raise `FileNotFoundError`,但触发点是窗口 N 而非窗口 1(因为 W1 是从老训练 ckpt resume 的,不走 pretrain 加载),迷惑性高

**通用原则(项目级承诺)**:
- 任何**配套两个或以上脚本共享同一份数据 / 同一系列 ckpt** 的设计,必须把根路径常量(`OUTPUT_DIR` / `DATA_DIR` / `CHECKPOINT_DIR`)抽到**单一真相源**(优先 `project_config.py`,其次脚本顶部加 `# ★ 必须与 X 一致` 注释 + 字面值同步)
- 复查方法:`grep -n "^OUTPUT_DIR\|OUTPUT_DIR =" *.py`,一行命令对齐所有入口脚本的根路径定义;**字面值必须完全一致**(包括末尾斜杠 / 大小写)
- 配套脚本顶部 import 后第一时间打印一遍 `print(f"[Config] OUTPUT_DIR={OUTPUT_DIR}")`,运行时立刻能发现路径走偏

**为什么 v5 时代没暴露**:v5 pipeline `qlib_pipeline_mera_v5.py` 与 `pretrain_mera.py` 都用 `Qlib_DL`,字面值碰巧一致。v6 升级 `Mera_Output` 后两个脚本不同步,bug 浮出水面。这条原则是 v6 演进过程中**新发现**,A12 [M6] 改输出目录时漏检了配套脚本。

### 15.25 strict=False 加载必须配 critical key 显式校验

**A18 防回归发现**:`model.load_state_dict(state_dict, strict=False)` 是 PyTorch 的"宽容加载"接口,设计本意:**ckpt 比 model 少的 key**(下游 model 多了模块如 market_gate / moe / pred_head)进 `missing` 列表,**ckpt 比 model 多的 key**(命名错误或 model 砍了模块)进 `unexpected` 列表。`strict=False` 让前者不 raise,但**后者也不 raise**——只是 silently 忽略。

但有个隐藏坑:**如果 pretrain 端拼装 state_dict 时 key 命名拼错了**(例如把 `similar_proj.0.weight` 错写成 `similar_proj.proj.0.weight`,本质是 SimilarProjector 内层 nn.Sequential 未做 prefix 映射),strict=False 会把所有错命名的 key 全塞进 `missing`,日志只显示 `unexpected=0` 看起来一切正常,实际 model 等于完全随机初始化。这种 bug 在 OOS Sharpe 跌了之后才暴露,事后归因极困难。

**A18 防回归方案**(普适所有 strict=False 加载场景):

```python
missing, unexpected = model.load_state_dict(ckpt_state, strict=False)

# 1. unexpected 必须为空(>0 = pretrain 端 key 命名拼错)
assert len(unexpected) == 0, f"unexpected keys: {unexpected[:5]}"

# 2. 关键 key 必须真的不在 missing 里(否则等于完全随机初始化但日志不报错)
_critical = [
    "input_bn.weight",                                    # 输入归一化
    "input_proj.weight",                                  # 输入投影
    "encoder.layers.0.self_attn.in_proj_weight",          # 至少第一层 attention
    "similar_proj.0.weight",                              # 检索投影
]
_missed = [k for k in _critical if k in missing]
assert not _missed, f"critical keys 在 missing 里: {_missed}"
```

**关键 key 选择标准**:覆盖每个被预训练的子模块的"代表 key"(每个 nn.Module 至少一个 weight),且必须是名字稳定不会因 PyTorch 版本变动的(`encoder.layers.0.self_attn.in_proj_weight` 是 PyTorch 1.0 起不变的 nn.MultiheadAttention 内部命名)。

### 15.26 ckpt 必须带 version + config 字段做硬校验

**A18 经验**:不带 version 字段的 ckpt 在版本演进时会产生静默 bug。例:用户不小心把 v5 老 ckpt(`input_proj` 6→128,360 维 sim_proj)拷到 v6 ckpt 路径下。`load_state_dict` 在维度不匹配时会 raise `RuntimeError: size mismatch for input_proj.weight: copying a param with shape torch.Size([128, 6])`,这条触发是好事,但堆栈是 PyTorch 内部错误,定位需要回查上下文才能发现"哦这是 v5 ckpt"。

**改进**:ckpt 顶层必须带 `version`(整数,e.g. `version=6`)与 `config`(字典,含所有维度/字段命名/超参)字段。加载方逐项校验,业务层 raise 业务级错误。

**A18 ckpt 字段规约**:

```python
torch.save({
    "encoder_state": combined_state,            # state_dict 主体
    "preprocessor": pre.to_state(),             # 含 industry_map / pe_neutralize_stats / pb_neutralize_stats
    "best_loss": best_loss,
    "version": 6,                               # ★ 硬校验字段
    "config": {                                 # ★ 维度 / 字段 / 超参
        "input_size": 13,
        "seq_len": 60,
        "flat_dim": 780,
        "retrieval_flat_dim": 360,
        "field_names": [...],
        "use_industry_neutralize": True,
        "align_weight": 0.5,
        # 模型超参
        "hidden_size": 128, "num_layers": 2, "num_heads": 4, "dropout": 0.30,
    },
}, ckpt_path, pickle_protocol=4)
```

**加载方校验链**:
1. 文件存在性(否则 `FileNotFoundError` + 提示去跑 pretrain)
2. `ckpt["version"] == EXPECTED_VERSION`(否则 raise 业务级 `"v6 model 需要 v6 ckpt, 当前 ckpt version=5"`)
3. `ckpt["config"]["input_size"] == INPUT_SIZE` 等关键维度逐项校验(防 cache stale 或参数改了忘重 pretrain)
4. critical key 校验(见 §15.25)

**通用原则**:任何序列化产物(ckpt / pkl cache / json meta)都应在顶层带 `version` + `config` / `meta` 字段,作为版本演进的硬校验入口。这与 §15.26 是同一个思路在不同载体上的应用——pkl cache 已经在 A12 实现(`alpha_meta_p2_d13.json`),A18 把 ckpt 也升级到这个标准。

### 15.27 文件命名规范应该按"业务模块所有制"分组,不是按动作分组

**A18 决策**:v5 时代命名 `pretrain_mera.py`(动词 + 模块)让所有 pretrain 脚本聚到一起,但项目其实没有第二个 pretrain。改为 `mera_pretrain_v6.py`(模块 + 动词 + 版本)让所有 mera 资产聚到一起,与 `qlib_pipeline_mera_*` 管线本体并列,与 `ensemble_*.py` 融合版系列对应。

**通用原则(项目级承诺)**:
- 文件命名按**业务模块所有制**分组,不是按**动作类型**分组
- 一个业务模块的所有配套工具用同一前缀(`mera_*` / `ensemble_*`),让 IDE 文件树里所有相关文件聚在一起
- 入口文件(管线 / 主交易脚本)可以用复合命名(`qlib_pipeline_mera_*.py` / `qlib_trading_mera_*.py`),反映"在 qlib 框架内的 mera 入口"语义
- 命名变更必须**全文搜索引用同步**:`grep -rn "old_name\.py" .` 找到所有错误信息 / 注释 / docstring 里的引用,一并替换

**A18 项目级文件命名规约**:

| 文件类型 | 命名模式 | 例子 |
|---|---|---|
| 管线本体 | `qlib_pipeline_<module>_<version>.py` | `qlib_pipeline_mera_v6.py` |
| 交易主入口 | `qlib_trading_<module>_<version>.py` | `qlib_trading_mera.py` |
| MERA 配套工具 | `mera_<purpose>_<version>.py` | `mera_pretrain_v6.py` / `mera_single_window_diag_v6.py` / `mera_hb_grid_search.py` |
| 融合版工具 | `ensemble_<purpose>_<version>.py` | `ensemble_*.py` 系列 |
| 通用工具 | `<purpose>.py`(无前缀) | `inspect_mera_checkpoint.py` / `exposure_constraint.py` |

### 15.28 A18 总结性教训

- **配套脚本必须共享根路径常量**,字面值不同步是隐性 bug 源(§15.24)
- **strict=False 加载是 PyTorch 的常见陷阱**,key 命名错误时静默全 missing,必须配 critical key 显式校验(§15.25)
- **ckpt 必须带 version + config 字段**,版本演进硬校验,与 pkl cache 的 meta 字段同思路(§15.26)
- **文件命名按业务模块所有制分组**,不是按动作类型(§15.27)
- **A18 三个工程坑都是上一版没暴露的隐性 bug**:`SimilarProjector` 维度(被 user 截图警告)、`OUTPUT_DIR` 错配(第一次跑出 `FileNotFoundError`)、命名规范(user 主动提议)。**没有用户的快速反馈,都不会发现** — 工程实现质量的"最后一公里"必须靠真实运行 + 用户审视
- **预训练实质实现(A11 → A16 → A17 → A18 三轮列首要任务终于落地)**:不是技术问题难,是 v6 升级期 13 维输入 + 行业中性化 + 检索 bank 360 维降级三个变化叠加,做配套需要把 §3.3 v5→v6 表里的 [M0]-[M3] 全在 pretrain 端镜像一遍;A18 真正的工作不是写 SimCLR,而是确保 pretrain 看到的输入分布与 fine-tune 看到的分布完全一致

---

## 十五.9、★ A19 合规过滤继承期工程方法论

### 15.29 上游重构波及检测应优先于自身重构

主策略侧 Phase 53 重构 ST/IPO 过滤(`find_st_stocks` + `filter_ipo` → `qualified_mask.pkl`),MERA 端 import 这两个共享函数,理论上要么:(a) 跟随重构,(b) 切自己的过滤,(c) 验证上游接口删除是否影响。A19 选择(c),发现两个 import 站点都是 `_ = find_st_stocks()` 死代码模式 — 调用后立即丢弃返回值,无副作用,意图不明(可能是早期想做 cache warmup,但函数无副作用)。结论:MERA 不需要任何过滤逻辑,直接删 import + 删死代码即可,改动量从预期"几十行接 mask"降级为"4 行删除"。

**方法论**:每次主策略侧重构之后,MERA 端先做 `grep -nE "(被删函数列表)" mera_*.py` 全文扫描。若所有出现都是死代码或注释,直接清扫;若有真实业务依赖,再考虑接新接口。

### 15.30 隐式继承优于显式 plumbing

MERA 不需要直接读 `qualified_mask.pkl`,因为 MERA 调 `run_qlib_backtest`,而 `run_qlib_backtest → HBTopKStrategy.generate_trade_decision()` 内部已经在选股阶段调 `get_qualified_universe(pred_start_time)`。A19 没在 MERA 加任何合规相关代码,但 MERA 的回测**自动享用**了主策略 Phase 53 的合规改进。

**方法论**:对于"调用层 → 策略层 → 选股层"这种层级架构,合规/过滤这种横切关注点应**只在最低层(选股层)实现一次**,上层不参与 plumbing。判断原则:这个过滤逻辑是否依赖**调用方传入的特定 context**? 如果不依赖(只需要 pred_date 这种公共信息),就在底层实现;如果依赖(需要调用方业务字段),才上提到调用方。本 case 中合规判定只需要 pred_date,所以放在 `HBTopKStrategy` 内部最佳。

### 15.31 死代码识别的安全删除流程

`_ = func()` 模式(调用后立即丢弃返回值)的死代码,在删除前需要确认:

1. **函数是否有副作用?** Python 函数若只 return 不修改全局状态、不触发 IO、不发请求,则无副作用。`find_st_stocks()` 读 pkl 文件无 side effect。
2. **是否有 module-level cache?** 比如 `lru_cache` 装饰器,删调用会让 cache 不预热。`find_st_stocks` 无 cache。
3. **是否有性能影响?** 删除 cache warmup 调用可能让首次正式调用变慢,但本 case 后续不再调用所以无影响。

**方法论**:`_ = expr` 模式 + "无副作用 + 无 cache + 无性能依赖"三确认后可以安全删。这种模式通常源于早期开发遗留(开发者写了调用想观察输出但忘了删),不是有意设计。

### 15.32 调用方与被调方签名变更的兼容性检测

主策略 Phase 53 把 `run_qlib_backtest` 的 `listing_dates=None` 形参删了。MERA 端的两处 `run_qlib_backtest` 调用本来就**没传**这个参数,所以删除上游形参对 MERA 零影响。

**方法论**:上游函数删除某个**默认参数**(有默认值的关键字参数)时,先 `grep "run_qlib_backtest(.*listing_dates=" mera_*.py`,看下游是否真的传过。若全部没传,签名变更对下游零影响,无需协调改动。这与"删除位置参数"或"删除必传参数"完全不同(后者会立即编译/运行报错)。

### 15.33 A19 总结性教训

A19 的实际工作量(约 10 行删除)远小于上游 Phase 53 的工作量(约 600 行重构),但**做之前不知道**。预估的时候按"最坏情况(完全镜像上游重构)"估,做的时候按"实际情况(继承 + 死代码删除)"。A18 PENDING 验收项继承到 A19 PENDING 不变化:`_print_backtest_stats` IR + beta 口径仍未做;pretrain 实跑验证仍未做;diag 同步 pretrain 加载逻辑仍未做。**A19 不引入新 PENDING**。

---

## 十六、★ A20 MERA 信号再归因(主线 KB 66 联动)★★★ A20 新增

> 本章是 A20 阶段唯一新增的实质内容,来自主线 KB 66 对 mera 实验数据(v3_24f 实验 B 80 格 + delta + 逐年)的取证复核。它评估的是 **mera 信号的性质与价值**(horizon / 有无 alpha / 该怎么用),不涉及 mera 模型架构本身(后者见 §三、§三B)。`pred_score.csv` 是 mera 模型层(本文档主体)与主线评估(本章)的共享接口。

### 16.1 三个被撤销 / 修正的论断(原属主线 KB 65)

主线 KB 65 对 mera 的三个论断在 baseline 完整逐年数据下站不住:

**① ~~"mera_score importance 排名 1/24 → 信号真实有 alpha"~~ → 撤销**

importance 衡量"模型用了多少",不是"用对了多少"。与实际 OOS 改善的相关性全部为负:

```
corr(importance, 全期 Δsharpe)     = -0.18
corr(importance, 全期 Δ累计超额)   = -0.21
corr(importance, 2026 Δsharpe)     = -0.33
corr(importance, 2026 Δ累计超额)   = -0.36
```

且 importance **完全由 label period 决定**(period 5→7.1% / 10→9.4% / 15→10.7% / 20→12.2% / 30→16.4%,与 hb/top_k 无关,因同 period 共享一个训练好的 XGB)。**长 period 训练 → XGB 越"信"mera → OOS 越差**——典型的训练 importance ≠ 泛化 importance 过拟合形态。一个噪声大但截面 z-score 后分布广的因子很容易在 split-gain importance 上排第一。**正确的 alpha 诊断**见 16.3,而非 importance。

**② ~~"实盘候选格加 mera,2026 超额砸到 0 → mera 伤到 2026 强超额"~~ → 修正为"每年都拖累"**

主线实盘候选格 p15/.20/k50 加 mera 的全期累计超额 Δ = −56.6pp,拆解:

```
2026 (93 天) 部分:   -12.4pp
2019-2025 部分:      -44.2pp   →  均摊 -6.3pp/年
```

mera 在这格**每年都拖累**,2026 只是最狠一年。本质是该长 period(period=15)配置与短 horizon mera 信号的**结构性冲突**,与 2026 这个特定年份无关。

**③ ~~"换手翻倍是 mera 硬伤"~~ → 修正:grid 中位现象,候选格持平**

翻倍是全 grid 中位现象。实盘候选格 p15/.20/k50 换手 19.9% → 20.0% **基本持平**;14 个双赢格里 6 个换手 ≤25%。翻倍只发生在短 period + 低 hb 区域。换手是 portfolio construction 问题(成本旋钮),可独立处理,不是信号本身的硬伤。

### 16.2 mera v6 的正确画像

**(a)mera 是 short-horizon 信号(核心新发现)。** 在 80 格 +mera 网格中,"全期+2026 双赢格"按 period 分布:period 5:9 格 / period 10:3 格 / period 15:0 格 / period 20:2 格 / period 30:0 格。**12/14 集中在 period ≤ 10**;且四象限里 **0/80 格"全期变差但 2026 改善"**(排除 mera 是 2026 单年运气)。mera 用 ret_2d 短期标签训练,天然 short-horizon——喂给长 horizon(period 15/20)XGB 时被当噪声过拟合(故长 period importance 高但 OOS 差)。这与本文档 §三 mera 的 ret_2d 预测器本质一致。

**(b)mera 的价值是短 period 的全期收益抬升,量级很大。** 用累计超额(pp)看 mera 增量比用 Sharpe 看剧烈得多:p10/.10/k30 在 baseline 口径全期 Sharpe 仅 **0.914**(弱),叠 mera 后累计超额从 **120% 拉到 236%(Δ +116pp,近翻倍)**;而 period-15 高 hb 区(baseline 调好的强配置)mera Δ 深度为负(−56~−63pp)。所以 mera 不是"救弱配置的噪声",而是**在它适配的短 horizon 上有真实、大幅的全期收益增益**。

**(c)mera 不是 2026 题材。** mera 在 2026 的**增量**几乎整片为负(80 格 Δ2026 超额中位 −4.4pp),只有 period-5 两三格勉强正(最多 +6.1pp);连全期明星 p10/.10/k30 的 2026 mera 增量也仅 **+0.2pp**(baseline 该格 2026 本就 +13.1pp,2026 的强是 baseline 配置自带的)。且 2026 仅 93 个交易日(主线 KB 66 §四十三),不应据此判断。

### 16.3 下一步(按 ROI 排序)

**P0(零成本,~1 天)— 正交化诊断,把"共线/有 alpha"从推断变数据**(不需重训):
1. mera_score 对 23 因子的线性回归 R² + 相关矩阵 → R² 高=共线;低=有独立信息,问题在用法。
2. 取残差 `mera_resid`(mera_score 减 23 因子可解释部分)作为新因子,看其 OOS RankIC 是否显著>0。
3. **按 period 分层算 mera 边际 IC**。预测:短 period(5/10)边际 IC 显著、长 period(15/20)接近 0 或负——与 importance↔Δ 负相关一致。
4. 留一法:去掉 mera_score 后短 period 配置 OOS 掉多少。

**P1 — ensemble 验证(取代"mera 当因子")**:短 period mera 子模型 + 长 period baseline 主模型 ensemble。二者 horizon 不同、相关性应较低,理论上两个 Sharpe ~1.1、相关 <0.5 可合成 Sharpe 1.4+。**前置是 P0 的相关性诊断**(相关 <0.5 才有显著合成增益)。评估用 walk-forward,2026 仅旁证。**这优先于 v7——便宜的先做。**

**P2 — 换手治理**:对 mera 适配的短 period 强格(如 p10/.10/k30,换手 41%)加 turnover penalty / holdings smoothing,把换手压到 25-30%,使其全期增益可净落地。

**P3 — mera v7+ 信号质量提升**(回到本文档 §十二.5 路径表):**v6 短 period 价值未榨干前不做 v7**。若 P0/P1 表明 v6 短 period 已可用(ensemble 达标),v7 为锦上添花;若 P1 失败(相关性太高)再做 v7 降相关。v7 方向仍按 §十二.5 路径 4(行业 one-hot / 北向 / 换手异动),目标是**降低与 23 因子共线度 + 稳定跨年表现**,而非单纯抬 Sharpe。

### 16.4 与主线 KB 65 决策的差异小结

| 维度 | 主线 KB 65 结论 | A20 修正 |
|---|---|---|
| importance 1/24 | alpha 证据 | 过拟合伪信号(与 OOS 改善负相关) |
| mera horizon | 未区分 | short-horizon(period≤10),长 period 被过拟合 |
| "2026 拖累" | 2026 单年反向 | 每年都与长 period 配置冲突 |
| mera 价值 | 全期有信息量但共线 | 短 period 有大幅全期收益增益(p10 近翻倍) |
| 推荐用法 | 因子(实验 B),不上线 | 因子用法错 → 改 ensemble |
| ensemble | 暂缓(v6 自评 < 主策略) | 优先路径(价值在低相关,非子策略单独强) |
| 下一步顺序 | 先做 v7 提质 | 先做零成本正交化诊断 + ensemble,v7 殿后 |

> 与本文档 §十二.5 / §战略定位的衔接:A20 的发现给"mera 做低相关信号源融进 ensemble"提供了数据背书(short-horizon + 低相关),并把 v7 信号增强从"立即第一步"重排为"ensemble 验证后条件触发"。

---

## 文档版本历史

- **★ v-A19 (2026-04-29)**:MERA 端 ST/IPO 过滤死代码清理 + 隐式继承主策略 `qualified_mask` 合规过滤阶段。主策略侧 Phase 53 把 `find_st_stocks` + `filter_ipo` + `apply_st_safety` 三个共享函数全部删除,改用基于 `qualified_mask.pkl`(date×code bool 矩阵,合规判定 = 上市≥365天 AND 过去365天无 ST)的合规过滤体系。MERA 端 `qlib_pipeline_mera_v6.py` 和 `mera_single_window_diag_v6.py` 各两处 `find_st_stocks` 引用全部识别为**死代码**(`_ = find_st_stocks()` 调用后立即丢弃,无副作用,可能是早期 cache warmup 遗留)。两个文件各删 2 行:`from qlib_pipeline_native import find_st_stocks` 删,`_ = find_st_stocks()` 删。MERA **不直接读 `qualified_mask.pkl`**,合规过滤通过 `run_qlib_backtest → HBTopKStrategy.generate_trade_decision() → get_qualified_universe(pred_start_time)` 隐式继承,即使中证1000 在某次再平衡日还没把刚被 ST 的股剔出去,mask 兜住。`run_qlib_backtest` 签名删除 `listing_dates=None` 形参不影响 MERA(MERA 调用本来就没传)。A19 工程方法论新增五条(15.29-15.33):上游重构波及检测优先于自身重构 / 隐式继承优于显式 plumbing / 死代码识别的安全删除流程 / 调用方与被调方签名变更的兼容性检测 / A19 总结性教训。**A19 不引入新 PENDING**,继承 A18 验收项原样不变。

- **v-A18 (2026-04-29)**:v6 自监督预训练落地 + Pipeline ckpt 加载链路完整化 + 文件命名规范统一阶段。`mera_pretrain_v6.py` 实质实现(A11/A16/A17 三轮列首要任务终于落地):1071 行,SimCLR 双视图增强 + cross-space 360↔[60,13] align 双任务联合训练,与 v5 `pretrain_mera.py` 完全独立并存。`SimilarProjector` 输入 360 维(v6 [M3] 设计选择,非 780),`PretrainDataset.flat_x_price` 切出量价 6 字段产生 `[N, 360]`。`FeaturePreprocessor` 完整搬过来含两阶段(逐字段非线性变换 + 行业中性化 + 全局 z-score),pretrain 与 fine-tune 看到同分布输入。Pipeline `_load_pretrained_encoder_into` 由 A17 "警告 stub" 升级为真实加载,4 重校验:存在性 / `version==6` / `input_size==13` 与 `retrieval_flat_dim==360` / 关键 key 真的不在 missing(防 strict=False 静默全 missing 隐藏 bug)。`USE_PRETRAINED_ENCODER` 默认 True 不再被强制覆盖。`PRETRAIN_DIR_V6 = OUTPUT_DIR / "mera_v6_pretrain"` 新常量。文件命名规范统一:`pretrain_mera_v6.py` → `mera_pretrain_v6.py`,与 `qlib_pipeline_mera_*` / `mera_single_window_diag_*` / `mera_hb_grid_search.py` 系列对应,与融合版 `ensemble_*.py` 系列对应,pipeline 5 处错误信息引用同步。第一次落地踩过的坑(都是隐性 bug):`OUTPUT_DIR` 错配 `Output/Qlib_DL` → `Output/Mera_Output`(否则 cache 重生成 36GB + ckpt 找不到)、`SimilarProjector` 误用 `flat_dim=780`(被 user 截图警告纠正)、文件命名(user 主动提议改规范)。A18 工程方法论新增五条(15.24-15.28):配套脚本必须共享根路径常量 / strict=False 加载必须配 critical key 显式校验 / ckpt 必须带 version + config 字段做硬校验 / 文件命名按业务模块所有制分组 / A18 总结性教训。**A18 PENDING 验收项**:`_print_backtest_stats` 改 IR + beta 口径(从 A16 §15.13 / A17 待办继承,仍未做);完整 expanding window 系列实跑验证 pretrain 效果(清空 mera_v6_checkpoints/D_m14*/ 重训对照 A16 融合版 baseline daily IC 0.0358 / Sharpe 1.084,验证 A11 在 v5 上观察到的 0.047→0.091 翻倍效果是否在 v6 13 维上重现);`mera_single_window_diag_v6.py` 同步 pretrain 加载逻辑;pretrain 单 window 实测 loss 曲线收集(SimCLR 应从 baseline log(2B-1)≈9.70 收敛到 3.x,align 从 log(B)≈9.0 收敛到 4-6)。

- **v-A17 (2026-04-28)**:`qlib_pipeline_mera_v6.py` 代码精简重构与 Helper 抽取阶段。业务逻辑一行不动,只做真重复抽公因子 + 长函数纯组装/IO 块外移。`train_mera_window_v5` 560→456 行,`main()` 160→31 行,`build_alpha360_expressions` 140 行硬编码模板→13 字段 spec-table + 单 loop(69 行),`HierarchicalMoE` 164→138 行,`SparseSequenceMoE` 104→83 行。新增 7 个顶层 helper:`_build_gate_in_modulated`(合并三处 RETRIEVAL_ABLATION 三分支 + regime z 调制)、`_moe_dispatch_topk`(flat + Hierarchical MoE 共享 scatter-gather,省 70 行重复)、`_load_pretrained_encoder_into`、`_resume_wip_or_init`、`_save_window_checkpoint`、`_print_config_summary`、`_print_backtest_stats`(注释里标 IR/beta 为 A17 后续待办)、`_plot_five_panels`。零风险冗余清理:重复 `import torch` 移除,`from qlib.data import D` 三处内层 import 上提模块顶部。等价性:`build_alpha360_expressions` 用 `/home/claude/test_equiv.py` 对 780 个表达式 + 780 个名字做字符串级逐字节比对,与原版完全一致;47 个诊断脚本依赖的 public 名字全部保留;config dict 字段名不变(ckpt 反序列化兼容)。文件总行数 3235→3325(+90 全部来自 helper docstring 与防回归注释)。A17 工程方法论新增七条(15.17-15.23):代码冗余的真假之分 / 长函数瘦身的 ROI 排序 / 抽公因子前必须做等价性测试 / 重构时严禁动的边界 / Helper docstring 是防回归文档 / Spec-table 重写规则 / A17 总结性教训。

- **v-A16 (2026-04-27)**:融合版三件套首战 + 基准对齐 csi1000 + 参数区分层整合 + 资源恢复加固阶段。Path 2 W1 第二次复现证实 2024 是**特征性负贡献而非随机噪声**;融合版三件套 D_m14_p2_p3_p4 W1 实测(csi1000 + SH000852)val IC 0.1087 / OOS daily IC 0.0358 / IR 6.94 / Sharpe 1.084 / 累计超额 152%,**2024 daily IC 救回 0.0284(+118%)**,无死 expert,所有分年 daily IC 全部改善(仅 2023 Sharpe 退步),Max DD -36.2% 创项目最差;deep evaluation 揭示策略 corr=0.976 / beta=0.94 / 残差 IR=1.93;基准对齐工程(mera_v6 切到 SH000852 / native run_qlib_backtest 加 benchmark 参数 / diag 全面切 csi1000 + 运行时配对 assert);Tag 命名规范重写为 Path X;参数区 9 段分层整合;Diag 同步 pipeline resume 逻辑 + 诊断函数加 `_get_expert(moe, e_idx)` helper;A16 工程方法论新增五条;待办清单**调整为 MERA 自监督预训练为下一步首要任务**。

- **v-A14+ (2026-04-24 后期)**:Path 2 部署 + Path 3 Hierarchical MoE + UMI RankIC loss + BankCollator 5 元组修复。三轮 FGSM 修复(v1 scaler.scale 错加 / v2 FP16 g.norm overflow / v3 FP32 隔离整段 FGSM 数学根治,4× encoder forward → 1.3×);Path 2 W1 val IC 0.069 / daily IC 0.0232 / Sharpe 1.034 过 §7.7 阈值但 2024 daily IC 衰退到 0.0185 + Max DD -30.8% + α σ 4.5× 改善;Path 3 HierarchicalMoE 实现(2 macro × 4 micro);UMI RankIC loss 实现(零架构改动);Dataset 5 元组 + BankCollator 同步;A14+ 方法论新增三条(AMP 梯度 FP32 隔离 / 改 signature 全局追踪间接消费者 / 融合版 vs 单变量 ablation 决策框架);RSAP-DFM 定性"启发式借鉴",确认 RSAP-DFM 无开源代码,实现凭论文文字重建可能 20-30% 偏差。

- **v-A14 (2026-04-24)**:顶刊机制借鉴落地与论文理解诚实化阶段。单窗口诊断脚本扩展内置回测 + HB Sweep v2(7 档 HB,29min) + A13 修复效果确认(val=0.065 / daily=0.0226 / 2024 IC=0.033) + RSAP-DFM 借鉴版 Path 2 实现(pipeline v6 2521→2681 行) + 九篇顶刊论文原文精读并纠错 + §七 大幅重写 + v7+ 路线图重排 + §十五 A14 工程方法论新章。

- **v-A13 (2026-04-23)**:MERA v6 数据管线诊断与关键 bug 修复阶段。split_train_val 按股票切误诊为时序切修复(val IC 0.37 vs OOS IC 0.02 17x 差距)+ Feature 预处理三项 scale 缺陷修复(volume / roe 重尾,turnover_log 公式 ×100)+ MASTER gate expanding → rolling 252 日 z-score(修 OOS α σ 衰减 7 倍)+ FeaturePreprocessor.transform in-place 省 5GB RAM + 检索 bank FP16 加速(3:33→1:30)+ TF32 训练加速 + 诊断脚本 del DataFrames 释放 16GB + 数据对齐验证/Split 验证/Scale 诊断/RAM 探测四件套工具 + A13 工程方法论。

- v-A12 (2026-04-22):MERA v6 架构升级阶段(13 维异构输入 + MASTER gate + 行业中性化 + ETL 集成 + Mera_Output 命名规范)
- v-A11 (2026-04-16):Expert 分化深度诊断阶段
- v-A10 (2026-04-11):D 组 OOS 结果 + Shared Expert Isolation
- v-A9 (2026-04-07):预训练 v3 + 数据预处理修复
- v-A8 (2026-04-03):标签修正 + 短期信号对齐 + 单窗口诊断
- v-A7 (2026-04-01):检索 Ablation + 实验隔离 + MERA Trading
- v-A6 (2026-04-01):训练循环对齐官方 + 预训练工程完善
- v-A5 (2026-03-31):官方代码审查 + 关键参数修正
- v-A4 (2026-03-30):训练速度优化 + 断点续跑
- v-A1~A3:架构建立、参数对齐、Universe 缩放

**v-A20**(Phase 66 联动):新增 §十六 MERA 信号再归因(来自主线 KB 66 对实验 B 80 格逐年数据的取证复核)—— importance 非 alpha 证据(与 OOS 改善负相关 −0.18~−0.36)、mera 是 short-horizon 信号(双赢格 12/14 在 period≤10)、价值是短 period 全期收益增益(p10 近翻倍)、2026 增量几乎全负、用法从因子改为短×长 horizon ensemble、v7 重排为 ensemble-first。完整继承 A19 全部正文,过时处 ~~划线~~ 修正。下一 A 系列文档:做 ensemble 实测或 v7 信号增强时新开。

---

## A21 增量(Phase 71 联动:数据底座迁通联对 Mera 的影响)

> 主线 Phase 71 把数据底座从聚宽切到通联(开关 `project_config.QLIB_REGION`,`"cn_tl"`/`"cn_jq"`)。Mera 端影响如下,均为"数据源换轨",与 A20 及更早的信号/模型结论正交。

- **provider 切换(随开关)**:Mera 的 `.bin` 路径已改为 `QLIB_DATA_DIR = PROJECT_ROOT / "qlib_data" / QLIB_REGION`,翻开关切 cn_tl/cn_jq,不再手工改 `provider_uri`。
- **`market_features_v6.pkl` 通联化(必跑一次)**:该 pkl 是纯 `.bin` 派生(3 个 market 指数 `SZ399303` / `SH000905` / `SH000985` 的 OHLCV → 30 维统计 → expanding z-score)。通联版由 `etl_RN/rebuild_market_features_tl.py` 产——monkeypatch 聚宽 `qlib_converter.QLIB_DATA_DIR → cn_tl` 后调原 `rebuild_market_features_v6()`,零改 converter。**切 cn_tl 后必须全新进程跑一次** `python etl_RN/rebuild_market_features_tl.py`,否则 Mera 读到的是聚宽版/旧版 → 口径或维度不符。那 3 指数已确认在通联 `daily_index_full_tl.py` 的 7 指数清单内(`399303/000852/000300/000905/000903/000906/000985`)。
- **ST / 合规(自动跟随,无需单独改)**:Mera 经 native 的 `run_qlib_backtest` 内部 `HBTopKStrategy` 自动读 `qualified_mask`;cn_tl 下该矩阵由 `etl_RN/compliance_tl.py` 从 `.bin` 的 `$is_st` 重建(语义同聚宽:上市≥365天 ∧ 过去365天无ST,落 `Dataset2006TL/qualified_mask.pkl`)。Mera 不必单独改合规,跟主线 shim 走。
- **仍从聚宽借的两个文件**:① `cons_csi1000.csv`(Mera universe = 中证1000成分,通联不产,沿用 `Dataset2006JQ/`,缺则 `FileNotFoundError`);② `industry_map.pkl`(仅 `USE_INDUSTRY_NEUTRALIZE=True` 时用,`_load_industry_map` 读不到只 warn 返回空 dict、跳过行业中性化、不报错)。要在通联下也有行业映射:用聚宽副本,或从 `.bin` 的 `industry`(申万一级)字段重建一份落到 `Dataset2006TL/`。
- **★ 训练/回测窗口收缩到 2014+**:~~实测 cn_tl `.bin` 从 2014 起~~ → **【A22 修正】`cn_tl` 日历实测 4969 交易日(2006–2026 全市场并集);但非两融小盘个票仍被主查询 INNER JOIN 截到 2014 起**(两融 INNER JOIN 截断,主线 71 §七十八),`market_features_v6.pkl` 同样只有 2014+。**Mera 在 cn_tl 上的训练/回测起点须 ≥ 2014**,否则前段空数据;要恢复 2006–2014 须把 `daily_stock_full_tl.py` 两处 INNER JOIN 改 LEFT JOIN(主线待办)。
- **验证**:① 重建后跑 Mera,确认不报 `market_features_v6.pkl` shape/格式错;② 同一配置 `QLIB_REGION` cn_jq vs cn_tl 对照 pred_score / 回测指标同量级(两边窗口都取 2014+ 才可比)。

*A21 增量 · Phase 71 联动 · 数据底座迁通联(承载 A20 全文,过时处见主线 71 划线修正)*
