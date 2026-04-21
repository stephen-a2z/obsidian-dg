---
{"dg-publish":true,"permalink":"/01-技术分享/uv：写给 Python 开发者的下一代包管理器指南/","tags":["python","uv","包管理","入门教程","工具链"],"noteIcon":"","created":"2026-04-21T17:47:08.501+08:00","updated":"2026-04-21T17:47:08.502+08:00"}
---


# uv：写给 Python 开发者的下一代包管理器指南

> 如果你受够了 pip 的慢、virtualenv 的繁琐、pyenv + pipenv + poetry 的选择困难症，这篇文章介绍的 uv 可能是你的终极答案。

## 一、uv 是什么？

uv 是由 [Astral](https://astral.sh)（就是做 Ruff 那家公司）用 Rust 写的 Python 包管理器和项目管理工具。它的目标很简单：**用一个工具替代 pip、pip-tools、pipenv、poetry、pyenv、virtualenv 这一整套工具链。**

速度方面，uv 安装依赖比 pip 快 **10-100 倍**，不是夸张，是实测。

## 二、为什么 Python 需要 uv？

Python 的包管理一直是个老大难问题。来看看一个典型 Python 开发者的工具栈：

| 需求 | 传统方案 | 问题 |
|------|---------|------|
| 安装 Python 版本 | pyenv | 需要单独安装和配置 |
| 创建虚拟环境 | virtualenv / venv | 手动管理，容易忘记激活 |
| 安装依赖 | pip | 没有 lock 文件，解析慢 |
| 锁定依赖版本 | pip-tools / pipenv / poetry | 三个工具三种格式，互不兼容 |
| 运行脚本 | 手动激活 venv 后运行 | 繁琐 |

这意味着你要学 5 个工具，维护 5 套配置。而 Node.js 开发者只需要一个 npm（或 pnpm）就搞定了。

**uv 就是 Python 世界的 pnpm。** 一个工具，全部搞定。

## 三、安装 uv

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# 或者用 Homebrew
brew install uv

# Windows
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

安装完验证一下：

```bash
uv --version
```

## 四、核心用法

### 1. 管理 Python 版本（替代 pyenv）

不需要再装 pyenv 了，uv 自己就能管理 Python 版本：

```bash
# 安装指定版本的 Python
uv python install 3.12

# 安装多个版本
uv python install 3.11 3.12 3.13

# 查看已安装的版本
uv python list

# 固定项目使用的 Python 版本
uv python pin 3.12
```

`uv python pin` 会在项目根目录生成一个 `.python-version` 文件，团队成员 clone 项目后 uv 会自动使用对应版本。

### 2. 创建项目（替代 `mkdir + venv + pip`）

```bash
# 创建新项目
uv init my-project
cd my-project
```

这会生成：

```
my-project/
├── .python-version    # Python 版本
├── README.md
├── hello.py           # 示例入口
└── pyproject.toml     # 项目配置（统一标准）
```

`pyproject.toml` 是 Python 官方推荐的项目配置格式（PEP 621），uv 完全遵循这个标准，不搞私有格式。

### 3. 管理依赖（替代 pip + requirements.txt）

```bash
# 添加依赖
uv add requests
uv add flask sqlalchemy

# 添加开发依赖
uv add --dev pytest ruff mypy

# 移除依赖
uv remove requests

# 同步依赖（类似 npm install）
uv sync
```

执行 `uv add` 后，uv 会：
1. 自动创建虚拟环境（如果还没有）
2. 解析依赖关系
3. 安装包
4. 更新 `pyproject.toml` 和 `uv.lock`

`uv.lock` 是跨平台的 lock 文件，锁定所有依赖的精确版本，确保团队每个人装的都一模一样。**这个文件应该提交到 git。**

### 4. 运行代码（替代 `source venv/bin/activate + python`）

这是 uv 最爽的地方——你不需要手动激活虚拟环境：

```bash
# 直接运行，uv 自动使用项目的虚拟环境
uv run hello.py

# 运行模块
uv run -m pytest

# 运行任意命令
uv run flask run
```

`uv run` 会自动：
- 检查虚拟环境是否存在，不存在就创建
- 检查依赖是否最新，不是就同步
- 在正确的环境中执行命令

再也不用 `source .venv/bin/activate` 了。

### 5. 一次性运行工具（替代 pipx）

想临时用一个 CLI 工具，但不想污染全局环境？

```bash
# 不安装，直接运行
uvx ruff check .
uvx black .
uvx httpie https://api.github.com

# 等价于
uv tool run ruff check .
```

`uvx` 会在临时隔离环境中运行工具，用完即走，不留痕迹。类似 Node.js 的 `npx`。

### 6. 全局安装工具（替代 pipx install）

有些工具你天天用，想装到全局：

```bash
# 全局安装
uv tool install ruff
uv tool install httpie

# 查看已安装的全局工具
uv tool list

# 升级
uv tool upgrade ruff
```

每个工具都在独立的虚拟环境中，互不干扰。

## 五、实战：从零开始一个项目

完整走一遍流程：

```bash
# 1. 创建项目
uv init web-api
cd web-api

# 2. 固定 Python 版本
uv python pin 3.12

# 3. 添加依赖
uv add fastapi uvicorn[standard]
uv add --dev pytest httpx

# 4. 写代码
cat > main.py << 'EOF'
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"message": "Hello from uv!"}
EOF

# 5. 运行
uv run uvicorn main:app --reload

# 6. 运行测试
uv run -m pytest
```

全程没有 `python -m venv`，没有 `source activate`，没有 `pip install`。

## 六、迁移指南：从现有工具迁移到 uv

### 从 pip + requirements.txt 迁移

```bash
# 在项目目录下初始化 uv
uv init

# 从 requirements.txt 导入依赖
uv add $(cat requirements.txt)

# 之后就可以删掉 requirements.txt，用 pyproject.toml + uv.lock 管理了
```

### 从 poetry 迁移

uv 能直接读取 `pyproject.toml` 中 poetry 格式的依赖声明：

```bash
# 在 poetry 项目中直接运行
uv sync
```

uv 会解析 `pyproject.toml`，生成自己的 `uv.lock`，创建虚拟环境并安装所有依赖。

### 从 pipenv 迁移

```bash
# 从 Pipfile 导出，再导入 uv
pipenv requirements > requirements.txt
uv init
uv add $(cat requirements.txt)
```

## 七、uv 常用命令速查表

| 命令 | 作用 | 等价于 |
|------|------|--------|
| `uv init` | 创建新项目 | `mkdir + venv + pip init` |
| `uv add <pkg>` | 添加依赖 | `pip install + 写入 requirements.txt` |
| `uv remove <pkg>` | 移除依赖 | 手动编辑 requirements.txt + pip uninstall |
| `uv sync` | 同步所有依赖 | `pip install -r requirements.txt` |
| `uv run <cmd>` | 在项目环境中运行 | `source venv/bin/activate && cmd` |
| `uv lock` | 生成/更新 lock 文件 | `pip-compile` |
| `uv python install` | 安装 Python 版本 | `pyenv install` |
| `uv tool install` | 全局安装 CLI 工具 | `pipx install` |
| `uvx <tool>` | 临时运行工具 | `pipx run` / `npx` |
| `uv pip install` | 兼容 pip 的接口 | `pip install`（但快得多） |

## 八、为什么选 uv 而不是 poetry / pdm？

| 维度 | uv | poetry | pdm |
|------|-----|--------|-----|
| 语言 | Rust | Python | Python |
| 安装速度 | 极快（10-100x） | 慢 | 中等 |
| Python 版本管理 | ✅ 内置 | ❌ 需要 pyenv | ❌ 需要 pyenv |
| 全局工具管理 | ✅ 内置 | ❌ | ❌ |
| lock 文件 | ✅ 跨平台 | ✅ | ✅ |
| 标准兼容 | PEP 621 原生 | 有私有扩展 | PEP 621 原生 |
| 活跃度 | 非常活跃 | 活跃 | 活跃 |

uv 的核心优势：**快、全、标准**。一个工具覆盖整个 Python 工具链，且完全遵循 Python 官方标准。

## 九、注意事项

1. **uv 还在快速迭代中**——功能在不断增加，建议定期 `uv self update` 保持最新
2. **uv.lock 要提交到 git**——它保证团队环境一致
3. **不需要手动管理 .venv**——uv 会自动处理，你只管 `uv run`
4. **CI/CD 中也用 uv**——GitHub Actions 有官方的 `astral-sh/setup-uv` action

```yaml
# GitHub Actions 示例
- uses: astral-sh/setup-uv@v4
- run: uv sync
- run: uv run pytest
```

## 十、总结

uv 对 Python 生态的意义，类似于 pnpm 对 Node.js 生态的意义——它不是又一个包管理器，而是一个**统一的、极速的、标准的** Python 项目管理工具。

如果你今天开始一个新的 Python 项目，没有理由不用 uv。

> 相关链接：
> - [uv 官方文档](https://docs.astral.sh/uv/)
> - [uv GitHub 仓库](https://github.com/astral-sh/uv)
> - [Astral 官网](https://astral.sh)
