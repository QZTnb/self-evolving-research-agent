# 自进化 Research Agent

会研究自己、每笔花费可追溯的 AI 研究员。

本项目是「Research Agent 系统设计」赛题的参赛作品。系统用多个 agent 角色自主完成「规划 → 收集证据 → 提假设 → 实验 → 验证 → 修正」的研究闭环：每次模型调用按四类 token 记账，每个结论都能追溯到账本和实验树；系统还会在练习题上研究并调整自己的设置，研究出的版本必须先通过一次性的验证准入，才能参加考试。

本仓库包含代码、配置和题包（含评分用数据与参考解），不含 agent 运行结果、日志、账本、内部文档或密钥。运行产物写入被 Git 忽略的 `runs/` 目录。

## 系统结构

| 部分 | 目录 | 做什么 |
|---|---|---|
| Agent 本体 | `research_agent/`、`library/`、`skill_layer/` | 同一个大模型分饰规划者、执行者、诊断者、总结者，按角色设定推理强度；纯代码调度器决定下一个实验、预算和何时停止；只追加的实验树；跨任务能力库（经验 → 技能）；按实验编号重新打分 |
| 题库与评分 | `tasks/`、`harness.py`、`harness_worker.py`、`tabular_harness.py`、`tabular_worker.py` | 两类研究任务：Python 代码提速、表格二分类。agent 写的代码在 macOS `sandbox-exec` 沙箱里运行（禁止联网、读不到隐藏数据、限制 CPU 和内存），另有审计钩子；holdout 每题只评一次 |
| 评估与展示 | `ledger.py`、`token_accounting.py`、`evaluation/`、`dashboard/` | 四类 token 账本与预算闸门；练习 → 验证 → 考试三阶段批次与一次性标记；四条对照线；自研究与准入；报告、出图、看板和独立审计 |

## 一道题怎么做

1. 题目自带的基线方案先打分，作为实验树的起点。
2. 规划者开局报价（预计花费、时间、分数），并列出几条假设。
3. 调度器按「预期提升 ÷ 预估花费」挑一条，每隔几个实验留一个探索位；执行者写出完整的 `solution.py`，在沙箱里运行并按 dev 数据打分。
4. 评审脚本只接受超过正常波动的提升；同一个实验失败 3 次才叫诊断者。
5. 预算、时间、分数到顶或止损规则触发时停止，从实验树写出经验，最佳方案在 holdout 上评一次分。

## 怎么评估系统本身

- **三层题目**：练习题用来积累能力库和研究自己，不限次数；验证题用于研究版的一次性准入；考试题只跑一轮。dev / holdout 是每道题内部的数据切分，与这三层无关。
- **四条对照线**：朴素版 `naive`、默认版空库 `default`、默认版加练过的能力库 `default_lib`、研究版加能力库 `tuned_lib`，每条线跑 3 遍。
- **准入规则**：研究版在验证题上每一遍分数都不低于「默认版加库」，并且平均花费更低，才能进入考试。
- **记账**：每次调用分输入、缓存输入、输出、推理四类 token 记账，价格表可以替换。示例价格表把四类都按 1 计（单位 TOKENS），只是 token 计量，不代表现金价格。预算闸门在调用前预留余量，并在预算的 90% 处停止扩展。
- **比较与离线回放**：逐题配对比较并给出置信区间；止损规则可以用已有运行日志离线回放来估算效果，不需要调用模型。

## 可选开关

`config/agent.json` 是默认配置；`config/agent_optimized.json` 打开了下面这些开关。每个开关缺省时都保持原有行为。

| 键 | 作用 |
|---|---|
| `stop_loss` | 按场景止损：表格题前 N 个实验都没进步就停，提速题连续 K 个实验没进步就停 |
| `ceiling_excludes_baseline` | 基线本身已达上限时不直接判到顶，交给规划者决定 |
| `planner_mode`、`hypotheses_per_plan` | 精简版规划者提示词，一次给出多条假设 |
| `failure_feedback` | 把失败类别、作弊标记和报错摘要告诉执行者和诊断者 |
| `cost_priors` | 用经验值代替规划者对单个实验的花费估计 |
| `experience_writer` | 设为 `script` 时由脚本从实验树提取经验，不调用模型 |
| `summarizer_reserve` | 设为 `worst_case` 时，总结者只按自身最坏花费预留预算 |
| `library_brief` | 规划者只看高可信度、截断后的能力库摘要 |
| `role_controls` | 按角色关闭思考、限制输出长度；使用前先用探测脚本确认网关支持 |

## 快速开始

需要 macOS（提供 `sandbox-exec`）和 Python 3.11；LightGBM 需要 Homebrew 的 `libomp`。经过核对的依赖版本见 `requirements-tabular.txt`。

```bash
python3.11 -m venv .venv
.venv/bin/python -m pip install -e '.[llm,tabular,speedup]'
.venv/bin/python -m unittest discover -s tests
.venv/bin/python -m unittest discover -s skill_layer/tests
.venv/bin/python -m unittest discover -s dashboard/tests
```

调用模型需要一个 OpenAI 兼容接口，只通过环境变量配置，不写入任何文件：

```bash
export OPENAI_API_KEY=...
export OPENAI_BASE_URL=https://<你的网关>/v1
```

常用命令：

```bash
# 用假模型走通一次对比实验，不需要密钥
.venv/bin/python -m scripts.run_ab_experiment --dry-run --configs baseline=config/agent.json optimized=config/agent_optimized.json --reps 1 --workers 1

# 真实对比实验与分析
.venv/bin/python -m scripts.run_ab_experiment --configs baseline=config/agent.json optimized=config/agent_optimized.json --reps 3 --workers 3
.venv/bin/python -m scripts.analyze_ab runs/ab-<编号>

# 用已有运行日志离线回放止损规则，不调用模型
.venv/bin/python -m scripts.replay_stoploss runs --output replay.json

# 探测网关是否支持推理强度和关闭思考（默认最多 15 次调用）
.venv/bin/python -m scripts.probe_reasoning_controls

# 只读看板
.venv/bin/python -m dashboard.serve --run-dir runs/<运行编号> --port 8765
```

正式流程见 `evaluation/README.md`。公开副本不附带校准证据；运行题库冻结流程前须自行生成所需证据。

## 题库

- `tasks/bank.json` 是已冻结的 bank-v1：提速题和表格二分类题各 12 道，每类按练习 5、验证 2、考试 5 分层；难度按朴素版实测花费定档；考试题里有止损题和泄漏诱饵题。
- `tasks/` 还包含准备中的 bank-v2 候选题；是否入库以 `tasks/bank.json` 为准。
- 每道题一个目录：`task.json` 是题面和预算，`workspace/` 是 agent 能读的材料，`hidden/` 是评分用的 dev / holdout 数据或参考解，agent 在沙箱里读不到。

## 数据与许可

- 表格题数据来自 UCI 机器学习库，已重新切分为训练 / dev / holdout（60/20/20），部分题做了特征选择或目标改写。逐题来源、许可与加工说明见各题目录下的 `SOURCE.md`。
- 提速题的原始代码和参考解由团队编写。

## 第三方依赖与 AI 工具

- 依赖：OpenAI Python SDK、NumPy、pandas、scikit-learn、LightGBM、AutoGluon Tabular（含 CatBoost、XGBoost）、xlrd，见 `pyproject.toml` 和 `requirements-tabular.txt`。主循环、调度、记账和评分均为自行实现，没有使用 agent 框架。
- 模型：研究实验使用 qwen3.5-27b，经 OpenAI 兼容接口调用，接口地址可配置。
- 开发过程中使用了 OpenAI Codex 和 Claude Code（Anthropic）辅助编码与写文档。

## 目录

| 位置 | 内容 |
|---|---|
| `research_agent/` | 主循环、角色、调度、实验树、模型接口、重新打分 |
| `library/`、`skill_layer/` | 能力库与技能层 |
| `harness*.py`、`tabular_*.py` | 评分器与沙箱 |
| `tasks/` | 题库与题包 |
| `ledger.py`、`token_accounting.py` | 账本与计量 |
| `evaluation/` | 批次、自研究、准入、比较、报告与出图 |
| `dashboard/` | 只读看板 |
| `scripts/` | 建题、校准、冻结、审计、对比实验、离线回放等工具 |
| `config/` | 配置、价格表与提示词 |
| `pilot/` | 最早的计时与经验库联调原型 |
| `tests/` | 自动化测试 |
