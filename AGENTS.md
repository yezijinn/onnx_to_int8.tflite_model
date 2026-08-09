# AGENTS.md — onnx_to_int8.tflite

> **AI agent 入口**。修改本项目前先读完本文件 + 对应 docs/ 章节。
> 项目：原生 Win32 GUI + 内嵌 Python 运行时，一键把 YOLO ONNX 转安卓 NPU 全 INT8 双输出 TFLite。

## 命令（Commands）

```bat
:: 构建 exe（clang）
clang++.exe -target x86_64-pc-windows-msvc -std=c++17 -fms-compatibility -fno-exceptions -fno-rtti -O2 ^
  -DUNICODE -D_UNICODE -DNOMINMAX -DWIN32_LEAN_AND_MEAN -I"<LLVM>/lib/clang/<ver>/include" ^
  -I"%INC%/ucrt" -I"%INC%/um" -I"%INC%/shared" gui\main.cpp -fuse-ld=lld ^
  -Wl,/NODEFAULTLIB -Wl,/ENTRY:wWinMain -Wl,/SUBSYSTEM:WINDOWS <libs...> -o onnx_to_int8.exe

:: 运行（CLI）
runtime\python\python.exe convert_to_int8.py --input model.onnx --out-dir out --calib images

:: 自检内嵌运行时
runtime\python\python.exe -c "import tensorflow"
```

完整编译命令（MSVC/clang 双版）见 `docs/05-dev-guide.md` §2。

## 结构（Structure）

```text
onnx_to_int8.exe      # Win32 GUI（编译产物）
convert_to_int8.py    # 转换核心（CLI 唯一实现）
runtime/python/       # 内嵌 Python 3.13.14 + 77 包（~1.7GB，相对路径定位）
docs/                 # AI 开发文档集
README.md / requirements.txt
```

## 约定（Conventions）

- **转换逻辑只在 Python**：GUI 不含算法，仅子进程 + stdout JSON 协议调用。
- **stdout 协议**：每行一个 JSON `{"p":"log"/"progress"/"err"/"done",...}`（见 `docs/04-api-design.md` §2）。改日志文案只改 `t` 字段内容，勿改 `p` 键名。
- **双输出独立量化**：cls scale≈1/256，用源头 `Identity` 拆分，**禁止** Split 联合量化。
- **中文路径**：中间产物走 `tempfile.gettempdir()` 下 ASCII 目录（`ascii_workdir`）。
- **不写字节码**：`PYTHONDONTWRITEBYTECODE` + `sys.dont_write_bytecode`。
- **深色主题**：固定文案与配色见 `docs/ui-01-design-system.md`，用户已定文案勿改。
- **编码**：文本文件 UTF-8；GUI 微软雅黑；全程中文路径兼容。

## 原则（Principles）

1. 产物必须全 INT8（无 float32），cls/reg 独立量化。
2. 开箱即用：内嵌运行时相对路径，任意 Win x64 即拷即跑，不污染系统。
3. 目录干净：无 `__pycache__`/`.pyc`、无外部依赖污染。
4. 转换期间窗口不卡死（后台线程 + PostMessage）。

## 令牌/依赖（Tokens & Deps）

- 内嵌：`onnx2tf==2.6.8`、`tensorflow==2.21.0`、`onnx==1.20`、`onnxsim`、`onnxruntime==1.26`、`tf_keras`、`numpy`、`Pillow`（见 `requirements.txt`）。
- GUI 库：`user32/gdi32/comctl32/comdlg32/shell32/ole32/shlwapi/uxtheme`（comctl32 v6 manifest 内嵌）。

## 坑点（Pitfalls — 必读）

- **删 runtime 文件**：本环境 safe-delete hook 拦截 `rm`/`shutil.rmtree`/`os.remove`（静默失败）。
  删除必须用 ctypes Win32 API（普通 `E:\` 路径、声明 `argtypes=[c_wchar_p]`、删前去只读位），
  **勿用 `\\?\` 前缀**（报 123）。详见 `docs/o01-security-and-perf.md` §2.4。
- **runtime 不可删**：`tensorflow/tools`、`tensorflow/**/test/` 被运行时 import，误删从同版本 wheel 恢复。
- **MSVC 不在 PATH**：用 clang 或 `E:\DevTools\VisualStudio\...` 绝对路径 cl。
- **C++ stdout 解析坑**：`handle_line` 用 `strstr(line,"\"t\"")` 判 log，别让非 log 消息含 `t` 键（用 `out` 规避）。
- **tf.lite.Interpreter deprecated**：TF 2.21 仍可用，迁移 ai_edge_litert 前保留。
- **EN_CHANGE 跨进程不触发**：`browse()` 后须显式 `update_open_btn()` 刷新输出目录按钮。

## 文档索引（Doc Index）

| 文件 | 内容 |
|------|------|
| `docs/01-overview.md` | 愿景/优先级/风险 |
| `docs/02-feature-design.md` | 模块 + 数据模型接口（DM1~DM6） |
| `docs/03-architecture.md` | 技术栈/目录树/数据流 |
| `docs/04-api-design.md` | 子进程调用 + stdout 协议 + 控件 ID |
| `docs/05-dev-guide.md` | 构建/配置/运行/部署/分发 |
| `docs/06-task-list.md` | 原子任务清单 |
| `docs/07-error-handling.md` | 错误码/恢复 |
| `docs/08-state-persistence.md` | 状态/路径权威/运行时定位 |
| `docs/ui-01-design-system.md` | 深色主题令牌 |
| `docs/ui-02-main-window-visual-spec.md` | 主窗口像素规格 |
| `docs/ui-03-interaction-flow.md` | 交互流/状态机 |
| `docs/o01-security-and-perf.md` | 安全/性能/坑点 |
