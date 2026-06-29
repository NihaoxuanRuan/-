# MERA v6 MoE 四组诊断脚本

四个脚本都是从当前 `qlib_pipeline_mera_v6.py` 完整复制并修改得到的，可直接独立运行。它们继续共用原来的 `mera_v6_pretrain` checkpoint，不会重新预训练编码器。

## 默认运行方式

```bash
python test_01_expert_count_and_aux.py
```

默认会：

1. 依次运行该脚本中定义的全部 variant；
2. 只测试最新一个 expanding window，保留原始 window 编号，因此能正确加载对应预训练 checkpoint；
3. 为每个 variant 使用独立 `EXPERIMENT_TAG` 和下游 checkpoint；
4. 自动输出比较表、日志和专家诊断文件。

## 常用参数

列出脚本内所有实验：

```bash
python test_03_topk_and_expert_layout.py --list
```

只运行一个实验：

```bash
python test_03_topk_and_expert_layout.py --variant 2x8_16exp_top2
```

运行指定 expanding window：

```bash
python test_03_topk_and_expert_layout.py --variant 2x8_16exp_top2 --window 7
```

对筛选后的配置跑完整 expanding windows：

```bash
python test_03_topk_and_expert_layout.py --variant 2x8_16exp_top2 --window all
```

快速调试时临时缩短训练：

```bash
python test_03_topk_and_expert_layout.py --max-epochs 10 --max-steps 50 --patience 4 --no-backtest
```

正式比较时不要使用上述缩短参数。

## 四个脚本包含的实验

### 1. `test_01_expert_count_and_aux.py`

验证“8 个专家是否过多”和“负载均衡损失是否强迫无意义拆分”：

- 8 experts, aux=0.01
- 8 experts, aux=0.001
- 8 experts, aux=0
- 4 experts, micro top-1, aux=0.001
- 4 experts, micro top-2, aux=0.001

### 2. `test_02_neighbor_count.py`

验证邻居过多是否平滑 gate 输入：

- K=50
- K=30
- K=20
- K=10

额外记录邻居注意力的有效邻居数：

`Neff = 1 / sum(attention^2)`

### 3. `test_03_topk_and_expert_layout.py`

验证 Top-K 与专家布局：

- 2×4=8 experts, micro top-2（基准）
- 2×4=8 experts, micro top-1
- 2×8=16 experts, micro top-2
- 2×8=16 experts, micro top-3
- 4×4=16 experts, macro top-1 + micro top-1

### 4. `test_04_topk_renormalization.py`

做 2×2 实验：

- micro top-2，不重归一化
- micro top-2，重归一化
- micro top-1，不重归一化
- micro top-1，重归一化

重归一化作用于 Macro Gate 和 Micro Gate 的 Top-K 混合权重；负载均衡辅助损失仍使用原始、未重归一化的 gate 权重，因此不会把“混合权重变化”和“aux 尺度变化”混在一起。

## 自动输出

每个测试套件会生成：

```text
Output/Mera_Output/test_summaries/<suite_name>/comparison_latest.csv
Output/Mera_Output/test_summaries/<suite_name>/logs/*.log
```

每个 variant 的诊断文件位于：

```text
Output/Mera_Output/mera_v6_diagnostics/<EXPERIMENT_TAG>/ret2d/
```

包括：

- `window_XX_summary.json`
- `window_XX_expert_usage.csv`
- `window_XX_expert_overlap.csv`
- `window_XX_expert_output_corr.csv`

重点关注：

- `best_val_ic`
- `diag_expert_output_corr_abs_mean`：越高表示专家函数越相似
- `diag_mean_offdiag_overlap`：越高表示专家接收样本越重叠
- `diag_route_entropy_mean`：越高表示单样本路由越犹豫
- `diag_top1_top2_margin_mean`：越高表示专家选择越明确
- `diag_dead_experts_lt_1pct`：近似死专家数量
- `diag_neighbor_effective_count_mean`：实际发挥作用的邻居数量
- `diag_gate_input_effective_rank`：gate 输入的有效维度
- `diag_gate_sum_mean`：Top-K 权重和；归一化后应接近 1

先用最新 window 做筛选，再对最有希望的 2 至 3 个配置使用 `--window all` 做完整样本外验证。
