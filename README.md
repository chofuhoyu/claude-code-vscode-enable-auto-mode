# Claude Code VS Code 扩展 — Auto Mode macOS 修复

## 测试环境

| 项目 | 值 |
|------|-----|
| **硬件** | MacBook Air M4 |
| **操作系统** | macOS 25.5.0 (Darwin arm64) |
| **VS Code 版本** | ^1.94.0（兼容） |
| **Claude Code 扩展版本** | 2.1.145 |
| **扩展目标平台** | darwin-arm64 |
| **修改文件** | `extension/webview/index.js`（4.6MB） |
| **修改方法** | 手动编辑或 sed 命令 |
| **测试结果** | ✅ 修改后 auto mode 出现在权限模式下拉列表中，功能正常 |

## 问题背景

在 VS Code 的 Claude Code 扩展中，权限模式下拉列表（Permission Mode）包含 `default`、`acceptEdits`、`plan`、`bypassPermissions` 等选项。其中 **auto mode** 是一种让 Claude 自动决策是否执行工具的模式。

**现象**：在 macOS 上，auto mode 不显示在权限模式列表中。但在 WSL2（Linux）和 Windows 上，auto mode 正常显示。

**其他验证**：
- CLI 中 `claude --enable-auto-mode` 在 macOS 上可以正常使用，说明原生二进制 **有能力执行 auto mode**，问题不在核心功能逻辑
- macOS 和 Linux 的 `extension.js`、`webview/index.js` MD5 **完全一致**，排除了前端代码差异

> **该问题仅在 macOS 上可复现，Linux 和 Windows 下无法复现。**

## 根因分析（推测）

### 争议：并非 GrowthBook 服务端配置差异

Claude Code 的原生二进制内嵌了 **GrowthBook** SDK（特征标志平台），启动时会拉取远程实验配置。涉及的关键特征标志是 `tengu_auto_mode_config.enabled`。

通过直接调用 GrowthBook remote eval API 验证（该端点公开，无需认证）：

```bash
# Linux 和 macOS 返回完全相同的值
curl -s "https://api.anthropic.com/api/eval/sdk-zAZezfDKGoZuXXKe" \
  -X POST -H "Content-Type: application/json" \
  -d '{"attributes":{"platform":"linux"}}' | jq '.features.tengu_auto_mode_config'
# → { "enabled": "enabled", ... }

curl -s "https://api.anthropic.com/api/eval/sdk-zAZezfDKGoZuXXKe" \
  -X POST -H "Content-Type: application/json" \
  -d '{"attributes":{"platform":"darwin"}}' | jq '.features.tengu_auto_mode_config'
# → { "enabled": "enabled", ... }  相同！
```

两个平台返回的值 **完全一致**，均为 `"enabled"`。因此问题**不在 GrowthBook 服务端配置**。

### 推测根因：macOS 原生二进制内部网络请求失败

问题可能出在 macOS 原生二进制内部的 GrowthBook SDK 网络请求环节。在 macOS 上，二进制内嵌的 Node.js 运行时向 `api.anthropic.com` 发起 GrowthBook remote eval 请求时：

- 可能因 **SSL 证书链**（bundled CA 证书与 macOS 系统证书不一致）失败
- 可能因 **代理环境**（macOS 下 VS Code 扩展沙箱的网络环境差异）失败
- 或有其他 **macOS 平台特定的运行时差异**

导致 GrowthBook 特征数据为空，代码回退到 `"opt-in"`，最终 webview 返回 `"unavailable"`。

以下函数是确定 `tengu_auto_mode_state` 的核心逻辑（Linux 和 macOS 的函数名因编译混淆而异，但逻辑相同）：

```js
// Linux: aq_() / macOS: z95()
function aq_() {
  let H = V$("tengu_auto_mode_config", {})?.enabled;
  return H === "enabled" || H === "disabled" || H === "opt-in" ? H : "opt-in";
}
```

```js
// webview 中的消费逻辑
let Y = $.experimentGates.tengu_auto_mode_state;
if (Y === "enabled") return "available";       // Linux/Windows 走这里
if (Y === "opt-in" && ...) return "available";  // macOS GrowthBook 失败后回退到这里但 bypass 条件不满足
return "unavailable";                           // ← macOS 实际走到这里
```

## 临时解决方案

修改 VS Code 扩展安装目录下的 `webview/index.js`，绕过门控检查的最后一个 `return` 语句。

> **⚠️ 这是一个临时 walkaround，扩展更新后会被覆盖，需要重新应用。**

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

  // ③ GrowthBook 远程门控（macOS 上有问题的路径）
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

- **仅 macOS 需要此修改**，Linux/Windows 无需任何改动
- **扩展更新后会被覆盖**，需要重新应用此修改
- 修改前需关闭 VS Code
- 修改后重启 VS Code 即可看到 auto mode 出现在权限模式下拉列表中

## 代码溯源总结

| 层级 | 文件 | 机制 | 状态 |
|------|------|------|------|
| 扩展后端 | `extension/extension.js` | `buildGate: ()=>!1` | 硬编码关闭，仅影响 JSON Schema，**无运行时影响** |
| 原生二进制 | `extension/resources/native-binary/claude` | 通过 MCP 发送 `experiment_gates` | **负责**：从 GrowthBook 远程 eval 获取门控值（macOS 上可能因网络/SSL 失败） |
| Webview | `extension/webview/index.js` | 检查 `experimentGates.tengu_auto_mode_state` | **被修改**：绕过门控检查 |
| 权限模式列表 | `extension/webview/index.js` | `M5()` 根据 `autoModeAvailability` 决定 | 修改后生效 |
| 防回退监视器 | `extension/webview/index.js` | `autoModeAvailability === "unavailable"` 时重置 | 修改后不再触发 |

**跨平台一致性**：macOS/Linux/Windows 三个平台的 `extension.js`、`webview/index.js`、`package.json`、`schema.json` MD5 完全一致。差异仅在于原生二进制（`claude` Mach-O arm64 vs `claude` ELF x86-64）。

### GrowthBook remote eval 端点

该端点为公开 API，无需认证即可访问（Client Key `sdk-zAZezfDKGoZuXXKe` 为公开标识符）。实测三个平台发送相同 attributes 时返回结果一致：

```bash
curl -s "https://api.anthropic.com/api/eval/sdk-zAZezfDKGoZuXXKe" \
  -X POST -H "Content-Type: application/json" \
  -d '{"attributes":{"platform":"<linux|darwin|win32>"}}' | jq '.features.tengu_auto_mode_config'
```

结果均为 `{ "enabled": "enabled", "twoStageClassifier": true, "model": "claude-opus-4-6[1m]" }`。
