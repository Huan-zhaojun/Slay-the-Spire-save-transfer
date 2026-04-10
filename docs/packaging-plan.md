# Slay the Spire 存档传输工具 — PyInstaller 打包计划

## Context

上游作者不打算做编译打包，用户需要自行安装 Python 和 ADB 才能使用。目标是将工具打包为 Windows exe，降低使用门槛。ADB 不捆绑分发（Google SDK 协议禁止再分发预构建二进制），改为首次运行时从 Google 官方 URL 自动下载（合法合规）。

## 方案概要

- **打包工具**：PyInstaller，文件夹模式
- **目标平台**：仅 Windows
- **ADB 策略**：首次运行自动从 `dl.google.com` 下载（自动走系统代理），解压到本地 `platform-tools/`
- **config.ini**：首次运行自动扫描生成，用户确认即可；也可手动编辑

## 分发目录结构

```
SlayTheSpire-SaveTransfer/
├── mover.exe                    # 主程序
├── config.ini                   # 首次运行自动生成，用户可编辑
├── pc-to-mobile.bat             # 双击即用
├── mobile-to-pc.bat             # 双击即用
├── _internal/                   # PyInstaller 运行时（自动生成）
├── platform-tools/              # 首次运行自动下载
│   ├── adb.exe
│   ├── AdbWinApi.dll
│   └── AdbWinUsbApi.dll
└── tmp/                         # 运行时临时目录
```

## 小白用户体验流程

```
① 下载 zip 解压
② 手机开启 USB 调试，连电脑
③ 双击 pc-to-mobile.bat（或 mobile-to-pc.bat）
④ 首次运行自动引导：
   → "正在扫描 Steam 游戏目录..."
   → "找到杀戮尖塔：D:\Steam\steamapps\common\SlayTheSpire\"
   → "检测到系统时区：UTC+8"
   → "是否使用以上配置？[Y/n]"
   → 自动生成 config.ini
   → "ADB 未找到，是否自动下载？[Y/n]"
   → 自动下载 ADB（走系统代理）
⑤ 自动完成传输
```

**后续使用只需：双击 bat → 自动完成**

## 实施步骤

### 步骤 1：新建 `adb_utils.py` — ADB 路径解析与自动下载

纯 stdlib 模块（urllib.request + zipfile + shutil），约 120 行。

核心函数：
- `get_base_dir()` — 返回 exe 所在目录（PyInstaller: `sys.executable`）或脚本所在目录（普通: `__file__`）
- `find_adb()` — 按优先级查找：本地 `platform-tools/` → 系统 PATH → 提示自动下载
- `_download_adb()` — 从 `https://dl.google.com/android/repository/platform-tools-latest-windows.zip` 下载解压，带进度显示

代理说明：`urllib.request` 默认读取 Windows 系统代理设置（注册表）和 `HTTP_PROXY`/`HTTPS_PROXY` 环境变量，Clash/V2Ray 等代理客户端设置的系统代理会自动生效。下载失败时打印手动下载引导 URL。

### 步骤 2：新建 `setup_utils.py` — 首次运行引导

纯 stdlib 模块（winreg + datetime + configparser），约 100 行。

核心函数：
- `auto_detect_game_path()` — 自动扫描杀戮尖塔安装目录：
  1. 从 Windows 注册表读取 Steam 安装路径（`HKCU\Software\Valve\Steam\SteamPath`）
  2. 解析 `steamapps/libraryfolders.vdf` 获取所有 Steam 库文件夹（多磁盘）
  3. 在每个库里查找 `steamapps/common/SlayTheSpire/`
  4. 验证目录包含 `preferences/` 或 `runs/` 或 `saves/`
- `auto_detect_timezone()` — 自动检测系统时区偏移：
  - `datetime.now().astimezone().utcoffset()` 获取 UTC 偏移
  - 转换为 `+8`、`-5`、`+5:45` 格式
- `interactive_setup()` — 首次引导主流程：
  - 调用以上两个函数，展示检测结果
  - 用户确认或手动输入
  - 生成 config.ini

### 步骤 3：修改 `mover.py` — 集成所有模块

**3a. 路径解析**：将 `os.getcwd()` 替换为 `get_base_dir()`（从 adb_utils 导入）

**3b. ADB 路径参数化**：19 处 `subprocess.run(["adb", ...])` 全部替换为 `[adb_path, ...]`。函数签名变更：
- `validate_adb(adb_path)`
- `push_files(..., adb_path, ...)`
- `pull_files(..., adb_path)`
- `copy_runs_directory(..., adb_path)`
- `pull_encoded_json(..., adb_path)`
- `main()` 开头调用 `find_adb()` 获取路径，传递给所有函数

**3c. config.ini 首次引导**：`load_config()` 中检测 config.ini 不存在时，调用 `interactive_setup()` 自动扫描并生成配置，然后继续正常执行（无需重启）

### 步骤 4：新建 `mover.spec` — PyInstaller 构建规格

- 文件夹模式（`exclude_binaries=True` + `COLLECT`）
- `console=True`（命令行交互）
- 不打包 config.ini（首次运行自动生成）
- `adb_utils.py` 和 `setup_utils.py` 作为 import 依赖自动打包

### 步骤 5：新建 `build.py` — 一键构建脚本

- 调用 PyInstaller 构建
- 生成 bat 启动脚本（`mover.exe pc_to_mobile` / `mover.exe mobile_to_pc`）
- 输出到 `dist/SlayTheSpire-SaveTransfer/`

### 步骤 6：更新 `.gitignore`

追加：`/dist`, `/build`, `platform-tools/`, `platform-tools.zip`, `__pycache__/`, `*.pyc`

## 文件变更清单

| 文件 | 操作 | 说明 |
|------|------|------|
| `adb_utils.py` | 新建 | ADB 发现、下载、路径管理 |
| `setup_utils.py` | 新建 | 首次运行引导：游戏路径扫描 + 时区检测 + config 生成 |
| `mover.py` | 修改 | get_base_dir + ADB 路径参数化 + 首次引导集成 |
| `mover.spec` | 新建 | PyInstaller 构建规格 |
| `build.py` | 新建 | 一键构建脚本 |
| `.gitignore` | 修改 | 追加构建产物忽略规则 |

## 注意事项

- **代理**：urllib.request 默认走系统代理，无需额外配置
- **杀软误报**：未签名 PyInstaller exe 可能被 Windows Defender/SmartScreen 拦截，文档说明
- **编码兼容**：所有新代码 `open()` 显式指定 `encoding='utf-8'`
- **脚本兼容**：所有修改保持 `python mover.py` 直接运行的能力
- **Steam 多库**：通过 `libraryfolders.vdf` 支持游戏装在任意磁盘

## 验证方案

1. `python mover.py pc_to_mobile` — 脚本模式仍正常工作
2. `python build.py` — 构建成功，输出到 `dist/SlayTheSpire-SaveTransfer/`
3. 删除 `config.ini` 后运行 `mover.exe pc_to_mobile` — 触发首次引导（扫描游戏目录 + 检测时区 + 生成配置）
4. 删除 `platform-tools/` 后运行 — 触发 ADB 自动下载
5. 完整 `pc_to_mobile` 和 `mobile_to_pc` 双向传输测试
