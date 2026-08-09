# D08 · State Persistence — onnx_to_int8.tflite

> 本项目**无持久化状态**（无配置、无数据库、无用户数据保存）。
> 本文件规定各数据类型的权威存储位置与内嵌运行时定位机制。

## 1. 数据类型 → 存储权威（Storage Authority）

| 数据类型 | 权威位置（Authority） | 持久化？ | 说明 |
|----------|----------------------|----------|------|
| 用户输入（onnx/calib/out 路径） | GUI 编辑框（内存） | 否 | 关闭即丢，无记忆 |
| 转换中间产物（saved_model/.npy） | `tempfile.gettempdir()/onnx2tf_<md5>` | 否 | 转换完 `rmtree` 清理 |
| 最终产物 `{stem}_int8.tflite` | 用户指定 `--out-dir` | 是（用户文件） | 落盘即用户资产 |
| 内嵌 Python 运行定位 | `runtime/python/python313._pth` | 是（静态） | 相对路径，只读 |
| GUI 运行态（g_running 等） | 进程内存 | 否 | `volatile LONG` |

## 2. Python 运行时定位（Runtime Resolution）

**权威机制**：`runtime/python/python313._pth`

```text
python313.zip
.
DLLs
Lib
Lib/site-packages
import site
```

- 解释器从 exe 同目录读 `_pth`，将 `Lib/site-packages` 加入 `sys.path`。
- **不读注册表、不读 PATH、不写全局环境变量** → 纯目录内加载。
- GUI 用 `find_python()` 探测顺序：
  1. `runtime\python\python.exe`（相对 exe 目录）
  2. `.venv\Scripts\python.exe`
  3. 回退 `python`（系统 PATH）

## 3. 中间产物目录（Intermediate Work Dir）

```python
def ascii_workdir(stem: str) -> str:
    safe = "onnx2tf_" + hashlib.md5(stem.encode("utf-8")).hexdigest()[:12]
    return os.path.join(tempfile.gettempdir(), safe)
```

- **权威**：系统临时目录（`%TEMP%`/`tempfile.gettempdir()`）。
- **理由**：TF `file_io` 无法写非 ASCII 路径 → 中文路径下必须用 ASCII 目录；同时不污染输出目录。
- **清理**：`convert_to_int8` 末尾 `shutil.rmtree(work, ignore_errors=True)`。

## 4. 迁移策略（Migration）

无版本化状态，无需迁移。若未来引入配置持久化，权威位置应为
`<exe_dir>/.config/onnx2int8.json`（目录内，不写用户目录）。

## 5. 冲突解决（Conflict Resolution）

无多进程共享状态，无冲突。GUI 单进程单转换（`g_running` 互斥）。
