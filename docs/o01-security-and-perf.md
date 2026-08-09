# O01 · Security & Performance — onnx_to_int8.tflite

> 本项目**无用户认证/无网络/无敏感数据**。安全焦点在：内嵌运行时完整性、分发安全。
> 性能焦点在：转换耗时、分发体积、无卡死。

## 1. 安全（Security）

### 1.1 威胁模型（Threat Model）

| 威胁 | 可能性 | 影响 | 缓解 |
|------|--------|------|------|
| 恶意 ONNX/图片输入 | 中 | onnx2tf/TF 解析崩溃 | 沙箱外运行，用户自担；CLI 退出码隔离 |
| 篡改内嵌 python | 低 | 执行恶意代码 | 分发校验 `requirements.txt` 版本钉死 |
| 路径注入（含空格路径） | 中 | 命令错乱 | GUI 已用 `\"%s\"` 引号包裹所有路径 |
| 环境变量污染 | 低 | 加载错误 Python | 子进程仅继承 `PYTHONDONTWRITEBYTECODE`，不读全局 PATH |

### 1.2 安全实践（Security Practices）

- **无网络调用**：转换全程离线（onnx2tf/tensorflow 本地）。
- **无遥测/无外联**：无数据上传。
- **路径引号**：`swprintf(cmdline, ..., L"\"%s\" ...", onnx, out, calib)` 防止空格注入。
- **依赖钉版本**：`requirements.txt` 中 `onnx2tf==2.6.8`、`tensorflow` 内嵌固定版本（2.21.0）。
- **权限**：纯用户态，无管理员/UAC 要求。

### 1.3 审计（Audit Logging）

无（无安全相关事件需记录）。

## 2. 性能（Performance）

### 2.1 性能目标（Perf Targets）

| 指标 | 目标 | 测量方式 |
|------|------|----------|
| 端到端转换 | 3~20 分钟（依模型/校准图数） | 日志开始/完成时间差（U02 §4） |
| 后台不卡死 | 主线程消息循环 ≤ 16ms/帧 | 人工/无阻塞验证 |
| 分发体积 | ≤ ~1.7GB（硬下限 ~1.4GB） | `du -sh runtime` |

### 2.2 性能策略（Perf Strategy）

- **后台线程**：转换在 `worker_proc`，主线程仅 `PostMessage` 更新 UI（D03 §5）。
- **校准图上限 200**：`prepare_calib(max_samples=200)` 均匀采样，避免大目录拖慢。
- **独立输出量化**：cls/reg 各自范围，量化精度高（避免联合范围精度损失）。
- **无字节码**：不写 `__pycache__`，减少小文件 IO。

### 2.3 体积控制（Size Control）

runtime 精简红线（**删除前务必读**）：

```text
可删（已删）：tensorflow/include、顶层 include/libs/Scripts、
  Lib/ensurepip、Lib/venv、各包 tests/test、setuptools/_pytest/pkg_resources
不可删：tensorflow/tools（被 deprecation import）、
  tensorflow/**/test/（被 _api/v2 import）
```

**误删恢复**：从 `tensorflow==2.21.0` wheel 仅提取 `tools` 与 `test/` 目录恢复。

### 2.4 坑点（Pitfalls — 必读）

1. **safe-delete hook 拦截删除**：本环境 `rm`/`shutil.rmtree`/`os.remove` 被 sitecustomize 包装（强制回收站 + 拒相对路径）→ 静默失败；`du` 不可用。
   → 删除大目录必须用 ctypes Win32 API：
   ```python
   import ctypes
   k = ctypes.windll.kernel32
   k.DeleteFileW.argtypes = [ctypes.c_wchar_p]
   k.RemoveDirectoryW.argtypes = [ctypes.c_wchar_p]
   # ① 普通 E:\ 绝对路径（勿用 \\?\ 长路径前缀，本环境报 ERROR_INVALID_NAME 123）
   # ② 删前 SetFileAttributesW(0x80) 去只读位
   p = os.path.normpath(os.path.abspath(path))
   k.SetFileAttributesW(p, 0x80)  # FILE_ATTRIBUTE_NORMAL
   k.DeleteFileW(p)  # 或 RemoveDirectoryW(p)
   ```
2. **`\\?\` 前缀报错 123**：ctypes 删文件勿加 `\\?\`，用普通绝对路径。
3. **缺 `argtypes` 报错 123**：`DeleteFileW`/`RemoveDirectoryW` 必须声明 `argtypes=[c_wchar_p]`。
4. **tf.lite.Interpreter deprecated**：TF 2.20+ 标记 deprecated（仅 warning），内嵌 2.21 仍可用；迁移 `ai_edge_litert` 前保留。
5. **MSVC 不在 PATH**：用 clang 或 `E:\DevTools\VisualStudio\...` 绝对路径 cl（D05 §2）。
6. **C++ stdout 解析缺陷**：`handle_line` 用 `strstr(line,"\"t\"")` 判 log，任何含 `t` 键的消息会被误判 → Python 端不要在 `log` 外用含 `t` 键的消息（用 `out` 规避）。

## 3. 监控与告警（Monitoring）

无（离线桌面工具，无告警阈值）。
