# MRG_RL：医学报告生成优化实验

[English](README.md) | [中文](README.zh-CN.md)

本仓库整理了我们在 IU X-Ray 数据集上围绕 Medical Report Generation (MRG) 做的一系列实验，目标是同时提升报告文本指标和临床准确性。

最终 MRG model 的目标指标是：

- `Bleu_4 >= 0.20`
- `ROUGE_L >= 0.40`
- `CheXbert micro F1 >= 0.40`
- `RadGraph rg_er >= 0.39`

需要说明的是，本仓库不是原始 R2GenGPT paper 的复现或 fork。代码中保留了少量历史兼容命名，例如 `models/R2GenGPT.py`，只是因为远端实验环境和训练脚本当时依赖这些接口。仓库的主体工作是我们自己的 MRG_RL 实验栈，包括 Qwen/DINO-based SFT、DPO 诊断、retrieval candidate 构建、oracle analysis、clinical metric evaluation、candidate reranking、template baseline，以及 candidate-copy / number-selection 相关实验。

## 仓库内容

核心实验工作包括：

- IU X-Ray 上的 SFT / DPO / RL-style 试错记录和 failure analysis。
- 面向 `Bleu_4` / `ROUGE_L` 的 template baseline 与 decode sweep。
- 基于 image-nearest reports 的 top-k retrieval candidate 构建。
- top-k candidate pool 的 reference-aware oracle 和 clinical-aware oracle 分析。
- `CheXbert` 与 `RadGraph` 评估工具。
- 多种 candidate report reranking 尝试：
  - DINO / scalar reranker
  - image-label-guided selector
  - BERT-text cross-reranker
  - model log-prob candidate selector
  - candidate-number selection SFT
- 可用于复盘的 result artifacts、prediction JSON、experiment summaries。

大型 checkpoint 没有纳入仓库。详见 `CHECKPOINTS_NOT_INCLUDED.md`。

## 模型栈

本实验主要使用的 model stack：

- Vision encoder：`facebook/dinov2-base` / 本地 `resources/models/facebook_dinov2-base`。
- LLM backbone：`Qwen3-8B` / 本地 `resources/models/Qwen3-8B`。
- Trainable bridge：Perceiver-style visual resampler，加上 LLM 侧 LoRA adapters。
- Text reranking encoder：`bert-base-uncased`，用于 text cross-reranker 中的 candidate report embeddings。
- Clinical evaluation models：`CheXbert` 用于 label F1，`RadGraph` / ModernBERT-based RadGraph resources 用于 entity-relation F1。

## 主要发现

核心结论是：目标分数在 retrieved-candidate space 中是可以达到的，但在当前 small-data setting 下，deployable selector / generator 还无法稳定达到。换句话说，瓶颈已经从“能不能找到好报告”转移到“模型能不能在没有 reference 的情况下可靠地选择或使用这条报告”。

| Setting | Test Bleu_4 | Test ROUGE_L | CheXbert micro F1 | RadGraph rg_er |
| --- | ---: | ---: | ---: | ---: |
| Top-50 clinical oracle | 0.3258 | 0.5287 | 0.7849 | 0.4798 |
| Top-20 clinical copy oracle | 0.2556 | 0.4671 | 0.7111 | not rerun |
| Fixed template baseline | 0.1824 | 0.4001 | 0.0000 | 0.3965 |

我们尝试过的实验和结论如下：

| Experiment line | Best / representative result | Finding |
| --- | --- | --- |
| Original SFT / decode sweeps | 常规 SFT 最好大约 `B4≈0.13-0.14`, `Rouge-L≈0.37` | 标准 generation 没有达到 language-overlap 目标，仅靠 decode tuning 不够。 |
| DPO variants | DPO 没有提升目标指标，并被诊断为当前 setup 下不稳定 / reward 不对齐 | Pairwise preference training 在没有更强 reward / selection signal 时收益很差。 |
| Fixed template baseline | `B4=0.1824`, `Rouge-L=0.4001`, `CheXbert F1=0.0000`, `RadGraph rg_er=0.3965` | 通用 normal-report template 是很强的 B4/Rouge baseline，但无法处理 abnormal clinical correctness。 |
| Template refinement / sentence fusion | 可以维持 `Rouge-L≈0.40`，但不能把 B4 提到 `0.20`，也不能有效提升 CheXbert | Template 方法利用了 IU X-Ray 的 dataset bias，但解决不了临床异常准确性。 |
| Top-k retrieval oracle | Top-50 oracle: `B4=0.3645`, `Rouge-L=0.5624`；clinical oracle 同时通过 CheXbert / RadGraph | retrieval candidates 中确实包含能达到所有目标的信息，问题在于如何无 reference 选择。 |
| Clinical scalar / image-label selector | Test candidate selector 达到 `CheXbert F1=0.4017`，但只有 `B4=0.1284`, `Rouge-L=0.3473` | clinical signal 能拉标签准确率，但当前 image features 选出的报告 ngram overlap 差。 |
| Text cross-reranker | Best test: `B4=0.1350`, `Rouge-L=0.3528`, `CheXbert F1=0.3551` | frozen text embeddings + DINO image embeddings 不足以做可靠 candidate selection。 |
| Template-candidate hybrid | 只要用 selector candidate 替换一部分 template，B4/Rouge 就下降 | 弱 selector 会破坏 strong template baseline，而不是改进它。 |
| Copy-oracle SFT with top-20 candidates in prompt | Free generation: `B4=0.1111`, `Rouge-L=0.3152`, `CheXbert F1=0.1905`, `RadGraph rg_er=0.2689` | 即使用 oracle candidate 训练，模型也没有可靠学会利用 long retrieved context 来 copy / fuse。 |
| Model log-prob candidate selection | `B4=0.0632`, `Rouge-L=0.2735`; average selected rank `9.69` | sequence likelihood 主要受 language prior / style 影响，而不是 image-conditioned correctness。 |
| Candidate-number SFT | 最好 checkpoint: `B4=0.1068`, `Rouge-L=0.3256`; 输出偏向高编号候选 | 把任务降成 20-way candidate-number selection 仍没有学到稳定的 image-conditioned ranking。 |

总体判断：继续在 IU X-Ray 小数据上做 prompt-level tuning 或轻量 SFT/DPO，大概率不是最有效方向。下一步模型改进应当优先替换弱 selection signal，引入 radiology image-text aligned retriever / reranker，最好用 MIMIC-CXR 这类更大规模 image-report 数据进行训练或适配；等 selection quality 足够强之后，再做 candidate copy / fusion 或 generation。

## 仓库结构

```text
configs/                 训练和推理配置
dataset/                 IU X-Ray parsing 与 retrieved-context loading
models/                  MRG model wrapper 与 vision resampler
tools/                   实验构建、评估、reranker 和工具脚本
scripts/                 可复现实验 shell scripts
data/iu_xray/            实验用 lightweight annotations
save/iu_xray/            关键 summaries、metrics、predictions 和 logs
save/iu_xray/model_registry/
                         人类可读的 experiment registry 和 trial notes
```

## 关于历史命名

仓库中仍有一些路径包含 `R2GenGPT` 或 `qwen_dino_sft_*`，这是因为它们来自远端实验工作区的历史文件名。它们应被理解为 MRG_RL 实验代码中的兼容命名，而不是说明本仓库是 upstream R2GenGPT 项目。

本实验中的 model stack 已经在工作中发生了改变，包括 Qwen-based language modeling、DINO visual features、retrieval-context prompts、自定义 SFT/DPO utilities、candidate reranking tools，以及 clinical metric evaluation。

## 当前状态

本仓库是一个 experiment archive 和 reproducibility package，记录了成功的 upper-bound analyses，也记录了未达到目标的 deployable approaches。最有价值的后续方向是：

1. 训练或引入 radiology image-text aligned retrieval model。
2. 用 aligned model rerank top-k candidate reports。
3. 只有当 selection quality 足够强时，再做 candidate copy / fusion。
4. 使用 `Bleu_4`, `ROUGE_L`, `CheXbert micro F1`, `RadGraph rg_er` 重新评估。
