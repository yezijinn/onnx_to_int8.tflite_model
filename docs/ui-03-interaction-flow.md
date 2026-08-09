# U03 · Interaction Flow — onnx_to_int8.tflite

## 1. 用户主流程（Primary Flow）

```text
[启动] → 显示窗口 + 日志首行作者信息
   │
   ├─ 选 ONNX (浏览/手填)
   ├─ 选校准目录 (浏览/手填)
   ├─ 选输出目录 (浏览/手填) ──▶ EN_CHANGE/update_open_btn → 输出目录按钮启用
   │
   ▼
[点 开始转换]
   │
   ├─ 校验：三框非空 + onnx 文件存在（否则提示并中止）
   ├─ 禁用开始按钮 + 文案「正在转换......」
   ├─ 日志：开始时间提示
   ├─ 启动后台线程 → CreateProcessW(python.exe ...)
   │
   ▼ (后台)
 python 逐行 emit → 管道 → WM_APP+1 → 日志实时
   │
   ▼
[进程退出 WM_APP+2]
   ├─ code==0 → 完成时间 + 新文件路径 + 作者信息；恢复按钮
   └─ code≠0 → [失败，退出码 N]；恢复按钮
```

## 2. 状态机（State Machine）

```text
            ┌──────────┐
            │  IDLE    │◀──────────────────────────┐
            └────┬─────┘                           │
      点开始(校验通过) │                            │
                 ▼      │ 校验失败/进程退出          │
            ┌──────────┐│                          │
            │ RUNNING  │┘                          │
            └────┬─────┘                           │
       进程退出   │ code==0 / code≠0               │
                 ▼                                 │
            ┌──────────┐                           │
            │ DONE/ERR │───────────────────────────┘ (恢复 IDLE)
            └──────────┘
```

状态存储：`g_running` (0=IDLE, 1=RUNNING)，`Interlocked` 保护。

## 3. 输出目录按钮状态流（关键）

```text
[无输出目录] ──▶ 按钮 disabled
[设有效目录] ──▶ EN_CHANGE / browse → update_open_btn → enabled
[点按钮]    ──▶ open_out_dir:
                ├─ 产物已存在 → explorer /select,"产物路径"（定位选中）
                └─ 产物未生成 → explorer "输出目录"
```

> ⚠️ 跨进程 `SetWindowText` 不触发 `EN_CHANGE`，故 `browse()` 后**显式调用 `update_open_btn()`**。

## 4. 错误流（Error Flow）

```text
缺输入 → [提示] 请选择... → 停（不启动）
onnx 不存在 → [错误] onnx 文件不存在 → 停
python 启动失败 → [错误] 无法启动 python 进程 → 恢复按钮
转换失败 → stdout [错误] + 退出码 → [失败，退出码 N]
```

## 5. 键盘快捷键（Keyboard）

- 默认：`Tab` 切换控件，`Space/Enter` 触发默认按钮（开始转换，`BS_DEFPUSHBUTTON`）。
- 无自定义快捷键。

## 6. 边界与防呆（Edge Cases）

| 场景 | 行为 |
|------|------|
| 转换中再点开始 | `InterlockedCompareExchange` 拒绝（按钮也禁用） |
| 输出目录中途被删 | 产物定位失败 → 回退打开输出目录根 |
| 中文路径 | 全链路 UTF-8 + ASCII 临时目录兼容 |
| 强制关闭窗口 | `WM_CLOSE` → `DestroyWindow` → `PostQuitMessage` |
