---
{"dg-publish":true,"permalink":"/01-技术分享/不小心提交了敏感信息到github仓库怎么办/","tags":["git","filter-repo","工具","入门教程"],"noteIcon":"","created":"2026-04-23T18:25:03.197+08:00","updated":"2026-04-23T18:49:14.687+08:00"}
---


# git filter-repo：重写 Git 历史的正确姿势

> 仓库里不小心提交了密钥？历史里堆满了大文件？想把子目录拆成独立仓库？`git filter-repo` 是 Git 官方推荐的历史重写工具，比 `filter-branch` 快几十倍，用法也更直观。

## 一、它是什么？

`git filter-repo` 是一个基于 Python 的 Git 历史重写工具，用来替代已被官方标记为"不推荐使用"的 `git filter-branch`。

核心优势：

- **快**：比 `filter-branch` 快 10-100 倍
- **安全**：默认要求在全新克隆上操作，防止误操作
- **简洁**：一行命令搞定大多数场景

## 二、安装

```bash
# macOS
brew install git-filter-repo

# pip（任何平台）
pip3 install git-filter-repo
```

安装后直接作为 `git` 子命令使用：`git filter-repo ...`

## 三、使用前必读

`filter-repo` 会**重写提交哈希**，这意味着：

1. 必须在**全新克隆**的仓库上操作（或加 `--force`）
2. 操作完成后需要 `git push --force`
3. 所有协作者需要**重新克隆**仓库
4. 操作前务必**备份**

典型的工作流：

```bash
git clone --mirror https://github.com/user/repo.git
cd repo.git
# 执行 filter-repo 操作...
git push --force
```

## 四、常用场景

### 1. 删除敏感文件

把 `secrets.env` 从所有历史提交中彻底移除：

```bash
git filter-repo --path secrets.env --invert-paths
```

`--invert-paths` 的意思是"排除匹配的路径"，即删除它。

删除多个文件：

```bash
git filter-repo --path secrets.env --path config/keys.json --invert-paths
```

### 2. 删除整个目录

```bash
git filter-repo --path build/ --path dist/ --invert-paths
```

### 3. 只保留某个子目录（拆分子项目）

把 `src/` 提升为仓库根目录，其他文件全部移除：

```bash
git filter-repo --subdirectory-filter src/
```

适合从 monorepo 中拆分独立项目。

### 4. 替换历史中的敏感文本

创建 `replacements.txt`：

```
my_actual_password==>REDACTED
sk-xxxxxxxxxxxxxxxx==>REDACTED
```

执行替换：

```bash
git filter-repo --replace-text replacements.txt
```

所有提交中的匹配文本都会被替换，适合清理泄露的密钥。

### 5. 修改提交者信息

```bash
git filter-repo --email-callback '
    return email if email != b"old@example.com" else b"new@example.com"
'
```

也可以同时改名字：

```bash
git filter-repo --name-callback '
    return name if name != b"Old Name" else b"New Name"
' --email-callback '
    return email if email != b"old@example.com" else b"new@example.com"
'
```

### 6. 重命名/移动路径

把历史中所有 `old/path/` 改为 `new/path/`：

```bash
git filter-repo --path-rename old/path/:new/path/
```

### 7. 删除大文件（仓库瘦身）

删除历史中所有超过 10MB 的文件：

```bash
git filter-repo --strip-blobs-bigger-than 10M
```

## 五、分析仓库（只看不改）

在动手之前，先用 `--analyze` 了解仓库情况：

```bash
git filter-repo --analyze
```

会在 `.git/filter-repo/analysis/` 生成报告，包含：

- 按大小排序的 blob 列表（找出大文件）
- 路径统计
- 扩展名统计

建议每次操作前都先跑一遍分析。

## 六、与 BFG Repo-Cleaner 的对比

| 特性 | git filter-repo | BFG Repo-Cleaner |
|------|----------------|-----------------|
| 语言 | Python | Java（需要 JRE） |
| 速度 | 非常快 | 快 |
| 功能范围 | 全面（路径、内容、作者等） | 主要针对大文件和敏感数据 |
| 官方推荐 | ✅ Git 官方推荐 | ❌ |
| 维护状态 | 活跃 | 基本停止更新 |

结论：优先用 `filter-repo`。

## 七、实战：清理一个臃肿的仓库

完整流程示例：

```bash
# 1. 全新克隆
git clone --mirror https://github.com/user/repo.git
cd repo.git

# 2. 分析，找出问题
git filter-repo --analyze
cat .git/filter-repo/analysis/blob-shas-and-paths.txt | head -20

# 3. 删除大文件 + 敏感文件
git filter-repo --strip-blobs-bigger-than 10M \
    --path .env --path credentials.json --invert-paths

# 4. 推送
git push --force
```

## 八、常见坑：push 时报 no upstream branch

执行 `filter-repo` 后直接 `git push --force` 会报错：

```
fatal: The current branch main has no upstream branch.
```

这是因为 `filter-repo` 会**自动移除所有 remote 配置**，这是它的安全机制，防止你不小心 force push 到原仓库。

解决方法：

```bash
# 1. 重新添加 remote（地址换成你自己的）
git remote add origin https://github.com/your-user/your-repo.git

# 2. force push 并设置 upstream
git push --force --set-upstream origin main
```

如果忘了原来的 remote 地址，filter-repo 有备份：

```bash
cat .git/filter-repo/clone-url
```

## 九、操作完成后，原本地仓库怎么办？

远程历史被重写后，原本地仓库的提交哈希全部对不上了。最干净的做法是**删掉重新克隆**：

```bash
rm -rf old-local-repo
git clone https://github.com/your-user/your-repo.git
```

如果本地有未提交的修改想保留：

```bash
# 1. 备份未提交的改动
cp -r old-local-repo/some-changed-files /tmp/backup/

# 2. 重新克隆
rm -rf old-local-repo
git clone https://github.com/your-user/your-repo.git

# 3. 把改动复制回去
cp -r /tmp/backup/* your-repo/
```

> ⚠️ 不要尝试在旧仓库上 `git pull --rebase` 或 `git fetch && git reset`，历史完全不同了，容易出各种问题。直接重新克隆最干净。

团队中的所有成员都需要执行同样的操作。

## 十、总结

`git filter-repo` 是处理 Git 历史问题的首选工具。记住三个原则：

1. **先分析再动手**：`--analyze` 先跑一遍
2. **在全新克隆上操作**：别在工作仓库上直接搞
3. **通知团队**：操作后所有人需要重新克隆
