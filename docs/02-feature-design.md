# D02 · Feature Design — onnx_to_int8.tflite

> **最关键文档**：定义所有功能模块与完整数据模型接口。AI agent 修改代码前必须先读此文件。
> 每个字段、类型、枚举都必须显式，禁止 `any`/占位。

## 模块一览（Module Map）

| 模块 ID | 名称 | 位置 | 职责 |
|---------|------|------|------|
| M1 | GUI 外壳 | `gui/main.cpp` | Win32 窗口、控件、后台线程、管道读 stdout |
| M2 | 转换核心 | `convert_to_int8.py` | ONNX 图手术 + onnx2tf + TFLiteConverter INT8 |
| M3 | 内嵌运行时 | `runtime/python/` | 解释器 + 依赖，相对路径定位 |
| M4 | 分发与构建 | `gui/main.cpp` 编译 + `runtime/python/` | exe 编译、目录打包分发 |

---

## 数据模型接口（Data Model Interfaces）

### DM1 · 子进程 stdout 消息协议（M1↔M2 IPC）

Python 脚本每行向 stdout 输出一个 JSON 对象，C++ 逐行解析。
**这是唯一进程间通信通道**，必须严格匹配字段。

```jsonc
// 日志行（C++ 显示正文 t）
{ "p": "log",  "t": "<UTF-8 文本>" }

// 进度（C++ 当前版本已忽略，但 Python 仍按原样发送，字段保留以防未来复用）
{ "p": "progress", "v": <float 0.0~1.0> }

// 错误（C++ 显示 [错误] msg，随后追加 [失败]）
{ "p": "err",  "msg": "<UTF-8 错误描述>" }

// 完成（C++ 当前忽略，完成统一由进程退出码 WM_APP+2 处理）
{ "p": "done", "out": "<产物绝对路径>" }
```

**字段约束**：
- `p`：枚举 `"log" | "progress" | "err" | "done"`，必填。
- `t`：string，UTF-8 编码（C++ 用 `MultiByteToWideChar(CP_UTF8)` 转宽字符）。
- `v`：float，范围 [0.0, 1.0]。
- `msg`：string，错误描述。
- `out`：string，产物绝对路径。

> ⚠️ 若修改日志内容，只在 `t` 内改文案，**不要**改 `p` 字段名或增删字段，否则 GUI 解析失败。

### DM2 · 命令行接口（M2 入参）

```text
python.exe convert_to_int8.py --input <onnx> --out-dir <dir> --calib <dir>
```

| 参数 | 类型 | 必填 | 约束 |
|------|------|------|------|
| `--input`  | path | 是 | 须为存在的 .onnx 文件 |
| `--out-dir`| path | 是 | 不存在则自动创建 |
| `--calib`  | path | 是 | 须为存在目录，含 ≥1 张图片 |

图片扩展名白名单：`".png", ".jpg", ".jpeg", ".bmp", ".webp"`（见 `prepare_calib`）。

### DM3 · ONNX 模型 I/O 契约（M2 内部）

**输入约定**：`[B, H, W, 3]` NHWC，B 强制设为 1（onnx2tf `-b 1`）。

**输出约定（转换目标）**：

```text
reg : [1, 4,   A]   // 检测框 xywh，float32 经独立 INT8 量化
cls : [1, Nc,  A]   // 类别置信度，float32 经独立 INT8 量化（scale≈1/256）
```

其中 `A` = anchor 数，`Nc` = 类别数。

**输入模型两种形态**（自动识别）：

| 形态 | 输出张量形状 | 处理方式 |
|------|--------------|----------|
| 原料型 | `[B, 4+Nc, A]` 单输出 | `split_merged_output` 源头拆分双输出 |
| end2end 成品型 | `[B, N, 6]`（含 NMS） | `recover_head_from_end2end` 还原检测头 + xyxy→xywh |

### DM4 · TFLite 张量契约（产物）

```text
输入 : [1, H, W, 3] INT8   scale=0.00392157 zp=-128
cls  : [1, Nc, A]   INT8   scale≈0.00390625 (1/256) zp=-128
reg  : [1, 4,  A]   INT8   scale≈1.7~1.9（独立）
全图 : 仅 INT8 / INT32，无 float32
```

### DM5 · GUI 控件模型（M1）

控件 ID 枚举（`gui/main.cpp`）：

```c
enum { IDC_EDIT_ONNX = 1001, IDC_EDIT_CALIB, IDC_EDIT_OUT, IDC_EDIT_PY,
       IDC_BTN_ONNX, IDC_BTN_CALIB, IDC_BTN_OUT, IDC_BTN_START,
       IDC_BTN_OPENOUT, IDC_LOG };
```

| ID | 控件 | 角色 | 行为 |
|----|------|------|------|
| 1001 | EDIT | ONNX 文件路径 | 只读浏览结果，可手填 |
| 1002 | EDIT | 校准图片目录 | 可手填 |
| 1003 | EDIT | 输出目录 | 可手填；EN_CHANGE → 刷新输出目录按钮 |
| 1004 | EDIT | Python 路径（只读） | 预填内嵌路径，不可改 |
| 1005 | BTN | 浏览 ONNX | 打开文件选择 |
| 1006 | BTN | 浏览校准目录 | 打开文件夹选择 |
| 1007 | BTN | 浏览输出目录 | 打开文件夹选择 |
| 1008 | BTN | **开始转换**（最左，加粗） | 点击→禁用自身+置「正在转换......」→ 启动后台线程 |
| 1009 | BTN | **输出目录**（最右） | 未设有效目录时禁用；点击→资源管理器打开/定位产物 |
| 1010 | EDIT | 日志框（多行只读） | 显示 stdout 日志 |

### DM6 · 运行态全局状态（M1）

```c
volatile LONG g_running;              // 0=空闲 1=转换中（Interlocked 保护）
wchar_t g_pythonExe[MAX_PATH*2];     // 实际 python 路径
wchar_t g_outFile[MAX_PATH*2];       // 预期产物绝对路径（用于输出目录按钮定位）
```

---

## 功能清单（Feature List）

| FID | 功能 | 模块 | 关联数据模型 | 验收 |
|-----|------|------|--------------|------|
| F1 | GUI 选文件/目录 | M1 | DM5 | 三个编辑框可浏览/手填 |
| F2 | 一键转换（后台线程） | M1 | DM5,DM1 | 点开始→日志实时、窗口不卡死 |
| F3 | 输出目录按钮禁用逻辑 | M1 | DM5 | 无效目录禁用，有效后启用，点击打开资源管理器 |
| F4 | end2end 自动还原 | M2 | DM3 | `[B,N,6]` → `[B,4+Nc,A]` + xywh |
| F5 | 双输出源头拆分 | M2 | DM3 | 单输出 → reg/cls 独立（独立量化） |
| F6 | onnx2tf 转 saved_model | M2 | DM2 | 生成 saved_model.pb |
| F7 | 全 INT8 量化 | M2 | DM4 | 产物无 float32 |
| F8 | 中文路径兼容 | M2 | — | 中间产物走 ASCII 临时目录 |
| F9 | 推理校验 | M2 | DM4 | tf.lite.Interpreter 跑通空输入 |
| F10 | 内嵌运行时相对定位 | M3 | — | 任意 Win x64 直接运行 |
| F11 | 目录干净（无字节码） | M1+M2 | — | 无 __pycache__/.pyc |

> 每个 FID 的原子任务见 `docs/06-task-list.md`。
