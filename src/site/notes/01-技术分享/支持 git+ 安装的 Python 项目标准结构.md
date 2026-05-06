---
{"dg-publish":true,"permalink":"/01-技术分享/支持 git+ 安装的 Python 项目标准结构/","tags":["python","git","module"],"noteIcon":"","created":"2026-05-06T12:43:17.974+08:00","updated":"2026-05-06T12:46:21.162+08:00"}
---


# 支持 git+ 安装的 Python 项目标准结构

## 引言

在 Python 生态中，`pip install git+https://github.com/user/repo.git` 是一种常见的安装方式，它允许用户直接从 Git 仓库安装 Python 包。然而，要让你的项目完美支持这种安装方式，需要遵循特定的项目结构规范。本文将详细介绍支持 `git+` 安装的 Python 项目标准结构。

## 为什么需要标准结构？

1. **可安装性**：确保 `pip install git+...` 能够正确识别和安装你的包
2. **可发现性**：让 Python 工具链（如 `setuptools`、`pip`）能够找到你的代码
3. **一致性**：遵循社区约定，降低用户的学习成本
4. **可维护性**：清晰的目录结构便于长期维护

## 标准项目结构

```
my_package/
├── pyproject.toml          # 现代 Python 项目配置（推荐）
├── setup.py                # 传统配置（兼容性）
├── setup.cfg              # 传统配置（可选）
├── README.md              # 项目说明文档
├── LICENSE                # 许可证文件
├── .gitignore             # Git 忽略文件
├── requirements.txt       # 依赖声明（可选）
├── requirements-dev.txt   # 开发依赖（可选）
├── src/                   # 源代码目录（推荐结构）
│   └── my_package/
│       ├── __init__.py
│       ├── module1.py
│       └── module2.py
├── tests/                 # 测试目录
│   ├── __init__.py
│   ├── test_module1.py
│   └── test_module2.py
├── docs/                  # 文档目录（可选）
└── examples/              # 示例代码（可选）
```

## 核心配置文件详解

### 1. `pyproject.toml`（现代标准）

```toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "my-package"
version = "0.1.0"
description = "My awesome Python package"
readme = "README.md"
license = {text = "MIT"}
authors = [
    {name = "Your Name", email = "your.email@example.com"}
]
classifiers = [
    "Programming Language :: Python :: 3",
    "License :: OSI Approved :: MIT License",
    "Operating System :: OS Independent",
]

dependencies = [
    "requests>=2.25.0",
    "numpy>=1.20.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=6.0",
    "black>=21.0",
    "mypy>=0.900",
]

[project.urls]
Homepage = "https://github.com/user/repo"
Repository = "https://github.com/user/repo"
Issues = "https://github.com/user/repo/issues"

[tool.setuptools]
package-dir = {"" = "src"}
packages = {find = {where = ["src"]}}
```

### 2. `setup.py`（传统方式，兼容性）

```python
from setuptools import setup, find_packages

setup(
    name="my-package",
    version="0.1.0",
    description="My awesome Python package",
    long_description=open("README.md").read(),
    long_description_content_type="text/markdown",
    author="Your Name",
    author_email="your.email@example.com",
    url="https://github.com/user/repo",
    packages=find_packages(where="src"),
    package_dir={"": "src"},
    install_requires=[
        "requests>=2.25.0",
        "numpy>=1.20.0",
    ],
    extras_require={
        "dev": [
            "pytest>=6.0",
            "black>=21.0",
            "mypy>=0.900",
        ]
    },
    classifiers=[
        "Programming Language :: Python :: 3",
        "License :: OSI Approved :: MIT License",
        "Operating System :: OS Independent",
    ],
    python_requires=">=3.7",
)
```

## 关键要点

### 1. 源代码组织

**推荐使用 `src/` 布局**：
```python
# 正确：src/ 布局
my_package/
├── src/
│   └── my_package/
│       ├── __init__.py
│       └── module.py

# 不推荐：扁平布局
my_package/
├── my_package/
│   ├── __init__.py
│   └── module.py
```

`src/` 布局的优势：
- 避免导入本地开发版本而非安装版本
- 更清晰的分离源代码和项目文件
- 符合现代 Python 打包最佳实践

### 2. `__init__.py` 文件

每个包目录都需要 `__init__.py` 文件（即使是空的）：
```python
# src/my_package/__init__.py
__version__ = "0.1.0"
__author__ = "Your Name"

# 可选：导出主要功能
from .module1 import main_function
from .module2 import helper_function

__all__ = ["main_function", "helper_function"]
```

### 3. 版本管理

**推荐使用动态版本**：
```python
# setup.py 中的动态版本
import re

def get_version():
    with open("src/my_package/__init__.py") as f:
        content = f.read()
        match = re.search(r'__version__\s*=\s*["\']([^"\']+)["\']', content)
        return match.group(1) if match else "0.1.0"

setup(
    version=get_version(),
    # ...
)
```

### 4. 依赖管理

**明确指定依赖版本**：
```python
# 好：明确版本范围
install_requires=[
    "requests>=2.25.0,<3.0.0",
    "numpy>=1.20.0,<2.0.0",
]

# 不好：过于宽松
install_requires=["requests", "numpy"]
```

## 安装方式对比

### 1. 基本安装
```bash
# 从 PyPI 安装
pip install my-package

# 从 Git 仓库安装（支持 git+ 安装）
pip install git+https://github.com/user/repo.git

# 安装特定分支
pip install git+https://github.com/user/repo.git@main

# 安装特定标签
pip install git+https://github.com/user/repo.git@v0.1.0

# 安装特定提交
pip install git+https://github.com/user/repo.git@abc123def456
```

### 2. 开发模式安装
```bash
# 可编辑模式安装（便于开发）
pip install -e git+https://github.com/user/repo.git#egg=my-package

# 或使用本地克隆
git clone https://github.com/user/repo.git
cd repo
pip install -e .
```

## 常见问题与解决方案

### 问题1：`pip install git+...` 找不到包

**症状**：
```
ERROR: Directory '.' is not installable. Neither 'setup.py' nor 'pyproject.toml' found.
```

**解决方案**：
- 确保项目根目录包含 `setup.py` 或 `pyproject.toml`
- 检查文件命名是否正确（大小写敏感）

### 问题2：导入时找不到模块

**症状**：
```
ModuleNotFoundError: No module named 'my_package'
```

**解决方案**：
- 使用 `src/` 布局
- 确保 `setup.py` 中正确配置 `package_dir` 和 `packages`
- 检查 `__init__.py` 文件是否存在

### 问题3：版本冲突

**症状**：
```
pkg_resources.VersionConflict: (package 1.0.0 (/path), Requirement.parse('package>=2.0.0'))
```

**解决方案**：
- 在 `setup.py` 或 `pyproject.toml` 中明确指定依赖版本范围
- 使用 `~=` 操作符指定兼容版本

## 最佳实践

### 1. 使用现代工具链
```bash
# 使用 uv（更快）
uv pip install git+https://github.com/user/repo.git

# 使用 pdm
pdm add git+https://github.com/user/repo.git

# 使用 poetry
poetry add git+https://github.com/user/repo.git
```

### 2. 自动化测试
```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.8", "3.9", "3.10", "3.11"]

    steps:
    - uses: actions/checkout@v3
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}
    
    - name: Install package
      run: pip install git+https://github.com/${{ github.repository }}.git
    
    - name: Run tests
      run: python -m pytest tests/
```

### 3. 版本发布自动化
```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: "3.10"
    
    - name: Build package
      run: |
        pip install build
        python -m build
    
    - name: Publish to PyPI
      uses: pypa/gh-action-pypi-publish@release/v1
      with:
        password: ${{ secrets.PYPI_API_TOKEN }}
```

## 总结

支持 `git+` 安装的 Python 项目需要：

1. **标准文件结构**：包含必要的配置文件（`pyproject.toml`/`setup.py`）
2. **正确的源代码布局**：推荐使用 `src/` 布局
3. **明确的依赖声明**：在配置文件中指定依赖版本
4. **完整的元数据**：包含版本、作者、许可证等信息
5. **良好的文档**：README、许可证、贡献指南等

遵循这些规范不仅能确保你的项目可以通过 `git+` 方式安装，还能提高项目的可维护性和用户体验。随着 Python 打包生态的发展，建议优先使用 `pyproject.toml` 作为现代项目的标准配置。

## 参考资源

1. [Python Packaging User Guide](https://packaging.python.org/)
2. [setuptools Documentation](https://setuptools.pypa.io/)
3. [PEP 517 – Build system independent build frontend](https://peps.python.org/pep-0517/)
4. [PEP 518 – Specifying Minimum Build System Requirements for Python Projects](https://peps.python.org/pep-0518/)
5. [PEP 621 – Storing project metadata in pyproject.toml](https://peps.python.org/pep-0621/)

---

*本文基于 Python 3.8+ 和现代 Python 打包最佳实践编写。随着工具链的发展，建议关注官方文档获取最新信息。*