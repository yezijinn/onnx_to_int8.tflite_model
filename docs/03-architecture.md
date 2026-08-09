# D03 · Architecture — onnx_to_int8.tflite

## 1. 技术栈（Tech Stack）

| 层 | 技术 | 说明 |
|----|------|------|
| GUI | 原生 Win32 C++17（无 STL/无 MSVC CRT 可选） | 单文件 `gui/main.cpp`，`wWinMain` 入口 |
| 后端 | Python 3.13.14（内嵌，相对路径） | `convert_to_int8.py` 子进程 |
| 转换 | onnx2tf 2.6.8 + onnx 1.20 + tensorflow 2.21 + onnxsim + onnxruntime 1.26 | 见 `requirements.txt` |
| 视觉 | comctl32 v6（manifest 内嵌）+ uxtheme | 圆角按钮、深色主题 |
| 分发 | 目录复制（含 `runtime/`） | 约 1.7GB |

## 2. 目录树（Directory Tree）

```text
onnx_to_int8.tflite/
├── onnx_to_int8.exe          # 编译产物（Win32 GUI，manifest 已嵌入）
├── convert_to_int8.py        # 转换核心（CLI，唯一实现）
├── requirements.txt          # 依赖清单（分发用户无需装）
├── README.md
├── docs/                     # ★ AI 开发文档集（本目录）
│   ├── 01-overview.md
│   ├── 02-feature-design.md
│   ├── 03-architecture.md
│   ├── 04-api-design.md
│   ├── 05-dev-guide.md
│   ├── 06-task-list.md
│   ├── 07-error-handling.md
│   ├── 08-state-persistence.md
│   ├── ui-01-design-system.md
│   ├── ui-02-main-window-visual-spec.md
│   ├── ui-03-interaction-flow.md
│   └── o01-security-and-perf.md
└── runtime\python\           # ★ 内嵌 Python 3.13.14 + 77 包（约 1.7GB）
    ├── python.exe            # 可移植解释器
    ├── python313.dll
    ├── python313._pth        # 相对路径定位（见 D08）
    ├── vcruntime140.dll      # 自带 VC++ 运行库
    ├── vcruntime140_1.dll
    ├── DLLs\
    └── Lib\site-packages\    # onnx2tf / tensorflow / onnx / ...
```

## 3. 模块边界（Module Boundaries）

```text
┌──────────────────────────────────────────────────────────┐
│  onnx_to_int8.exe (Win32 GUI, main.cpp)                    │
│  ┌────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │ 窗口/控件   │  │ 后台线程      │  │ 管道读取 + JSON解析│  │
│  │ (wndproc)  │  │ (worker_proc) │  │ (handle_line)     │  │
│  └────────────┘  └──────┬───────┘  └───────────────────┘  │
│                         │ CreateProcessW                  │
└─────────────────────────┼────────────────────────────────┘
                          │  stdout pipe (UTF-8 JSON lines)
                          ▼
┌──────────────────────────────────────────────────────────┐
│  runtime\python\python.exe convert_to_int8.py (子进程)     │
│  ┌────────────────────────────────────────────────────┐   │
│  │ end2end 还原 → 双输出拆分 → onnx2tf → INT8 量化 → 校验│   │
│  └────────────────────────────────────────────────────┘   │
│  依赖: onnx2tf / tensorflow / onnx / onnxsim / onnxruntime │
└──────────────────────────────────────────────────────────┘
```

**边界规则**：
- GUI **不**直接 import Python；只通过子进程 + stdout 协议（DM1）。
- 转换逻辑 **全部** 在 `convert_to_int8.py`，GUI 不含转换算法。
- 内嵌运行时对 GUI 透明：GUI 仅用 `find_python()` 探测 `runtime\python\python.exe`。

## 4. 数据流（Data Flow）

```text
用户选 onnx+calib+out (GUI)
   │
   ▼
start_convert(): 校验非空 + 文件存在 → 禁按钮 → 计算 g_outFile
   │
   ▼
worker_proc: CreateProcessW(python.exe convert_to_int8.py --input --out-dir --calib)
   │
   ▼
python 逐阶段 emit JSON 到 stdout:
   {"p":"log",...} → GUI 显示
   {"p":"err",...} → GUI 显示 [错误] + [失败]
   │
   ▼
进程退出 → WM_APP+2(code):
   code==0 → 完成提示 + 产物路径 + 作者信息
   code≠0 → [失败，退出码 N]
```

转换内部数据流（M2，`convert_to_int8.py`）：

```text
onnx
 ├─ has_nms_ops? ──是──▶ recover_head_from_end2end → xyxy→xywh 双输出
 │                         (onnxsim 剪 NMS 悬空子图)
 └─▶ split_merged_output → 源头拆分 reg/cls（或 Split 回退）
      │
      ▼
 onnx2tf -i in -o work -b 1 → saved_model.pb
      │
      ▼
 prepare_calib: 图片→[1,H,W,3] float32 .npy（最多 200 张，均匀采样）
      │
      ▼
 TFLiteConverter: DEFAULT优化 + INT8 + 独立输出范围
   inference_input_type=int8, inference_output_type=int8
      │
      ▼
 {stem}_int8.tflite (全 INT8) → tf.lite.Interpreter 空输入校验
      │
      ▼
 shutil.rmtree(work) 清理中间目录
```

## 5. 跨进程并发模型

- GUI 主线程跑消息循环（`GetMessageW`），**不**阻塞。
- 转换在 `worker_proc` 后台线程，读管道逐行，每行 `PostMessageW(WM_APP+1)` 回主线程更新日志。
- 完成/失败通过 `WM_APP+2` 通知主线程。
- 用 `InterlockedCompareExchange(&g_running,1,0)` 防重入（按钮禁用双保险）。

## 6. 关键约束（Hard Constraints）

1. **不写字节码**：`wWinMain` 设 `PYTHONDONTWRITEBYTECODE=1`；脚本内 `sys.dont_write_bytecode=True`。子进程继承。
2. **中文路径**：TF `file_io` 不能写非 ASCII 路径 → 中间产物一律 `tempfile.gettempdir()` 下 ASCII 目录（`ascii_workdir`）。
3. **cls 独立量化**：必须源头拆分（Identity 独立张量），**禁止**用 Split（继承父联合范围压扁 cls）。
4. **全 INT8**：`inference_input_type/output_type=int8`，`supported_ops=[TFLITE_BUILTINS_INT8, SELECT_TF_OPS]`。
