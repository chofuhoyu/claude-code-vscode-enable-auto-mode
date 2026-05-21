# Claude Code VS Code 扩展 — Auto Mode macOS 修复

## 问题背景

在 VS Code 的 Claude Code 扩展中，权限模式下拉列表（Permission Mode）包含 `default`、`acceptEdits`、`plan`、`bypassPermissions` 等选项。其中 **auto mode** 是一种让 Claude 自动决策是否执行工具的模式。

**现象**：在 macOS 上，auto mode 不显示在权限模式列表中。但在 WSL2（Linux）和 Windows 上，auto mode 正常显示。

**验证**：CLI 中 `claude --enable-auto-mode` 在 macOS 上可以正常使用，说明原生二进制 **有能力执行 auto mode**，问题不在二进制本身。

## 根因：GrowthBook 远程实验门控

Claude Code 的原生二进制（`extension/resources/native-binary/claude`）内嵌了 **GrowthBook** SDK（特征标志平台），启动时会从 `https://cdn.growthbook.io` 拉取远程实验配置。

其中一个特征标志是 `tengu_auto_mode_state`，它控制 **webview UI 是否显示 auto mode 切换按钮**：

- **Linux/Windows**：GrowthBook 返回 `"enabled"` → webview 显示 auto 选项
- **macOS**：GrowthBook 未返回该标志 → webview 隐藏 auto 选项

而 `--enable-auto-mode` CLI 参数**不走 GrowthBook**，直接硬编码进二进制启动逻辑，所以不受影响。

## 修改方案

修改 VS Code 扩展安装目录下的 `webview/index.js`，绕过 GrowthBook 门控检查。

### 核心函数

```js
autoModeAvailability = l2(() => {
  let $ = this.config.value,
      Z = this.currentModelSupportsAutoMode.value;

  // ① 配置或模型信息未就绪
  if (!$ || Z === void 0) return "unknown";

  // ② 明确禁用或模型不支持
  if ($.claudeSettings?.effective.permissions?.disableAutoMode === "disable" ||
      $.claudeSettings?.effective.disableAutoMode === "disable" ||
      Z === !1) return "unavailable";

  // ③ GrowthBook 远程门控（问题所在）
  let Y = $.experimentGates.tengu_auto_mode_state;
  if (Y === "enabled") return "available";
  if (Y === "opt-in" && $.allowDangerouslySkipPermissions) return "available";

  return "unavailable";  // ← macOS 走这里，改为 "available"
});
```

### 修改内容

将函数的最后一个 `return` 值从 `"unavailable"` 改为 `"available"`。

**保留的安全检查**：
- ❌ 配置未就绪 → `"unknown"`（保留）
- ❌ `disableAutoMode: "disable"` → `"unavailable"`（保留）
- ❌ 模型不支持（`supportsAutoMode: false`）→ `"unavailable"`（保留）
- ✅ `"enabled"` 和 `"opt-in"` 的原有逻辑（保留）

**跳过的检查**：
- ❌ 仅因 GrowthBook 门控未给 `"enabled"` 就强制不可用 → 改为 `"available"`

### 精确的 sed 命令

```bash
sed -i '' 's/return"unavailable"});currentModelInfo/return"available"});currentModelInfo/' \
  ~/.vscode/extensions/anthropic.claude-code-2.1.145-darwin-arm64/webview/index.js
```

### 关键字符串（用于搜索定位）

搜索以下精简化字符串定位修改点（4.6MB 文件中唯一，仅出现 1 次）：

```
return"unavailable"});currentModelInfo=l2(()=>{
```

替换为：

```
return"available"});currentModelInfo=l2(()=>{
```

### 修改后验证

修改后的函数末尾应为：

```js
return"available"});
```

可以用以下命令验证：

```bash
grep -c 'return"unavailable"});currentModelInfo' \
  ~/.vscode/extensions/anthropic.claude-code-2.1.145-darwin-arm64/webview/index.js
```

返回 `0` 表示修改成功。

### 注意事项

- **扩展更新后会被覆盖**，需要重新应用此修改
- 修改前需关闭 VS Code
- 修改后重启 VS Code 即可看到 auto mode 出现在权限模式下拉列表中

## 代码溯源总结

| 层级 | 文件 | 机制 | 状态 |
|------|------|------|------|
| 扩展后端 | `extension/extension.js` | `buildGate: ()=>!1` | 硬编码关闭，仅影响 JSON Schema，**无运行时影响** |
| 原生二进制 | `extension/resources/native-binary/claude` | 通过 MCP 发送 `experiment_gates` | **负责**：从 GrowthBook 获取门控值 |
| Webview | `extension/webview/index.js` | 检查 `experimentGates.tengu_auto_mode_state` | **被修改**：绕过门控检查 |
| 权限模式列表 | `extension/webview/index.js` | `M5()` 根据 `autoModeAvailability` 决定 | 修改后生效 |
| 防回退监视器 | `extension/webview/index.js` | `autoModeAvailability === "unavailable"` 时重置 | 修改后不再触发 |

**跨平台一致性**：macOS/Linux/Windows 三个平台的 `extension.js`、`webview/index.js`、`package.json`、`schema.json` MD5 完全一致。差异仅在于原生二进制（`claude` vs `claude.exe`）。
