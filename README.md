# Depth State Translator

跨模型 KV 翻译实验归档：把 36 层的 Llama-3.2-3B 或 34 层的 Qwen3-4B 的逐层 KV
翻译到另一个模型，让接收方在**不加载发送方**的情况下读它的上下文。

本仓库是 `yezhe2021/new-plan-with-gudingkuangjia` 在 **2026-11-07** 之后的切片：
从 `depth_state_translator_fullsync_l3_to_q4_seed1234`（DST 主线成型）开始，
到 `dst_depth_recurrence_ablation_l3_to_q4_seed1234`（2026-11-16）为止的实验，
用 `git filter-repo` 连提交历史一起摘出。此前的 Full-Sync / Recolor /
Grouped-HeadMix 系列留在原仓库。

实验按 `EXPERIMENT_RESULTS_POLICY.md` 记账：一个研究实验一个顶层目录，
其阶段、控制组、臂作为子目录嵌套；每个目录的 `RESULTS.md` 由实验代码在
终评通过后自动生成，是本仓库唯一权威读数。`config.json` + `manifest.json`
固定输入、哈希与检查点，`status.json` 记录完成状态。

模型权重、检查点、KV cache 与 `prepared/train.jsonl` 均不入库；因此
**归档不等于可复现**——部分脚本含服务器绝对路径，重跑需要自己准备检查点。

## 主线：DST 是怎么长出来的

| 目录 | 一句话 |
|---|---|
| `recolor_kl_l3_to_q4_seed1234/depth_state_translator_fullsync_l3_to_q4_seed1234` | DST 第一版：逐层深度状态翻译 + Full-Sync，八任务宏平均 36.91，Recolor+C 基线 44.92 |
| `recolor_kl_l3_to_q4_seed1234/depth_retrieval_translator_fullsync_l3_to_q4_seed1234` | DRT：把共享的深度状态换成按需检索，接在 DST 之后 |
| `recolor_kl_l3_to_q4_seed1234/dst_stage1_rank64_then_static_m32_a4_seed1234` | 第二阶段：冻结 DST，加 Rank-64 残差 + 32-token 静态 memory，均值 51.56 |

三个都嵌在 `recolor_kl_l3_to_q4_seed1234/` 里，因为它们是那个父实验的阶段；
父目录其余子项（wrongdonor、layergroup、prefix sweep、早期 rnkv）属于迁移前的
工作，留在旧仓库。

## Memory 那条线：四种给答案层加位置的做法，全部打平

| 目录 | 一句话 |
|---|---|
| `dst_stage1_rank64_dynamic_memory_m32_a4_seed1234` | 动态 memory：50.98 对静态 50.59，对 Rank64 −0.59，八区间全跨零 |
| `dst_rank64_gated_dynamic_m32_a4_server_a_seed1234` | 36 个逐层门（初值 0）：53.32，+1.76 [+0.20,+3.32] 是这条线唯一不跨零的臂；但门值全程 \|g\| ≤ 0.039，记忆最多写入 4% |
| `dst_rank64_gated_joint_m32_a4_seed1234` | 门控臂再训 384 步（fresh cohort）：52.15 → 52.25，1024 行救 13 坏 12 |

结论：真正有用的是**深度递归携带的状态**，不是外加的 memory。见下。

## 因果证据：什么真的在起作用

| 目录 | 一句话 |
|---|---|
| `dst_depth_recurrence_ablation_l3_to_q4_seed1234` | 关掉 `carry_state` 重训：stage1 43.66→33.49（**+10.18** [+8.12,+12.19]）、stage2 56.84→53.58（**+3.26** [+1.33,+5.18]）；GSM8K 从 53.91 掉到 15.62。全项目最大的受控效应 |
| `dst_rank64_oracle_block_scan_seed1234` | 单个 native KV 块：dev 选出的固定块值 ARC +9.38 / Hotpot +4.69，逐题 oracle 有 +25 / +23.44 —— 上界很大，能转移的只有四分之一 |
| `dst_memory_guided_repair_seed1234` | 2048 条 oracle 标签、768 步学出来的修复：测试集 **0 行改变** |
| `dst_rank64_dynamic_joint_m32_a4_seed1234` | 384 步联合微调：51.56 进、51.56 出，触到 42 行、救 21 坏 21 |

## 跨模型：换方向、换底座

| 目录 | 一句话 |
|---|---|
| `gemma_dst_rank64_stage12_server_a_seed1234` | Gemma-3-4B → Qwen3-4B 正向：C 35.94 / S1 28.52 / **rank64 44.53** / native 61.13 |
| `qwen_dst_rank64_to_gemma_server_a_seed1234` | 反向 Qwen→Gemma（36→34 层，跳过第 9、26 层）：native 55.08 / S1 36.91 / **r64 50.59**，保留率 91.8% |
| `heterofold_qwen_to_gemma_paper_seed20260825` | HeteroFold R16 校准：41.80，显著项只有 hotpotqa/2wiki +14.06 —— 正是校准语料 OpenR1-HotpotQA 的分布 |
| `recolor_c_dst_data_l3_to_q4_seed1234` | Recolor 换成 DST 的数据重训：48.83 / +C 49.02。DST 那 6.64pp 优势里 **4.10pp 是数据和配方**，不是翻译结构 |
| `rnkv_gemma_recolor_c_keep_both_l19_30_3task_seed1234` | Gemma Recolor+C 三任务 keep-both，memory 可读 19–30 层 |

## 长上下文与代价

| 目录 | 一句话 |
|---|---|
| `dst_long_context_ruler_helmet_seed1234` | RULER 1k 起就是 **0/200**（三臂），错法是形状对数值错——丢的是精确 token 身份，不是衰减 |
| `dst_rank64_gated_dynamic_longbench_server_a_seed1234` | LongBench：开放问答保留 60%，一旦触发 16k 裁剪只剩 1/3。**快照，无 qasper** |
| `dst_e4_quality_latency_seed1234` | 首 token 延迟（无准确率）：DST − Recolor **恒定 0.30–0.32 ms/token**，永不摊薄；8k 时两 cache 臂比 native 快 2.5–3.2× |

## 怎么读

1. 想看某个结论 → 直接进目录读 `RESULTS.md`（每个都带逐任务表、配对区间、口径说明）。
2. 想知道实验怎么做的 → `config.json`（固定配置）+ `manifest.json`（输入与检查点哈希）+ `experiment.py`。
3. 想知道为什么这么记账 → `EXPERIMENT_RESULTS_POLICY.md` 与 `EXPERIMENT_PROTOCOL_DEFAULTS.md`。
4. `EXPERIMENT_RESULTS.md` 继承自原仓库的机器目录（`tools/build_catalog.py` 生成），
   主要记录**迁移之前**的实验；它不是本仓库这些实验的索引，那些以各自 `RESULTS.md` 为准。
   同理 `NEW_COMPUTER_SETUP.md` / `PROJECT_HANDOFF.md` / `AGENTS.md` 写于迁移前，
   其中的服务器路径仍然有效，但提到的实验目录可能指向旧仓库。

## 迁移记录

- 来源：`yezhe2021/new-plan-with-gudingkuangjia`，`main` @ `6773a330e`
- 方法：`git filter-repo --path …`，保留提交历史，空提交被剪除
- 结果：85 提交 → **27 提交**，128,732 文件 → **44,817 文件**，340 MB → 91 MB
- 原仓库未改动，仍在原处
