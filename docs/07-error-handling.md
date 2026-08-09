# D07 · Error Handling — onnx_to_int8.tflite

## 1. 错误码枚举（Error Code Enum）

### Python 子进程退出码（GUI `WM_APP+2` 读取）

```c
enum ExitCode {
  OK          = 0,   // 成功
  ARG_ERROR   = 1,   // 输入/校准参数错误（文件不存在、无校准图）
  MOD_ERROR   = 2,   // 缺 Python 模块（onnx2tf/tensorflow...）
  CONV_ERROR  = 3    // onnx2tf / TFLite 转换失败
};
```

### Python 内部错误函数（D04 §5）

```python
def error(msg: str, code: int = 3) -> NoReturn:
    # emit {"p":"err","msg":...} + print [ERROR] 到 stderr + sys.exit(code)
```

## 2. 错误 → 响应映射（Error → Response Mapping）

| 来源 | 条件 | stdout | 退出码 | GUI 显示 |
|------|------|--------|--------|----------|
| argparse | 缺 `--input/--out-dir/--calib` | — | 2 | 进程退出，GUI `[失败，退出码 2]` |
| 缺模块自检 | `import` 失败 | `{"p":"err","msg":"缺少模块 X"}` | 2 | `[错误] 缺少模块 X` + `[失败]` |
| 输入文件不存在 | `not isfile` | `{"p":"err",...}` | 1 | `[错误]...` + `[失败]` |
| 校准目录无图 | `not img_files` | `{"p":"err",...}` | 1 | `[错误]...` + `[失败]` |
| end2end 还原失败 | `recover_*` 返回 False | — | 1 | `[失败，退出码 1]` |
| onnx2tf 失败 | `returncode!=0` | `{"p":"err",...}` | 3 | `[错误]...` + `[失败]` |
| TFLiteConverter 异常 | 未捕获异常 | 默认 traceback | 1 | `[失败，退出码 1]` |
| 推理校验失败 | Interpreter 异常 | `log("推理校验跳过")` | 0（仍成功） | 日志提示，不致命 |

## 3. 恢复策略（Recovery Strategy）

| 错误 | 恢复 | 类型 |
|------|------|------|
| end2end 还原失败 | 提示改用原始检测头导出版本（输出 `[B,4+Nc,A]`） | 用户侧修正输入 |
| 源头拆分失败 | 自动回退 `Split` 拆分（精度稍差但可用） | 自动降级（fallback） |
| onnx2tf 失败 | 展示 stdout 末 1500 字符供诊断 | 人工排查 |
| 推理校验跳过 | 非致命，产物仍产出 | 降级（degradation） |
| GUI 无法启动 python 进程 | `append_log("[错误] 无法启动 python 进程")` + 恢复按钮 | 自动恢复按钮态 |

## 4. GUI 侧错误（非退出码）

| 情形 | 处理 |
|------|------|
| 未选 onnx/校准/输出 | `append_log("[提示] 请选择...")` + 不启动 |
| onnx 文件不存在 | `append_log("[错误] onnx 文件不存在")` |
| 输出目录无效 | 「输出目录」按钮 `EnableWindow(FALSE)` |
| 创建线程失败 | `append_log("[错误] 无法创建工作线程")` + 恢复按钮 |

## 5. 防重入（Reentrancy Guard）

```c
if (InterlockedCompareExchange(&g_running, 1, 0) != 0) return;  // 已在转换中
```
按钮禁用（`EnableWindow(g_btnStart, FALSE)`）为 UI 层双保险。

## 6. 错误日志字段约束

所有 `{"p":"err","msg":...}` 中 `msg` 为 UTF-8 字符串（C++ `MultiByteToWideChar(CP_UTF8)`）。
含中文时确保 Python 端 stdout 为 UTF-8（已 `_ensure_utf8_stdio`）。
