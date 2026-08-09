# D01 · Overview — onnx_to_int8.tflite

> 面向 AI agent 的项目总览。本文件定义愿景、用户、优先级、roadmap、度量与风险。
> 所有下游设计（D02~D09、UI 文档）必须与本文件一致。

## 1. 产品愿景（Vision）

把任意 **YOLO 系列 ONNX 模型**一键转换为**安卓 NPU 可用的全 INT8 双输出 TFLite**
（`{模型名}_int8.tflite`），全程零 float32 张量，确保量化精度与安卓端推理正确。

**核心差异点（vs 普通 INT8 转换）**：

- **双输出独立量化**：reg `[1,4,A]` 与 cls `[1,Nc,A]` 各自独立量化，
  cls 的 scale ≈ 1/256（精细量化），避免联合量化把 0~1 的置信度范围压扁成 1 级。
- **end2end 成品自动还原**：含 NMS/TopK 的成品模型，自动从图内还原原始检测头
  `[B,4+Nc,A]`，再转 xywh 标准布局（避免安卓端框放大数倍）。
- **开箱即用**：内嵌完整 Python 3.13 运行时 + 全部依赖（约 1.7GB），
  用户**无需安装 Python、无需 pip、不污染系统环境**，拿目录即跑。

## 2. 目标用户（Users & Personas）

| 角色 | 场景 | 需求 |
|------|------|------|
| 模型部署工程师（主用户） | 有 YOLO ONNX，要上安卓 NPU | 稳定、可重复、零环境配置的转换 |
| 终端使用者（GUI） | 双击 exe 选文件即转换 | 简单、有进度/日志、不卡死 |
| CI / 批量脚本（CLI） | 流水线里批量转模型 | 命令行可控、stdout 可解析、退出码明确 |

## 3. 交付物（Deliverables）

| 产物 | 类型 | 说明 |
|------|------|------|
| `onnx_to_int8.exe` | 原生 Win32 GUI | 深色主题单文件，双击即用 |
| `convert_to_int8.py` | Python CLI 核心 | 转换逻辑唯一实现，GUI 通过子进程调用 |
| `runtime/python/` | 内嵌 Python 3.13.14 | 解释器 + 77 包（onnx2tf/tensorflow/onnx…） |
| `{stem}_int8.tflite` | 转换产物 | 全 INT8 双输出，供安卓 NPU 推理 |

## 4. 优先级（Priorities）

1. **P0 — 转换正确性**：零 float32、双输出独立量化、end2end 还原。
2. **P0 — 开箱即用**：内嵌运行时相对路径定位，任意 Windows x64 可跑。
3. **P1 — GUI 体验**：深色主题、日志实时、转换期间不卡死、按钮状态正确。
4. **P1 — 目录干净**：不写 `__pycache__`/`.pyc`、中文路径兼容。
5. **P2 — 分发体积**：runtime 已精简至 ~1.7GB（tensorflow/include 等已删）。

## 5. 路线图（Roadmap）

| 阶段 | 内容 | 状态 |
|------|------|------|
| Phase 1 | 转换核心（end2end 还原 + 双输出拆分 + INT8 量化） | ✅ 完成 |
| Phase 2 | 原生 GUI（管道读 stdout、后台线程、深色主题） | ✅ 完成 |
| Phase 3 | 内嵌运行时精简 + 相对路径定位 | ✅ 完成 |
| Phase 4 | UI 重构（去进度条、输出目录按钮禁用逻辑） | ✅ 完成 |
| Phase 5 | 跨平台 / macOS 支持 | ⬜ 未排期 |

## 6. 度量指标（Metrics）

| 指标 | 目标 | 验证方式 |
|------|------|----------|
| 产物全 INT8 | 无 float32 张量（仅 INT8/INT32） | 解析 tflite 张量类型 |
| cls scale | ≈ 1/256 | 读取量化参数 |
| 端到端跑通 | toy YOLO `[1,9,1024]` → 成功 | CI/手动构造模型验证 |
| 中文路径 | 输入/输出含中文可成功 | 含中文目录测试 |

## 7. 风险（Risks）

| 风险 | 影响 | 缓解 |
|------|------|------|
| 内嵌 tensorflow 体积（~1.7GB） | 分发大 | 已删 include/tests，硬下限 1.4GB |
| `tf.lite.Interpreter` 在 TF 2.20+ deprecated | 校验警告 | 保留至迁移 ai_edge_litert |
| 中文路径下 TF file_io 失败 | 转换中断 | 中间产物走系统临时 ASCII 目录 |
| MSVC 不在 PATH | 编译失败 | 用 clang+Windows SDK 或绝对路径 cl |
| 误删 tensorflow/tools / test 目录 | import 失败 | 仅从同版本 wheel 恢复（见 D05） |

## 8. 文档索引（Doc Index）

| ID | 文件 | 内容 |
|----|------|------|
| D01 | `docs/01-overview.md` | 本文件 |
| D02 | `docs/02-feature-design.md` | 功能模块 + 数据模型接口 |
| D03 | `docs/03-architecture.md` | 技术栈、目录树、数据流 |
| D04 | `docs/04-api-design.md` | 子进程调用、stdout JSON 协议、控件 ID |
| D05 | `docs/05-dev-guide.md` | 构建、配置、运行、部署、分发 |
| D06 | `docs/06-task-list.md` | 原子任务清单 |
| D07 | `docs/07-error-handling.md` | 错误码、恢复策略 |
| D08 | `docs/08-state-persistence.md` | 状态/路径权威、运行时定位 |
| U01 | `docs/ui-01-design-system.md` | 深色主题设计令牌 |
| U02 | `docs/ui-02-main-window-visual-spec.md` | 主窗口像素级规格 |
| U03 | `docs/ui-03-interaction-flow.md` | 交互流、状态机 |
| O01 | `docs/o01-security-and-perf.md` | 安全与性能 |
| D09 | `AGENTS.md` | AI agent 入口 |

## 9. 相关链接

- 参考项目：<https://github.com/microsoft/onnxruntime>
- 作者：<https://github.com/yezijinn/>
