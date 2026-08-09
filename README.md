# onnx_to_int8.tflite_model

> ONNX → **安卓 NNAPI / NPU 可用的全 INT8 双输出 TFLite** 转换工具（**开箱即用版**）。

本仓库只发布**可直接使用的分发包**，**不公开源码**；核心转换逻辑已做**加密混淆**（防反编译）。
同时公开 `docs/` 开发文档，供 AI Agent / 开发者自行复刻同类工具。

---

## 快速开始（下载即用）

1. 打开 [Releases](https://github.com/yezijinn/onnx_to_int8.tflite_model/releases)
2. 下载 `onnx_to_int8.tflite-v1.0.0.zip`（约 448MB，含完整内嵌 Python 3.13 运行时）
3. 解压 → 双击 `onnx_to_int8.exe`
4. 依次选择：ONNX 模型 / 校准图片目录（50~200 张真实场景图）/ 输出目录
5. 点「开始转换」→ 日志实时显示进度 → 产物 `{模型名}_int8.tflite` 在输出目录

**无需安装 Python、无需 pip install、不污染系统环境变量**，任意 Windows x64（Win10/11）可用。
新手请先阅读压缩包内的《使用说明.txt》。

## 命令行（可选）

```bat
runtime\python\python.exe convert_to_int8.py --input model.onnx --out-dir out --calib images
```

## 特性

- 全 INT8 量化：输入 / cls / reg 全部 INT8，无 float32 张量
- cls/reg 独立量化：cls scale ≈ 1/256，置信度保精度
- 自动还原 YOLO 检测头、xyxy→xywh 转换
- 中文路径兼容（中间产物自动走 ASCII 临时目录）
- 原生 Win32 GUI，零外部依赖，窗口不卡死

## 产物格式

```
输入 [1,H,W,3] INT8  s=0.00392157 zp=-128
cls  [1,Nc,A] INT8  s=0.00390625 zp=-128   ← 精细量化（≈1/256）
reg  [1,4,A]  INT8  s≈1.7~1.9（独立 scale）
全图无 float32 张量（仅 INT8/INT32）
```

## 面向 AI Agent：用 docs 复刻一个

本仓库 `docs/` 公开了完整开发文档（架构、数据流、API 协议、错误处理、UI 规范、
安全/性能红线），配合 `AGENTS.md` 入口，AI Agent 可据此从零复刻功能相同的工具：

- 文档入口：`AGENTS.md`
- 目录：`docs/`（D01~D08、U01~U03、O01）

## 常见问题

- 校准图片张数不足 → 转换失败或精度差，请准备 50~200 张
- 被杀毒软件拦截 → 添加信任后重试（本地运行工具）
- 更多问题见压缩包内《使用说明.txt》

## 参考

- 上游项目：[microsoft/onnxruntime](https://github.com/microsoft/onnxruntime)
- 转换核心：[onnx2tf](https://github.com/PINTO0309/onnx2tf)
