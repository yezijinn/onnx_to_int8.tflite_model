# D06 · Task List — onnx_to_int8.tflite

> 原子任务清单。每个任务引用具体文档 + 章节，供 AI agent 按需加载上下文。
> Phase 1~4 已完成；Phase 5 为未来扩展。

## Phase 1 · 转换核心（✅ 已完成）

| TID | 任务 | 依赖 | 验收 | 文件 |
|-----|------|------|------|------|
| T101 | 实现 end2end 还原 `recover_head_from_end2end` | — | `[B,N,6]`→`[B,4+Nc,A]`+xywh | convert_to_int8.py / D02 F4 |
| T102 | 实现双输出源头拆分 `split_merged_output` | T101 | reg/cls 独立量化 | convert_to_int8.py / D02 F5 |
| T103 | onnx2tf 转 saved_model | T102 | 生成 saved_model.pb | D03 §4 |
| T104 | TFLiteConverter 全 INT8 量化 | T103 | 产物无 float32 | D02 DM4 |
| T105 | 推理校验（tf.lite.Interpreter 空输入） | T104 | 校验通过 | D02 F9 |

## Phase 2 · GUI（✅ 已完成）

| TID | 任务 | 依赖 | 验收 | 文件 |
|-----|------|------|------|------|
| T201 | 窗口/控件布局（深色主题） | — | 10 控件就位 | gui/main.cpp / U02 |
| T202 | 后台线程 + 管道读 stdout | T201 | 窗口不卡死 | D03 §5 / D04 §1 |
| T203 | JSON 行解析 handle_line | T202 | log/err 正确显示 | D04 §2 |
| T204 | 开始转换按钮状态机 | T202 | 点击→「正在转换......」→恢复 | D03 §5 |
| T205 | 输出目录按钮禁用逻辑 | T201 | 无效目录禁用，点击打开资源管理器 | D02 DM5 |

## Phase 3 · 内嵌运行时精简（✅ 已完成）

| TID | 任务 | 依赖 | 验收 | 文件 |
|-----|------|------|------|------|
| T301 | 删 tensorflow/include 等无用文件 | — | 体积 1.949→1.721GB | D05 §12 |
| T302 | 恢复误删 tensorflow/tools/test | T301 | import tensorflow 通过 | D05 §12 红线 |
| T303 | _pth 相对路径定位验证 | T301 | 任意 x64 可跑 | D08 |

## Phase 4 · UI 重构（✅ 已完成）

| TID | 任务 | 依赖 | 验收 | 文件 |
|-----|------|------|------|------|
| T401 | 移除进度条 + 百分比 | T201 | 进度条消失 | gui/main.cpp |
| T402 | 开始转换放最左 + 点击改文案 | T401 | 最左，点击变「正在转换......」 | D02 DM5 |
| T403 | 输出目录按钮放最右 + 禁用态 | T401 | 最右，无效目录禁用 | D02 DM5 |
| T404 | 用户文案保持现状 | — | 不改变已定文案 | gui/main.cpp L520-561 |

## Phase 5 · 跨平台 / macOS（⬜ 未排期）

| TID | 任务 | 依赖 | 验收 | 文件 |
|-----|------|------|------|------|
| T501 | 抽象 GUI 层（Cocoa/Qt） | T201 | macOS 可运行 | 新建 |
| T502 | 内嵌 Python 改用 macOS 构建 | T301 | 同机制跨平台 | D08 |

> 跨 Phase 依赖：T2xx 依赖 T1xx 的 stdout 协议（D04 §2）；T4xx 依赖 T2xx 控件结构。
