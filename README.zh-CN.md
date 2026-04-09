[![English](https://img.shields.io/badge/lang-English-blue.svg?style=for-the-badge)](README.md) [![中文](https://img.shields.io/badge/lang-简体中文-red.svg?style=for-the-badge)](README.zh-CN.md)

# 简介

本项目可以通过 USB 连接和 ADB，在 PC 和 Android 手机之间双向迁移杀戮尖塔的存档数据，包括：当前进行中的存档、历史对局记录、统计数据和解锁内容等。

你**不需要** root 手机 —— 本脚本在非 root 的 Android 手机上测试通过。

## 前置条件

1. PC 和 Android 手机上都拥有 **杀戮尖塔**（PC 端测试了 Steam 版本，Android 端测试了 Google Play 版本）
2. PC 上安装 **Python 3.9** 或更高版本（确保已添加到 PATH 环境变量）
3. Android 手机开启 **USB 调试**，并准备好 USB 数据线
4. PC 上安装 **Android Debug Bridge**（`adb`）并添加到 PATH 环境变量
   - Windows 用户可以下载 [Android SDK Platform Tools](https://dl.google.com/android/repository/platform-tools-latest-windows.zip)，解压后将文件夹路径添加到 PATH
5. 验证安装：在命令提示符中运行 `adb version` 和 `python --version`，确认都能正常输出

> **注意**：
> 同步**不是自动持续的**。每次迁移数据时，本质上是用一个设备的数据**覆盖**另一个设备的数据。
>
> 例如，如果你在 PC 上有了新进度，却把手机上的旧数据传回 PC，PC 上的新进度就会丢失。虽然历史对局记录不会丢失，但进度、解锁和当前存档会被回退到手机上的状态。
>
> 因此，**请仔细选择迁移方向**，并强烈建议在每次迁移前备份 `preferences`、`runs`、`saves` 三个文件夹。

# 常见问题

1. **这个工具能迁移所有数据吗？**

   基本上能迁移所有相关数据（设置、按键映射等跨设备无意义的除外）。包括：历史对局记录（含时间戳转换）、统计数据和解锁内容、进行中的存档（自动处理 XOR 加密转换）。

2. **迁移到 PC 后能获得 Steam 成就吗？**

   可以！测试发现，把手机上的对局记录迁移到 PC 后，启动游戏时会自动解锁对应的 Steam 成就。游戏似乎会在启动时根据历史对局记录来发放成就。

3. **这样做安全吗？**

   安全。所有逻辑都在一个 Python 脚本中，仅使用标准库，代码完全透明。脚本只进行复制和覆盖操作，不会自动删除任何它未创建的文件。仍然建议做好备份以防选错方向。

4. **会被封号吗？**

   极不可能。你合法拥有两个平台的游戏，开发者也表示支持手动迁移存档。

5. **每次都需要手动运行吗？**

   是的，不会自动同步。每次需要迁移时都要按照步骤手动操作。

6. **支持 iOS 吗？**

   不支持。本脚本使用 ADB，仅适用于 Android。

7. **支持 MacOS/Linux 吗？**

   仅在 Windows 上测试过，理论上应该跨平台可用。

8. **支持 Mod 吗？**

   未测试。简单的 Mod 可能没问题，但存储额外数据的 Mod 可能会导致手机版崩溃。

# 配置

编辑项目根目录下的 `config.ini` 文件：

```ini
[DEFAULT]
; PC 上杀戮尖塔的安装路径
; Steam 中右键游戏 -> 属性 -> 已安装文件 -> 浏览，复制该路径
PcPathToGame = C:\Program Files (x86)\Steam\steamapps\common\SlayTheSpire\

; Android 上杀戮尖塔的数据路径（大多数情况下无需修改）
AndroidPathToGame = /sdcard/Android/data/com.humble.SlayTheSpire/

; 你的时区偏移（相对于 GMT，单位：小时）
; 例如：北京时间 GMT+8 填 +8，美东时间 GMT-5 填 -5
LocalTimezoneOffsetHours = +8
```

# 从 PC 同步到手机

1. 检查 `config.ini` 配置是否正确
2. 手机开启 USB 调试
3. 用 USB 线连接手机和电脑
4. 双击 `pc-to-mobile.bat` 或运行 `python mover.py pc_to_mobile`

# 从手机同步到 PC

1. 检查 `config.ini` 配置是否正确
2. 手机开启 USB 调试
3. 用 USB 线连接手机和电脑
4. 双击 `mobile-to-pc.bat` 或运行 `python mover.py mobile_to_pc`

# 故障排除

### PC 到手机后，旧数据残留或同步被撤销？

1. 在 Android 手机上打开 **Play 游戏** 应用（**不是** Play 商店），进入 `设置 > 你的数据 > 删除 Play 游戏账号和数据 > 删除单个游戏数据 > Slay the Spire > 删除`
2. 开启飞行模式
3. 使用文件管理器进入 `内部存储/Android/data/com.humble.SlayTheSpire/files`，删除所有内容，然后创建空的 `preferences` 和 `saves` 文件夹

### 手机到 PC 后，旧数据残留或同步被撤销？

1. Steam 云存档可能干扰同步，可以临时禁用
2. 进入 PC 上杀戮尖塔的安装目录（Steam 中右键 -> 属性 -> 已安装文件 -> 浏览）
3. 删除 `runs`、`saves` 和 `preferences` 文件夹（建议先备份），然后创建空的 `preferences` 和 `saves` 文件夹

### Python 脚本报错？

1. 检查是否是配置问题
2. 确认 Python 版本至少为 3.9

# Fork 改进

本 Fork 相对上游仓库的改进：

- **每日挑战支持**：`RUNS_DIRNAMES` 新增 `DAILY`，支持每日挑战对局记录迁移
- **UTF-8 编码修复**：所有 `open()` 调用添加 `encoding='utf-8'`，修复中日韩 Windows 系统上的编码崩溃
- **权限问题文档**：记录了部分 Android 设备上目录权限不一致的问题及解决方案，详见 [docs/permission-issue.md](docs/permission-issue.md)

# 相关链接

- 上游仓库：[TheShiftingQuiet/Slay-the-Spire-save-transfer](https://github.com/TheShiftingQuiet/Slay-the-Spire-save-transfer)
- 权限问题文档：[docs/permission-issue.md](docs/permission-issue.md)
