---
{"dg-publish":true,"permalink":"/01-技术分享/Selenium 启动 Chromium 报错 \"DevToolsActivePort file doesn't exist\" 排查实录/","title":"Selenium 启动 Chromium 报错 \"DevToolsActivePort file doesn't exist\" 排查实录","noteIcon":"","created":"2026-05-27T15:27:57.153+08:00","updated":"2026-05-27T15:29:03.072+08:00"}
---


# Selenium 启动 Chromium 报错 "DevToolsActivePort file doesn't exist" 排查实录

## 环境

- Ubuntu 18.04.6 LTS (Kernel 4.15.0-177-generic)
- Chromium 112.0.5615.49（deb 包安装）
- ChromeDriver 112.0.5615.49
- Selenium 4.43.0 / Python 3.11

## 问题现象

Selenium 通过 ChromeDriver 启动 Chromium 时报错：

```
selenium.common.exceptions.WebDriverException: Message: unknown error: Chrome failed to start: exited abnormally.
  (unknown error: DevToolsActivePort file doesn't exist)
  (The process started from chrome location /usr/bin/chromium-browser is no longer running, so ChromeDriver is assuming that Chrome has crashed.)
```

关键特征：
- **不带** `--user-data-dir` 参数时可以正常启动
- **带** `--user-data-dir` 指向特定目录时必现崩溃
- 问题出现在服务器安装 Docker 环境之后

## 排查过程

### 第一步：排除 AppArmor/沙箱问题

由于问题在安装 Docker 后出现，首先怀疑 Docker 安装过程修改了系统安全策略（AppArmor、namespace 限制等）。

诊断结果：
- AppArmor 无 Chromium 相关 profile，无 DENIED 日志
- Chromium 是 deb 包安装，不受 snap confinement 影响
- **命令行直接执行 `chromium-browser --headless --user-data-dir=... about:blank` 正常**

结论：不是系统安全策略问题。

### 第二步：排除 iptables 规则干扰

Docker 安装后会将 iptables FORWARD 链默认策略改为 DROP，怀疑影响了 ChromeDriver 与 Chromium 之间的 localhost 通信。

编写对比测试脚本：临时将 FORWARD 策略改为 ACCEPT，对比 Selenium 启动结果。

诊断结果：两种策略下 Selenium 均能正常启动（使用新建的 user-data-dir）。

结论：不是 iptables 问题。

### 第三步：精确对比定位

设计四组对比测试：

| 测试 | 条件 | 结果 |
|------|------|------|
| 1 | `--headless` + 新目录 | ✅ 成功 |
| 2 | `--headless=new` + 新目录 | ✅ 成功 |
| 3 | 完整参数 + 新目录 | ✅ 成功 |
| 4 | `--headless=new` + **已存在的实际目录** | ❌ 失败 |

结论：问题不在启动参数，而在**那个特定的 user-data-dir 目录本身**。

### 第四步：检查目录内容

```bash
ls /tmp/chrome_112/.../account-.../Singleton*
```

发现三个残留文件：
- `SingletonLock`
- `SingletonSocket`
- `SingletonCookie`

删除后问题解决。

## 根因

### Chromium 的单实例锁机制

Chromium 使用 `Singleton*` 文件实现 profile 目录的互斥访问：

```
启动 → 创建 SingletonLock/Socket/Cookie → 运行 → 正常退出 → 删除锁文件
```

三个文件各司其职：
- **SingletonLock**：符号链接，内容为 `hostname-PID`，标记持有者
- **SingletonSocket**：Unix domain socket，用于向已有实例发送消息（如"打开新标签页"）
- **SingletonCookie**：随机 token，验证 socket 连接的合法性

### 为什么锁文件会残留

安装 Docker 过程中系统服务重启（或手动 kill 了进程），Chromium 被 SIGKILL 强制终止，没有机会执行退出清理逻辑，锁文件留在磁盘上。

### 为什么残留锁文件会导致崩溃

1. Chromium 启动，检测到 `SingletonLock` 存在
2. 读取锁文件中的 PID，尝试通过 `SingletonSocket` 连接"已有实例"
3. 原进程早已不存在，连接失败
4. Chromium 112 在 headless 模式下对这种情况的处理不够健壮，直接退出
5. ChromeDriver 等待 `DevToolsActivePort` 文件超时，报错

### 为什么不带 --user-data-dir 时正常

不指定 `--user-data-dir` 时，Chromium 使用默认的 `~/.config/chromium` 目录。该目录没有残留锁文件（或者默认目录的锁文件在系统重启时已被清理），所以正常。

### Docker 安装的关联

Docker 本身不是直接原因。真正的因果链是：

```
安装 Docker → 系统服务重启/进程被杀 → Chromium 异常退出 → 锁文件残留 → 后续启动失败
```

任何导致 Chromium 非正常退出的操作都会触发同样的问题。

## 解决方案

### 临时修复

```bash
rm -f /path/to/user-data-dir/Singleton*
```

### 长期修复：启动前清理锁文件

```python
import os
import glob

def clean_profile_locks(user_data_dir):
    """清理 Chromium profile 目录中的残留锁文件"""
    for f in glob.glob(os.path.join(user_data_dir, "Singleton*")):
        os.remove(f)

# 在创建 WebDriver 之前调用
clean_profile_locks(user_data_dir)
driver = webdriver.Chrome(options=options)
```

### 更健壮的方案：检查锁文件对应的进程是否存活

```python
import os

def clean_stale_locks(user_data_dir):
    lock_file = os.path.join(user_data_dir, "SingletonLock")
    if os.path.islink(lock_file):
        target = os.readlink(lock_file)  # 格式: hostname-PID
        try:
            pid = int(target.split("-")[-1])
            os.kill(pid, 0)  # 检查进程是否存在
        except (ValueError, ProcessLookupError, PermissionError):
            # 进程不存在，清理死锁
            for f in glob.glob(os.path.join(user_data_dir, "Singleton*")):
                os.remove(f)
```

## 经验总结

1. **"DevToolsActivePort file doesn't exist" 不一定是参数或环境问题**，也可能是 profile 目录状态异常
2. **对比测试是最有效的排查手段**：控制变量，逐步缩小范围
3. **自动化场景中应始终处理锁文件清理**，因为进程随时可能被异常终止
4. **时间相关性 ≠ 因果关系**：问题在"安装 Docker 后"出现，但 Docker 只是触发了进程异常退出的间接原因
