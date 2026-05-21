# Claude Code VS Code 扩展 — Auto Mode macOS 修复

## 测试环境

| 项目 | 值 |
|------|-----|
| **硬件** | MacBook Air M4 |
| **操作系统** | macOS 25.5.0 (Darwin arm64) |
| **VS Code 版本** | ^1.94.0 |
| **Claude Code 扩展版本** | 2.1.145 |
| **扩展目标平台** | darwin-arm64 |
| **修改文件** | `extension/webview/index.js` |
| **测试结果** | 修改后 auto mode 出现在权限模式下拉列表中，功能正常 |

## 问题背景

在 VS Code 的 Claude Code 扩展中，权限模式下拉列表包含 `default`、`acceptEdits`、`plan`、`bypassPermissions` 等选项。其中 **auto mode** 让 Claude 自动决策是否执行工具。

**现象**：macOS 上 auto mode 不显示。WSL2（Linux）和 Windows 上正常。

**其他验证**：
- CLI 中 `claude --enable-auto-mode` 在 macOS 上可以正常使用，说明原生二进制 **有能力执行 auto mode**，问题不在核心功能逻辑
- macOS 和 Linux 的 `extension.js`、`webview/index.js` MD5 **完全一致**，排除了前端代码差异

> **该问题仅在 macOS 上可复现，Linux 和 Windows 下无法复现。**

## 根因分析

### 架构

```
GrowthBook API (api.anthropic.com/api/eval/{clientKey})
        ↑ HTTPS (remote eval, 5s timeout)
[原生二进制 (Bun 编译)] ──MCP──→ [extension.js] ──→ [webview/index.js]
        ↑                            ↑                   ↑
   GrowthBook SDK             V$("tengu_auto_mode    experimentGates
   初始化获取 features         _config", {})           .tengu_auto_mode_state
```

### 争议：并非 GrowthBook 服务端配置差异

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

### 原生二进制

- **Bun 编译**的独立可执行文件（`bun build --compile`），内含 JavaScriptCore + BoringSSL
- macOS 上通过 `Security.framework`（`SecTrustEvaluateWithError`）做证书验证，需访问 Keychain
- Linux/WSL2 上走文件系统的 CA bundle（`/etc/ssl/certs/...`）
- 二进制开启了 **Hardened Runtime**（`flags=0x10000(runtime)`），entitlements 中无 keychain 访问项

### GrowthBook 初始化

二进制内嵌 GrowthBook SDK，启动时通过 POST 请求获取远程 feature flags：

```
POST https://api.anthropic.com/api/eval/sdk-zAZezfDKGoZuXXKe
```

Client Key `sdk-zAZezfDKGoZuXXKe` 从二进制中提取。此 API 公开，无需认证（curl 可直达）。

初始化代码模式（函数名因混淆而异）：

```javascript
let client = new GrowthBookSDK({
  apiHost: "https://api.anthropic.com/",
  clientKey: "sdk-zAZezfDKGoZuXXKe",
  remoteEval: true,
  ...
});
client.init({timeout: 5000})
  .then(/* 解析 features → 写入缓存 → 通知订阅者 */)
  .catch(() => {});  // ← 错误被静默吞掉！
```

### 失败链路

1. macOS 上 GrowthBook init 的 HTTPS 请求失败（TLS 证书验证 / Keychain 访问问题）
2. 错误被 `.catch(() => {})` 静默吞掉，无日志
3. Feature 缓存（`BF` Map）为空
4. extension.js 中的门控函数调用 `V$("tengu_auto_mode_config", {})` 拿到空对象 `{}`
5. `.enabled` 为 `undefined`，回退到 `"opt-in"`
6. webview 看到 `"opt-in"`，需要 `allowDangerouslySkipPermissions` 才显示 auto mode
7. 最终 `return "unavailable"` — auto mode 不显示

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

### 跨平台差异

| | macOS | Linux/WSL2 |
|------|------|------|
| TLS 实现 | Security.framework + Keychain | 文件系统 CA bundle |
| Hardened Runtime | 有 | 无 |
| GrowthBook init | **失败**（推测） | 成功 |
| `tengu_auto_mode_config` | `undefined` → `"opt-in"` | `"enabled"` |
| 前端显示 | unavailable | available |

**注意**：`extension.js` 和 `webview/index.js` 三个平台的 MD5 完全一致，差异仅在于原生二进制。

## 修复方案

### 推荐方案：修改 webview/index.js

修改 VS Code 扩展目录下的 `webview/index.js`，绕过 GrowthBook 门控。这是最简单可靠的方式。

> **⚠️ 这是一个临时 walkaround，扩展更新后会被覆盖，需要重新应用。**

#### 核心函数

```js
autoModeAvailability = l2(() => {
  let $ = this.config.value,
      Z = this.currentModelSupportsAutoMode.value;

  if (!$ || Z === void 0) return "unknown";

  if ($.claudeSettings?.effective.permissions?.disableAutoMode === "disable" ||
      $.claudeSettings?.effective.disableAutoMode === "disable" ||
      Z === !1) return "unavailable";

  // GrowthBook 远程门控（macOS 上有问题的路径）
  let Y = $.experimentGates.tengu_auto_mode_state;
  if (Y === "enabled") return "available";
  if (Y === "opt-in" && $.allowDangerouslySkipPermissions) return "available";

  return "unavailable";  // ← macOS 走这里
});
```

#### 修改内容

将最后的 `return"unavailable"` 改为 `return"available"`。

保留的安全检查：配置未就绪 → `"unknown"`、`disableAutoMode` → `"unavailable"`、模型不支持 → `"unavailable"`。

跳过的检查：仅 GrowthBook 门控未给 `"enabled"` 就强制不可用。

#### sed 命令（版本 2.1.145）

```bash
sed -i '' 's/return"unavailable"});currentModelInfo/return"available"});currentModelInfo/' \
  ~/.vscode/extensions/anthropic.claude-code-2.1.145-darwin-arm64/webview/index.js
```

验证：

```bash
grep -c 'return"unavailable"});currentModelInfo' \
  ~/.vscode/extensions/anthropic.claude-code-2.1.145-darwin-arm64/webview/index.js
# 返回 0 表示修改成功
```

### 版本更新后的再修复方法

扩展更新后修改会被覆盖。在新版本中重新定位修改点的方法：

1. **在 `webview/index.js` 中搜索 `tengu_auto_mode_state`**，找到门控检查函数
2. **搜索 `return"unavailable"`**（紧挨着 `return"available"`），改为 `return"available"`
3. **可选**：对比新旧版本的 `extension.js`，搜索 `tengu_auto_mode_config` 确认门控逻辑是否变化

### 备用方案：修改 extension.js

如果 webview 结构变化太大，可在 `extension.js` 中找到门控函数（搜索 `tengu_auto_mode_config`），函数模式：

```javascript
function aq_(){ // 函数名随版本变化
  let H = V$("tengu_auto_mode_config", {}).enabled;
  return H === "enabled" || H === "disabled" || H === "opt-in" ? H : "opt-in";
}
```

改为直接返回 `"enabled"`。

### GrowthBook API 独立验证

如需验证 GrowthBook 服务端是否正常返回：

```bash
curl -s -X POST "https://api.anthropic.com/api/eval/sdk-zAZezfDKGoZuXXKe" \
  -H "Content-Type: application/json" \
  -d '{"features": {"tengu_auto_mode_config": {"defaultValue": {}}}}' | jq .
```

预期返回 `features.tengu_auto_mode_config.defaultValue.enabled` 为 `"enabled"`。

注意：Client Key `sdk-zAZezfDKGoZuXXKe` 可能随版本变化，需从新版 `extension.js` 中搜索 `sdk-` 前缀提取。

### 注意事项

- **仅 macOS 需要此修改**，Linux/Windows 无需任何改动
- **扩展更新后会被覆盖**，需要重新应用此修改
- 修改前需关闭 VS Code
- 修改后重启 VS Code 即可看到 auto mode 出现在权限模式下拉列表中

## 代码溯源总结

| 层级 | 文件 | 机制 | 状态 |
|------|------|------|------|
| 原生二进制 | `extension/resources/native-binary/claude` | GrowthBook SDK remote eval → 通过 MCP 发送 `experiment_gates` | macOS 上 TLS/Keychain 问题导致请求失败 |
| 扩展后端 | `extension/extension.js` | `V$()` 从原生二进制获取 feature 值 | 获取到空对象，回退到 `"opt-in"` |
| Webview | `extension/webview/index.js` | 检查 `experimentGates.tengu_auto_mode_state` | **被修改**：绕过门控 |
| 权限模式列表 | `extension/webview/index.js` | 根据 `autoModeAvailability` 决定 | 修改后生效 |
| 防回退监视器 | `extension/webview/index.js` | `autoModeAvailability === "unavailable"` 时重置 | 修改后不再触发 |
