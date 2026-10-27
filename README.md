# SWE-BenchFlow

SWE-BenchFlow 是用于软件工程任务数据构建与代码修复评测的工具集。项目包含 GitHub 仓库和 Issue/PR 数据采集、任务筛选、RepoLaunch 容器环境构建，以及基于测试结果的补丁评测。

当前项目按独立本地文件夹使用，不需要先初始化或克隆本项目的 Git 仓库。下文命令默认在项目根目录执行，另有说明的除外。

## 项目结构

```text
.
├── assets/                         # 图片与数据集说明
├── curation/                       # 数据采集与筛选
│   ├── crawl_repo.py               # 按语言、Star 数等条件采集仓库
│   ├── filter_repo.py              # 仓库筛选
│   ├── swe_task_crawling/          # PR、Issue 及任务数据处理
│   ├── llm_filter/                 # 使用模型筛选任务、区分操作系统
│   ├── push_dataset/               # 数据集发布脚本
│   └── run.sh                     # 批量采集流程参考脚本
├── evaluation/
│   ├── evaluation.py              # 标准补丁与预测补丁评测入口
│   └── validation.py              # 任务验证相关逻辑
├── launch/                        # 随项目保留的 RepoLaunch 源码
│   ├── launch/                    # 容器运行时、环境构建及日志解析
│   └── docs/                      # RepoLaunch 使用说明
└── pyproject.toml                 # Python 依赖与打包配置
```

## 环境准备

完整运行环境使用 **Python 3.12 或更高版本**：根目录配置要求 Python 3.10+，但内置 RepoLaunch 要求 Python 3.12+。执行评测还需要可用的 Docker 服务以及与任务操作系统匹配的容器环境。仓库采集与构建流程可能需要 Git、网络访问和相应 API 凭据。

### 创建虚拟环境

Windows PowerShell：

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
$env:PYTHONUTF8 = "1"
$env:PYTHONIOENCODING = "utf-8"
```

Linux / WSL：

```bash
python3.12 -m venv .venv
source .venv/bin/activate
```

如果已有 `.venv`，请先确认其中的 Python 版本和依赖是否适用于当前机器。

### 安装依赖

当前文件夹缺少根目录打包配置引用的 `LICENSE` 和 `launch/pyproject.toml` 引用的 `launch/README.md`，因此这里直接读取两个配置的依赖列表进行安装，不依赖本项目的可编辑安装。

以下命令可在上述已激活的 Python 3.12 环境中执行：

```text
python -m pip install --upgrade pip
python -c "import pathlib, subprocess, sys, tomllib; paths = ['pyproject.toml', 'launch/pyproject.toml']; deps = [dep for path in paths for dep in tomllib.loads(pathlib.Path(path).read_text(encoding='utf-8'))['project']['dependencies']]; subprocess.check_call([sys.executable, '-m', 'pip', 'install', *dict.fromkeys(deps), 'aiohttp'])"
python -m evaluation.evaluation --help
docker info
```

`aiohttp` 用于仓库筛选。根目录配置中的 Python 分发包名仍是 `swebench`，内置环境构建包名仍是 `launch`；`SWE-BenchFlow` 是本文使用的项目名称。

评测模块根据当前工作目录定位 `launch/`，请从项目根目录运行 `python -m evaluation.evaluation`。

## 快速开始：评测本地任务

### 1. 准备任务数据

将任务保存为 UTF-8 编码的 `data/tasks.jsonl`，每行一个 JSON 对象。评测入口直接使用的主要字段如下：

| 字段 | 用途 |
| --- | --- |
| `instance_id` | 任务唯一标识，与预测补丁文件中的键一致 |
| `test_patch` | 评测时应用的测试补丁 |
| `FAIL_TO_PASS` | 修复后必须通过的测试名称列表 |
| `PASS_TO_PASS` | 用于检查已有测试回归的测试名称列表 |
| `patch` | 标准答案补丁，使用 `--patch_dir gold` 时必需 |
| `docker_image` | 任务镜像；省略时按任务 ID 和平台生成默认镜像名 |
| `rebuild_cmds` | 修改源码后的重新构建命令列表 |
| `test_cmds` | 测试命令列表 |
| `print_cmds` | 读取测试日志的命令列表 |
| `log_parser` / `parser` | 测试日志解析器；优先使用 `log_parser`，`pytest` 使用内置解析逻辑 |
| `created_at` | 任务创建时间，月份筛选使用其中的 `YYYY-MM` 前缀 |

命令字段应为字符串列表，测试集合字段应为测试名称列表。任务数据需要配套可执行的镜像、测试命令和解析器，只有任务描述无法直接评测。

未提供 `docker_image` 时，镜像名称规则为：

```text
Linux:   starryzhang/sweb.eval.x86_64.<转换后的 instance_id>
Windows: starryzhang/sweb.eval.win.<转换后的 instance_id>
```

其中 `instance_id` 中的 `__` 替换为 `_1776_`，再转为小写。使用自建镜像时，建议在数据中显式设置 `docker_image`。

### 2. 验证标准答案补丁

先使用任务中的 `patch` 检查本机环境下的任务是否可以通过：

```text
python -m evaluation.evaluation --dataset data/tasks.jsonl --patch_dir gold --platform linux --workers 1 --output_dir logs/gold --overwrite 0
```

成功任务会额外写入 `logs/gold/gold_patch_evaluated_instances.jsonl`，可作为后续评测的数据子集。Windows 容器任务将 `--platform` 改为 `windows`，并准备匹配的 Windows 容器环境。

### 3. 准备并评测预测补丁

预测文件是一个 JSON 对象，**不是 JSONL**。示例 `predictions.json`：

```json
{
  "owner__repo-123": {
    "model_patch": "diff --git a/example.py b/example.py\n--- a/example.py\n+++ b/example.py\n@@ -1 +1 @@\n-old_value = 1\n+old_value = 2\n"
  }
}
```

实际使用时替换为对应任务的真实 Git diff，然后运行：

```text
python -m evaluation.evaluation --dataset data/tasks.jsonl --patch_dir predictions.json --platform linux --workers 1 --output_dir logs/predictions --overwrite 0
```

虽然参数名为 `--patch_dir`，这里传入的是 **JSON 文件路径**。只有在数据集和预测文件中都存在的任务才会进入预测评测；空白补丁单独计数，不会启动评测容器。

## 评测参数

| 参数 | 说明 |
| --- | --- |
| `--dataset` | 必填；本地 `.jsonl` 文件路径，或 Hugging Face 数据集标识 |
| `--patch_dir` | 必填；预测 JSON 文件路径，或小写 `gold` |
| `--platform` | 必填；`linux` 或 `windows` |
| `--workers` | 必填；并发线程数，初次运行建议设为 `1` |
| `--output_dir` | 必填；结果输出目录 |
| `--overwrite` | 必填；`0` 复用已有有效报告，`1` 重新执行 |
| `--split` | Hugging Face 数据集分区；读取本地 JSONL 时不需要 |
| `--instance_ids` | 仅评测指定任务，可传入多个以空格分隔的 ID |
| `--start-month` | 起始月份，格式 `YYYY-MM`，包含该月 |
| `--end-month` | 结束月份，格式 `YYYY-MM`，包含该月 |
| `--collect-only` | 仅汇总已有报告，不执行未完成任务 |

按月份筛选的示例：

```text
python -m evaluation.evaluation --dataset data/tasks.jsonl --patch_dir predictions.json --platform linux --workers 1 --output_dir logs/predictions --overwrite 0 --start-month 2025-06 --end-month 2025-07
```

未指定的月份边界取数据集中可解析的最早或最晚月份。当前实现不会因某条任务缺少可解析的 `created_at` 而排除该任务；如果所有任务都没有可解析时间，却指定了月份筛选，则会报错。

只汇总已有结果时，在相同数据、补丁及输出目录的命令后追加 `--collect-only`。该模式仍会读取数据和预测文件，并将缺少有效报告的非空补丁任务标为未完成。

## 输出结果

```text
logs/predictions/
├── results.json                   # 总体计数与任务 ID 列表
└── <instance_id>/
    ├── post_patch_log.txt          # 应用补丁后的测试日志
    ├── status.json                 # 测试名称到 pass/fail/skip 的映射
    └── report.json                 # 单任务判定与测试集合统计
```

`results.json` 包含 `submitted`、`success`、`failure`、`empty_patch`、`error`、`incomplete` 及对应 ID 列表。执行异常的任务可能只留下部分输出文件。

当前判定逻辑要求 `FAIL_TO_PASS` 全部通过，且 `FAIL_TO_PASS`、`PASS_TO_PASS` 中均没有记录为失败的测试。`PASS_TO_PASS` 中未出现在解析结果中的测试目前不会单独导致失败，因此分析评测结果时也应检查日志与解析结果是否完整。

修改数据、补丁或镜像后需要重新评测时，请使用新的输出目录或设置 `--overwrite 1`，避免复用旧报告。

## 数据采集与环境构建

数据构建流程主要分为以下步骤：

1. 使用 `curation/crawl_repo.py` 采集候选仓库，再用 `filter_repo.py` 筛选。
2. 使用 `curation/swe_task_crawling/` 中的脚本采集 PR、关联 Issue 并合并任务。
3. 使用 `curation/llm_filter/verify.py` 和 `split_os.py` 进行模型筛选和操作系统分类。
4. 使用 `launch/` 中的 RepoLaunch 为任务准备容器环境、构建和测试命令及日志解析器。
5. 使用评测入口验证标准答案补丁，再评测模型预测补丁。

采集脚本的参数及调用顺序可参考 `curation/run.sh`。该脚本包含特定机器的绝对路径、固定截止日期、模型名称和末尾孤立的 `~` 行，使用前需要按本机环境调整，不能直接视为开箱即用的入口。脚本应在 `curation/` 目录下通过 Bash 执行。

采集流程需要配置 GitHub Token 文件；模型筛选与 RepoLaunch 的模型调用需要配置所用服务的凭据。`curation/push_dataset/` 中的脚本会向远端发布数据，使用前应检查目标数据集和账号配置。

RepoLaunch 的详细配置见 [环境构建说明](launch/docs/Development.md)、[Windows 环境说明](launch/docs/Development-Windows.md) 和 [跨提交复用说明](launch/docs/Development-agent-memory.md)。这些子目录文档可能仍保留上游项目名称和安装方式，应结合当前文件夹内容使用。

## 离线使用说明

项目可以作为普通本地文件夹保存、复制和编辑，但完整流程并非默认完全离线：

- 仓库与 PR 采集需要访问 GitHub；远程数据集加载、镜像拉取和模型 API 调用也需要对应网络服务。
- 本地评测可以使用 JSONL 数据和预测 JSON 文件，但必须提前准备 Python 依赖、Docker 镜像及测试需要的其他资源。
- 即使镜像已在本地，任务内部的构建或测试命令仍可能访问网络，应按任务检查。
- 本项目本身不需要 Git 元数据，但被构建、打补丁或评测的目标仓库仍可能依赖 Git。

## 常见问题

| 现象 | 检查方式 |
| --- | --- |
| 无法导入 `launch.core` | 确认从项目根目录运行，且 `launch/launch/` 源码完整 |
| Python 版本或依赖报错 | 使用 Python 3.12+，并安装两个 `pyproject.toml` 声明的依赖 |
| 无法连接 Docker | 运行 `docker info`，检查服务、权限和容器平台 |
| 镜像不存在或拉取失败 | 检查任务的 `docker_image`、访问权限及本地镜像准备情况 |
| 评测没有执行任何任务 | 检查任务 ID 与预测文件键是否一致，以及 ID、月份筛选条件 |
| 修改补丁后结果未变化 | 更换输出目录或设置 `--overwrite 1` |
| 日志解析为空或结果异常 | 核对 `test_cmds`、`print_cmds` 与日志解析器是否匹配 |
| PowerShell 输出乱码 | 在运行前设置 `PYTHONUTF8=1` 和 `PYTHONIOENCODING=utf-8` |
