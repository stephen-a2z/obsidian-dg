---
{"dg-publish":true,"permalink":"/Git 同步 obsidian vault的操作步骤/","created":"2026-03-20T11:58:46.770+08:00","updated":"2026-03-20T12:00:21.224+08:00"}
---


Git 同步 vault的操作步骤

> ## 步骤一：GitHub 创建私有仓库

1. 打开 https://github.com/new
2. Repository name 填 obsidian-vault（随意）
3. 选择 Private
4. 不要勾选 README、.gitignore 等任何初始化选项
5. 点击 Create repository

## 步骤二：本地 vault 初始化 Git

打开终端，执行：

```bash
cd /path/to/your-vault
```
# 初始化
git init

# 创建 .gitignore
```bash
cat > .gitignore << 'EOF'
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.obsidian/plugins/obsidian-git/data.json
.trash/
.DS_Store
EOF
```

# 首次提交
```bash
git add .
git commit -m "init vault"
```

# 关联远程仓库并推送
```bash
git remote add origin git@github.com:你的用户名/obsidian-vault.git
git branch -M main
git push -u origin main
```


如果你没配过 SSH key，用 HTTPS 方式也行：
```bash
git remote add origin https://github.com/你的用户名/obsidian-vault.git
```

## 步骤三：安装 Obsidian Git 插件

1. Obsidian → Settings → Community plugins → Browse
2. 搜索 Obsidian Git，安装并启用

## 步骤四：配置 Obsidian Git

插件设置中关键项：

| 设置项 | 建议值 | 说明 |
|---|---|---|
| Auto backup every X minutes | 10 | 每 10 分钟自动 commit + push |
| Auto pull on open | 开启 | 打开 Obsidian 时自动拉取最新 |
| Pull on startup | 开启 | 同上，确保启动时同步 |
| Commit message | vault backup: {{date}} | 自动提交信息带时间戳 |

其余保持默认即可。

## 步骤五：其他设备配置

在新设备上：

bash
# 克隆仓库
git clone git@github.com:你的用户名/obsidian-vault.git

# 用 Obsidian 打开这个文件夹作为 vault


然后重复步骤三、四安装配置 Obsidian Git 插件。

## 步骤六：验证同步

1. 设备 A 新建一篇笔记，等待自动备份（或按 Ctrl/Cmd + P 输入 Obsidian Git: Commit and push）
2. 设备 B 打开 Obsidian，插件自动 pull，确认笔记出现

## 日常使用流程

打开 Obsidian → 插件自动 pull 最新内容
    → 正常写笔记
    → 插件每 10 分钟自动 commit + push
    → 想发布时用 Digital Garden 插件发布


基本上配好之后就不用管了，插件全自动处理。唯一要注意的是：如果两台设备同时编辑了同一个文件，pull 时会产生 Git 冲突，插件会弹通知提醒你手动解决。避免这种情况最简单的办法就是——换设备前先等自动备份完成，或者手动触发一次push。