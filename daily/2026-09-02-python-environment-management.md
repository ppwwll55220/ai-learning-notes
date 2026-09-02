# 阶段 0 第 06 课：Python 环境管理 — 学习笔记

**日期**：2026-09-02  
**核心工具**：uv、pyproject.toml、lockfile、虚拟环境  
**关键词**：依赖隔离、可复现构建、PEP 668、externally-managed-environment

---

## 本课核心目标

- 理解为什么 AI 项目需要虚拟环境（依赖地狱）。
- 掌握用 uv 创建、激活、管理虚拟环境的标准流程。
- 理解 pyproject.toml 和 lockfile 的作用和区别。
- 能够诊断常见的环境问题（全局安装、CUDA 不匹配、混用工具等）。
- 知道课程推荐的按 Phase 划分环境策略。

---

## 核心概念：虚拟环境到底隔离了什么？

虚拟环境不是对整个 Linux 系统生效的，它只对当前激活的终端会话生效。

### 两个容易混淆的概念

| 概念 | 说明 | 作用范围 |
| :--- | :--- | :--- |
| 系统全局 Python | Ubuntu 自带的 /usr/bin/python3 | 所有用户、所有终端 |
| 虚拟环境（如 ~/.venv） | 手动创建的独立 Python 副本 | 仅限当前激活的终端 |

### 为什么虚拟环境能解决依赖地狱？

AI 项目常常依赖特定版本的 PyTorch、NumPy、Transformers 等。不同项目需要的版本可能冲突（比如项目 A 要 PyTorch 2.4，项目 B 要 PyTorch 2.1）。虚拟环境为每个项目提供独立的工具箱，互不干扰。

---

## 工具选择：为什么全用 uv？

| 操作 | 推荐命令 | 避免 |
| :--- | :--- | :--- |
| 创建虚拟环境 | `uv venv` | `python -m venv` |
| 安装包 | `uv pip install` | `pip install` |
| 添加依赖 | `uv add numpy`（自动更新 pyproject.toml 和 lockfile） | 手动编辑 pyproject.toml |
| 锁定版本 | `uv lock` | `pip freeze > requirements.txt` |
| 同步环境 | `uv sync` | `pip install -r requirements.txt` |

**核心原则**：全用 uv，全用虚拟环境，永不全局安装。

---

## 项目声明文件：pyproject.toml

每个 Python 项目都应该有一个 pyproject.toml。它用一个文件取代 setup.py、setup.cfg 和 requirements.txt。

### 基础结构

```toml
[project]
name = "ai-engineering-from-scratch"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "numpy>=1.26",
    "matplotlib>=3.8",
    "jupyter>=1.0",
    "scikit-learn>=1.4",
]

[project.optional-dependencies]
torch = ["torch>=2.3", "torchvision>=0.18"]
llm = ["anthropic>=0.39", "openai>=1.50"]
安装指定依赖组
bash
uv pip install -e ".[torch]"        # 基础 + PyTorch
uv pip install -e ".[llm]"          # 基础 + LLM SDK
uv pip install -e ".[torch,llm]"    # 全部
-e 表示可编辑安装，让项目本身作为一个包被识别。

Lockfile（锁文件）：保证可复现性
pyproject.toml 写的是宽松的版本范围（如 numpy>=1.26）。Lockfile（如 uv.lock）则把每个包（包括传递依赖）固定到精确版本和哈希值。

提交到 git：pyproject.toml 和 uv.lock 都应提交。

不提交到 git：.venv/（虚拟环境文件夹）不应提交，因为它巨大且跨机器不可复用。

练习1：运行 env_setup.sh
目标：自动化创建课程基础环境，安装核心包（numpy、matplotlib、jupyter、scikit-learn、pandas）。

操作：

bash
cd ~/ai-engineering-from-scratch/phases/00-setup-and-tooling/06-python-environments/code/
bash env_setup.sh
原理：脚本检查 uv 是否存在、Python 版本是否 >= 3.11，然后在课程仓库根目录创建 .venv，安装核心包，最后验证导入是否成功。

遇到问题：

运行 which python 无输出，因为 Ubuntu 默认只有 python3 命令，没有 python 软链接。

解决方案：不影响虚拟环境使用，因为激活后 python 指向虚拟环境内的 Python。

练习2：创建第二个虚拟环境，验证隔离
目标：证明两个虚拟环境互不干扰。

操作流程：

bash
cd ~
mkdir test_venv2
cd test_venv2
uv venv
source .venv/bin/activate
uv pip install numpy==1.26.0
python -c "import numpy; print(numpy.__version__)"
deactivate
cd ~/ai-engineering-from-scratch
source .venv/bin/activate
python -c "import numpy; print(numpy.__version__)"
遇到问题：

尝试安装 numpy==1.24.0 时报错：ModuleNotFoundError: No module named 'distutils'。

原因：distutils 在 Python 3.12 中被移除，numpy 1.24.0 依赖 distutils 构建。

解决方案：安装 numpy==1.26.0（第一个正式支持 Python 3.12 的版本）。

教训：Python 版本升级会导致旧版依赖无法安装，这是 AI 工程中的常见场景。

练习3：编写 pyproject.toml（原理与模板）
目标：学会声明依赖分组。

原理：pyproject.toml 是现代 Python 项目的标准配置文件。uv add 会自动更新它和 lockfile。

标准模板：

toml
[project]
name = "my_project"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = []

[project.optional-dependencies]
torch = ["torch>=2.3", "torchvision>=0.18"]
llm = ["anthropic>=0.39", "openai>=1.50"]
安装命令：

bash
uv pip install -e ".[torch]"
练习4：全局安装实验（原理）
目标：体验全局安装的危害并学会清理。

遇到情况：

执行 python3 -m pip install colorama 时报错：error: externally-managed-environment。

原因：Ubuntu 24.04 默认启用 PEP 668 保护，阻止全局安装，防止破坏系统 Python。

解决方案：使用 --user 模拟用户级全局安装：

bash
python3 -m pip install --user colorama
python3 -m pip show colorama | grep Location
python3 -m pip uninstall colorama -y
教训：现代 Python 已经强制要求使用虚拟环境，全局安装被系统主动阻止。

常见错误与解决方案汇总
错误	原因	解决方案
ModuleNotFoundError: No module named 'distutils'	Python 3.12 移除了 distutils	安装支持 Python 3.12 的版本（如 numpy>=1.26.0）
externally-managed-environment	PEP 668 保护，禁止全局安装	使用虚拟环境，或 --user 安装到用户目录
command not found: python	Ubuntu 只有 python3 命令	在虚拟环境中使用 python，或安装 python-is-python3
which python 无输出	系统没有 python 软链接	不影响虚拟环境使用
课程推荐的按 Phase 划分环境策略
不同学习阶段依赖可能冲突，建议为每个 Phase 创建独立环境：

text
ai-engineering-from-scratch/
├── .venv/                    # 基础环境（phases 0-3）
├── phases/
│   ├── 04-neural-networks/
│   │   └── .venv/            # PyTorch 环境
│   └── 11-llm-apis/
│       └── .venv/            # API SDK 环境（无需 torch）
每日进入工作环境的流程
bash
cd ~/ai-engineering-from-scratch
source .venv/bin/activate
如果设置了别名，可以简化为一键：

bash
echo 'alias ai="cd ~/ai-engineering-from-scratch && source ~/.venv/bin/activate"' >> ~/.bashrc
source ~/.bashrc
本课交付物
一个可复现的课程基础 Python 环境（通过 env_setup.sh 创建）。

pyproject.toml + uv.lock 作为项目依赖的声明和锁定文件。

理解永远在虚拟环境中工作的工程原则，不再依赖全局 Python 状态。

退出虚拟环境的命令
bash
deactivate
本课笔记对应的 Git 提交命令
bash
cd ~/ai-learning-notes
git add daily/python-environment-management.md
git commit -m "docs: 添加 Python 环境管理课程笔记"
git push
本笔记基于阶段 0 第 06 课：Python 环境管理的学习内容整理，包含学习过程中的提问、错误排查和核心知识点总结。
