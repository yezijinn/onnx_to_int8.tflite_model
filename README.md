# onnx_to_int8.tflite

> ONNX → **安卓 NNAPI / NPU 可用的全 INT8 双输出 TFLite** 转换工具，开箱即用。

将 ONNX 模型一键转为安卓端可用的 INT8 量化模型（`*.tflite`），转换后体积更小、运行更快，适配低精度推理。

**下载即用**：Release 中的 zip 内含完整运行环境，解压即可运行，无需安装 Python、无需安装任何依赖、不污染系统环境变量。

---

## 快速开始

1. 打开 [Releases](https://github.com/yezijinn/onnx_to_int8.tflite_model/releases)，下载 `onnx_to_int8.tflite-v1.0.0.zip`（约 448MB，含完整内嵌 Python 3.13 运行时）
2. 解压 → 双击 `onnx_to_int8.exe`
3. 依次选择：**ONNX 模型文件** / **校准图片目录**（50~200 张真实场景图）/ **输出目录**
4. 点「开始转换」→ 日志实时显示进度 → 产物 `{模型名}_int8.tflite` 出现在输出目录

新手请先阅读压缩包内的《使用说明.txt》。

## 命令行（可选）

```bat
runtime\python\python.exe convert_to_int8.py --input model.onnx --out-dir out --calib images
```

| 参数 | 说明 |
|------|------|
| `--input`  | 输入 `.onnx`（必填） |
| `--out-dir`| 输出目录（必填） |
| `--calib`  | 校准图片目录（必填；真实场景图 50~200 张最佳） |

## 特性

- **全 INT8 量化**：输入 / cls / reg 全部 INT8，无 float32 张量，适配安卓端低精度推理
- **双输出独立量化**：cls 与 reg 各自独立 scale（cls scale ≈ 1/256），置信度保精度
- **YOLO 检测头还原**：自动还原 end2end 模型检测头；xyxy 框布局自动转 xywh（避免安卓端框放大数倍）
- **中文路径兼容**：中间产物自动走系统临时 ASCII 目录，路径含中文也能跑
- **实时日志**：GUI 用日志框逐行显示转换进度，窗口不卡死

## 产物格式

```
输入 [1,H,W,3] INT8  s=0.00392157 zp=-128
cls  [1,Nc,A] INT8  s=0.00390625 zp=-128   ← 精细量化（≈1/256）
reg  [1,4,A]  INT8  s≈1.7~1.9（独立 scale）
全图无 float32 张量（仅 INT8/INT32）
```

## 常见问题

- **转换失败 / 精度差**：校准图片太少，请准备 50~200 张与模型训练场景一致的图片
- **被杀毒软件拦截**：程序为本地运行工具，添加信任后重试
- **更多问题**：见压缩包内《使用说明.txt》

## 开发文档

`docs/` 目录提供完整开发文档（架构、数据流、API 协议、UI 规范等），入口见 `AGENTS.md`。

## 参考

- 上游项目：[microsoft/onnxruntime](https://github.com/microsoft/onnxruntime)
- 转换核心：[onnx2tf](https://github.com/PINTO0309/onnx2tf)
