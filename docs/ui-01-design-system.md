# U01 · UI Design System — onnx_to_int8.tflite

> 深色专业主题。所有颜色、字体、控件尺寸均为**精确值**（来自 `gui/main.cpp`）。
> 非权威文档可用实现变量名引用，权威令牌在此定义。

## 1. 颜色令牌（Color Tokens）

| Token | Hex | RGB | 用途 |
|-------|-----|-----|------|
| `--color-bg-window` | `#1E1E1E` | (30,30,30) | 窗口/说明文字背景（WM_CTLCOLORSTATIC/BTN 返回画刷） |
| `--color-bg-log` | `#252526` | (37,37,38) | 日志框背景 |
| `--color-bg-input` | `#FFFFFF` | (255,255,255) | 输入框背景（白底黑字，高对比） |
| `--color-text-label` | `#C8C8C8` | (200,200,200) | 说明文字（浅灰） |
| `--color-text-log` | `#D4D4D4` | (212,212,212) | 日志正文 |
| `--color-text-input` | `#1E1E1E` | (30,30,30) | 输入框文字（深） |
| `--color-accent` | 系统 comctl32 v6 默认 | — | 按钮（圆角，Win10 默认蓝/灰） |

> 说明文字用 `SetTextColor(RGB(0xC8,0xC8,0xC8))` + `SetBkColor(#1E1E1E)`；
> 输入框白底；日志框深色终端风（已 `SetWindowTheme(hLog,L"",L"")` 关主题以让自定义底生效）。

## 2. 字体（Typography）

| 角色 | 字体 | 大小 | 字重 |
|------|------|------|------|
| 基础（标签/输入框/日志） | Microsoft YaHei UI | 15px（`-15`） | 常规 |
| 标题/主按钮 | Microsoft YaHei UI | 18px（`-18`） | **加粗**（`FW_BOLD`） |

字符集：`DEFAULT_CHARSET`，渲染：`CLEARTYPE_QUALITY`。

## 3. 间距与布局栅格（Spacing & Layout）

- 窗口：`720 × 470`，起始位置 `(100, 100)`，样式 `WS_OVERLAPPEDWINDOW`。
- 左边距：`x = 14`；右侧按钮区 `x = 584`（宽 122）。
- 行高节奏：标签 `y` 后 20px 放置编辑框（高 24）；区块间距约 28px。
- 主按钮行：`y = 231`，高 38。
- 日志框：`x=14, y=282, w=692, h=150`（约 8 行高）。

## 4. 控件尺寸规范（Component Specs）

| 控件 | 宽 × 高 | 备注 |
|------|---------|------|
| 编辑框（路径类） | 560 × 24 | 左对齐，ES_AUTOHSCROLL |
| 浏览按钮 | 122 × 24 | 右侧 `x=584` |
| 主按钮（开始转换） | 220 × 38 | 加粗，最左 |
| 输出目录按钮 | 120 × 38 | 加粗，最右 `x=586` |
| Python 路径编辑框 | 692 × 24 | ES_READONLY |
| 日志框 | 692 × 150 | 多行只读 + 垂直滚动 |

## 5. 主题系统（Theme System）

- **仅深色**：无亮色模式（当前版本）。
- comctl32 v6 由 `#pragma comment(linker,"/manifestdependency:...")` 启用 → 圆角按钮。
- 无动画、无阴影（原生 Win32）。

## 6. 无障碍（Accessibility）

- 按钮均有文案标签（中文）。
- 说明文字为 `SS_LEFT` 自动换行。
- 日志框只读 + 滚动，避免内容溢出。
- 当前未做高对比模式/屏幕阅读器增强（未来可加）。

## 7. 通知系统（Notification）

无 toast/弹窗。所有反馈走日志框（DM5 ID 1010）：
- 启动首行：作者信息 `本程序作者：https://github.com/yezijinn/`
- 转换开始：`正在转换，请耐心等待3-20分钟，不要强制关闭本程序。开始时间是：<时间>`
- 转换完成：`转换工作已经完成。完成时间是：<时间>` + `新文件是：<路径>` + 作者信息
- 错误/提示：`[错误]...` / `[提示]...` / `[失败]`

## 8. 设计令牌 → 实现变量映射（Token → Implementation）

| 设计令牌 | 实现位置（main.cpp） |
|----------|----------------------|
| `--color-bg-window` | `g_brBg = CreateSolidBrush(RGB(0x1E,0x1E,0x1E))` |
| `--color-bg-log` | `g_brLog = CreateSolidBrush(RGB(0x25,0x25,0x26))` |
| `--color-bg-input` | `g_brInput = CreateSolidBrush(RGB(0xFF,0xFF,0xFF))` |
| `--color-text-label` | `WM_CTLCOLORSTATIC: SetTextColor(RGB(0xC8,0xC8,0xC8))` |
| `--color-text-log` | `WM_CTLCOLOREDIT(g_log): SetTextColor(RGB(0xD4,0xD4,0xD4))` |
| 字体基础 | `g_fontBase = CreateFontW(-15,...,L"Microsoft YaHei UI")` |
| 字体标题 | `g_fontTitle = CreateFontW(-18,...,FW_BOLD,...,L"Microsoft YaHei UI")` |
