# Android 目录权限问题

## 问题现象

部分 Android 设备上，杀戮尖塔的 `runs/` 子目录权限不一致，导致 ADB 无法访问某些角色的对局记录。

以下是实测的 `ls -la` 输出（路径：`/sdcard/Android/data/com.humble.SlayTheSpire/files/runs/`）：

```
drwxrws--- DAILY        ← ADB 可访问 ✓
drwx--S--- DEFECT       ← ADB 权限拒绝 ✗
drwxrws--- IRONCLAD     ← ADB 可访问 ✓
drwxrws--- THE_SILENT   ← ADB 可访问 ✓
drwx--S--- WATCHER      ← ADB 权限拒绝 ✗
```

同一父目录下的子目录，权限却不同：
- `drwxrws---`（770 + setgid）— group 有读写执行权限
- `drwx--S---`（700 + setgid）— group 无任何权限

### 测试设备

- 品牌：小米红米（Xiaomi Redmi）
- 型号：24117RK2CC
- 系统：Android 16（API 36），HyperOS V816（OS3.0.302.0.WOKCNXM）

## 根因分析

ADB shell 用户身份：

```
uid=2000(shell) gid=2000(shell) groups=...,1078(ext_data_rw),...
```

shell 用户属于 `ext_data_rw` 组。目录的 group 也是 `ext_data_rw`，因此：
- **770 目录**：group 有 rwx 权限 → shell 用户可以通过 group 权限访问 ✓
- **700 目录**：group 无权限 → shell 用户被拒 ✗

游戏在创建不同角色的 `runs/` 子目录时，使用了不一致的 umask，导致部分目录缺少 group 权限。具体哪些目录受影响可能因设备和游戏版本而异。

## 对脚本的影响

基于实测（使用 `drwx--S---` 权限的目录作为测试对象）：

| 操作 | 权限拒绝时行为 | exit code | 不存在时行为 | exit code |
|------|---------------|-----------|-------------|-----------|
| `adb shell ls` | `Permission denied` | **1** | `No such file or directory` | 1 |
| `adb pull 目录/` | `0 files pulled, 0 skipped` | **0** ⚠️ | `error: failed to stat` | 1 |
| `adb push` | `Permission denied` | **1** | — | — |

### 关键问题

**`adb pull` 在权限拒绝时返回 exit code 0**，这意味着：
- 脚本无法通过 `check=True` 捕获此错误
- `adb pull` 静默返回"0 files pulled"，脚本误以为成功
- 用户数据被**静默跳过**，不会收到任何错误提示

当前脚本的具体反应：

| 方向 | 操作 | 脚本反应 |
|------|------|---------|
| mobile_to_pc | `adb shell ls`（列出 runs 子目录） | catch 异常后打印"no history of runs...Skipping"（误导：实际是权限问题不是无数据） |
| mobile_to_pc | `adb pull`（拉取 preferences/saves） | 异常被 catch，打印"does not exist. Skipping"（误导） |
| pc_to_mobile | `adb push`（推送文件） | `check=True` 触发异常，脚本**直接崩溃** |

## 手动解决方案

### 前提条件

父目录 `runs/` 必须具有 `drwxrws---`（770 + setgid）权限，ADB 对其有写权限。

### 步骤

以 DEFECT 目录为例：

```bash
# 1. 将有问题的目录重命名（只需要父目录的写权限即可操作）
adb shell "mv /sdcard/Android/data/com.humble.SlayTheSpire/files/runs/DEFECT /sdcard/Android/data/com.humble.SlayTheSpire/files/runs/DEFECT_bak"

# 2. 新建目录（自动从 setgid 父目录继承 drwxrws--- 权限）
adb shell "mkdir /sdcard/Android/data/com.humble.SlayTheSpire/files/runs/DEFECT"

# 3. 验证新目录权限
adb shell "ls -la /sdcard/Android/data/com.humble.SlayTheSpire/files/runs/"
# 新的 DEFECT 应显示 drwxrws---
```

对其他有问题的目录（如 WATCHER）重复相同操作。

### 恢复数据

新建的目录是空的，旧目录（`DEFECT_bak`）内的数据因权限限制仍然无法通过 ADB 读取。需要从手机备份中恢复数据：

**小米手机**：
1. 手机上进入：设置 → 我的设备 → 备份与恢复 → 手机备份
2. 选择"第三方应用程序" → 勾选杀戮尖塔 → 立即备份
3. 备份文件位于手机存储的 `MIUI/backup/AllBackup/` 目录
4. USB 连接电脑后可以访问备份文件
5. 解压 `.bak` 文件，在 `apps/com.humble.SlayTheSpire/ef/runs/` 下找到对应的 run 文件
6. 将 run 文件通过 `adb push` 推送到新建的目录中

**其他品牌手机**：
- 请使用对应品牌的备份功能导出应用数据
- 也可以尝试第三方文件管理器（如具有 `Android/data` 访问权限的文件管理器）直接复制文件

## 注意事项

- **无破坏性**：旧目录只是改名保留，随时可以 `mv DEFECT_bak DEFECT` 回退
- **一次性修复**：修复后权限永久有效，后续脚本传输正常工作
- **_bak 目录残留**：无法通过 ADB 删除（权限不足），会残留在手机上，占用空间极小（几个 JSON 文件，几 KB），不影响游戏运行
- **新目录 owner**：新建目录的 owner 是 `shell` 而非 app 用户 `u0_a383`，实测未发现影响游戏的读写行为
- **preferences 和 saves 目录**：如果这些目录也有权限问题，同样的方法适用（前提是它们的父目录权限正常）

## 脚本改进方向（TODO）

### 1. 权限预检测 + 警告（优先级高）

在 `validate_adb()` 之后、实际传输之前，用 `adb shell ls -la` 检查 `preferences/`、`runs/` 子目录、`saves/` 的权限。发现 `drwx--S---`（700）目录时打印警告：

```
⚠ WARNING: Permission issue detected!
  The following directories have restrictive permissions (drwx--S---)
  and may not be accessible:
    - runs/DEFECT
    - runs/WATCHER
  Data in these directories may be silently skipped during transfer.
  See: https://github.com/TheShiftingQuiet/Slay-the-Spire-save-transfer/issues/2
```

特点：
- 只检测只警告，不修改任何数据，无破坏性
- 不阻断脚本执行，用户可忽略警告继续运行
- 把"静默跳过数据"变为"明确告知用户"

### 2. 自动修复（优先级低，需谨慎设计）

自动执行 rename → recreate 流程。难点在于：
- 新目录是空的，脚本无法从旧目录（_bak）读取数据
- 需要用户手动从手机备份中提取数据并提供给脚本
- 涉及用户交互和外部数据源，流程复杂
- 好在修复是一次性永久的，不需要每次运行

如果实现，建议采用**交互式**方式：检测到问题 → 告知用户 → 询问是否自动修复 → 用户确认后执行 rename + recreate → 提示用户从备份恢复数据。

## 相关链接

- 上游 Issue：[Permission denied #2](https://github.com/TheShiftingQuiet/Slay-the-Spire-save-transfer/issues/2)
