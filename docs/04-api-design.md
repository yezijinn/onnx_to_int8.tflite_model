# D04 · API Design — onnx_to_int8.tflite

> 本项目的“接口”即：①子进程调用约定 ②stdout JSON 行协议 ③GUI 控件事件接口。
> 无 HTTP/无 IPC socket——唯一通道是 stdout 管道 + 控件消息。

## 1. 子进程调用接口（Process Invocation）

**调用方**：`gui/main.cpp` `worker_proc`
**被调**：`runtime\python\python.exe convert_to_int8.py`

```text
"{python_exe}" convert_to_int8.py --input "{onnx}" --out-dir "{out}" --calib "{calib}"
```

- 工作目录 = exe 所在目录（脚本 `convert_to_int8.py` 同目录）。
- `CREATE_NO_WINDOW`：无控制台窗口弹出。
- 继承 `PYTHONDONTWRITEBYTECODE=1` 环境变量。

**退出码语义**（GUI `WM_APP+2` 读取）：

| code | 含义 | GUI 行为 |
|------|------|----------|
| 0 | 成功 | 完成提示 + 产物路径 + 作者信息 |
| 1 | 输入/校准参数错误 | `[失败，退出码 1]` |
| 2 | 缺 Python 模块 | `[失败，退出码 2]` |
| 3 | onnx2tf/TFLite 转换失败 | `[失败，退出码 3]` |

## 2. stdout JSON 行协议（DM1 详版）

每行一个 JSON 对象（UTF-8，`\n` 结尾）。C++ `handle_line` 解析。

### 2.1 log

```json
{ "p": "log", "t": "正在转换，请耐心等待3-20分钟..." }
```
→ GUI：`append_log(t)`。C++ 解析 `\"t\"` 抽取引号内字符串。

### 2.2 progress（GUI 当前忽略，保留字段）

```json
{ "p": "progress", "v": 0.45 }
```
→ GUI：`handle_line` 检测到 `\"p\":\"progress\"` 不处理（进度条已移除）。
→ **修改警告**：若未来复用，C++ 需先识别 `\"t\"`/`\"done\"`/`\"err\"` 之外的分支。

### 2.3 err

```json
{ "p": "err", "msg": "onnx2tf 转换失败" }
```
→ GUI：显示 `[错误] msg`，随后追加 `[失败]`。
→ C++ 解析 `\"msg\"` 引号内字符串。

### 2.4 done（GUI 当前忽略）

```json
{ "p": "done", "out": "D:\\out\\model_int8.tflite" }
```
→ GUI：`handle_line` 检测到 `\"done\"` 直接 return（完成统一由退出码处理）。

**解析顺序**（C++ `handle_line`）：先 `\"t\"` → `\"err\"` → `\"done\"`。
> ⚠️ 注意：C++ 用 `strstr(line,"\"t\"")` 判定 log——任何消息含 `"t"` 子串（如 `"out"` 含 t）也会被当 log！
> 因此 Python 端 **不要** 在 `log` 之外发送含 `"t"` 键的消息（当前 `done` 用 `"out"` 规避此坑）。
> 这是历史遗留解析缺陷，**改动解析逻辑需同步改 Python 键名约定**。

## 3. GUI 控件事件接口（WM_COMMAND）

| 消息来源 | LOWORD(wp) | 处理 |
|----------|------------|------|
| BTN 浏览 ONNX | `IDC_BTN_ONNX` (1005) | `browse(...,true)` |
| BTN 浏览校准 | `IDC_BTN_CALIB` (1006) | `browse(...,false)` |
| BTN 浏览输出 | `IDC_BTN_OUT` (1007) | `browse(...,false)` + `update_open_btn()` |
| BTN 开始转换 | `IDC_BTN_START` (1008) | `start_convert()` |
| BTN 输出目录 | `IDC_BTN_OPENOUT` (1009) | `open_out_dir()` |
| EDIT 输出文本变化 | `IDC_EDIT_OUT` (1003) + `EN_CHANGE` | `update_open_btn()` |

## 4. 自定义消息（User Messages）

| 消息 | wParam | lParam | 含义 |
|------|--------|--------|------|
| `WM_APP+1` | 0 | `char*` (堆分配，用完 `LocalFree`) | 一行 stdout 日志 |
| `WM_APP+2` | `DWORD` 退出码 | 0 | 子进程结束 |

## 5. Python 内部函数接口（供 AI 修改参考）

`convert_to_int8.py` 公开函数（含签名）：

```python
def convert_to_int8(in_path: str, out_dir: str, calib_dir: str) -> str
    # 返回产物绝对路径

def get_onnx_io(onnx_path: str) -> tuple[list, list]
    # (inputs, outputs)，每项 (name, dims)

def has_nms_ops(onnx_path: str) -> bool
def recover_head_from_end2end(onnx_path: str, out_path: str) -> bool
def split_merged_output(onnx_path: str, out_path: str) -> bool
def prepare_calib(calib_dir: str, input_dims: list, out_np_dir: str, max_samples: int = 200) -> list[str]
def ascii_workdir(stem: str) -> str
```

日志函数：`log(text)` / `progress(v)` / `done(out_path)` / `error(msg, code)`（见 DM1）。

## 6. 接口注册一致性（Registration Parity）

- `wm 命令` 的控件 ID 必须与 `enum` 定义一致（见 D02 表 DM5）。
- 命令行参数名 `--input/--out-dir/--calib` 在 `main.cpp` 构造与 `convert_to_int8.py` argparse 必须完全一致。
- stdout 键名 `p/t/v/msg/out` 在 C++ `handle_line` 与 Python `log/error/done` 必须一致。
