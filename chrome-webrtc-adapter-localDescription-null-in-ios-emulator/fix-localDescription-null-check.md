# webrtc-adapter localDescription null 检查修复

## 问题描述

**错误信息：** `TypeError: Cannot read properties of null (reading 'type')`

**发生环境：** Chrome DevTools 设备模拟（iPhone/iPad）

- macOS Chrome 浏览器
- DevTools → Toggle device toolbar → 选择 iPhone/iPad
- UA 示例：`Mozilla/5.0 (iPhone; CPU iPhone OS 18_5 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/18.5 Mobile/15E148 Safari/604.1`

## 根本原因分析

### 完整链路

```
API 检测为 Chrome → UA 版本解析为 NaN → 走旧版 shim 路径 → localDescription 为 null 时报错
```

### 1. 浏览器检测

Chrome DevTools 模拟 iOS 设备时，只改变 UA 字符串，不改变浏览器 API。

`utils.js` 中的检测逻辑（第 178-186 行）：

```javascript
if (navigator.webkitGetUserMedia ||
    (window.isSecureContext === false && window.webkitRTCPeerConnection)) {
  // Chrome, Chromium, Webview, Opera.
  result.browser = 'chrome';
  result.version = parseInt(extractVersion(navigator.userAgent,
    /Chrom(e|ium)\/(\d+)\./, 2));  // ← 从 UA 中提取 Chrome 版本
}
```

- 真实 Chrome 浏览器存在 `navigator.webkitGetUserMedia` API
- 模拟 iOS 时，API 仍存在 → 被识别为 Chrome
- 但 UA 变成了 Safari 格式，不含 `Chrome/XX` → 版本解析失败

### 2. 版本解析

但 UA 不包含 `Chrome/XX` 模式，导致版本解析失败：

| 环境 | UA | 解析结果 |
|------|-----|----------|
| 正常 Chrome | `...Chrome/144.0...` | `version: 144` |
| iOS 模拟器 | `...Safari/604.1` | `version: NaN` |

**日志对比：**

```javascript
// 正常（安卓模拟）
[adapter-debug] Browser detection: {
  browser: 'chrome',
  version: 144,
  ua: 'Mozilla/5.0 (Linux; Android 13; Pixel 7) AppleWebKit/537.36 ... Chrome/144.0.0.0 Mobile Safari/537.36'
}

// 异常（iOS 模拟）
[adapter-debug] Browser detection: {
  browser: 'chrome',
  version: NaN,
  ua: 'Mozilla/5.0 (iPhone; CPU iPhone OS 18_5 like Mac OS X) AppleWebKit/605.1.15 ... Safari/604.1'
}
```

### 3. 版本判断导致走不同路径

`shimAddTrackRemoveTrack` 函数（`chrome_shim.js` 第 371-379 行）中的版本检查：

```javascript
// chrome_shim.js
if (window.RTCPeerConnection.prototype.addTrack &&
    browserDetails.version >= 65) {
  return shimAddTrackRemoveTrackWithNative(window);  // 新路径
}
// 否则走旧路径，会包装 localDescription
```

- `144 >= 65` → `true` → WithNative 路径（不包装 localDescription）
- `NaN >= 65` → `false` → OLD 路径（包装 localDescription）

**日志对比：**

```javascript
// 正常
[adapter-debug] shimAddTrackRemoveTrack called, version: 144
[adapter-debug] -> Taking WithNative path

// 异常
[adapter-debug] shimAddTrackRemoveTrack called, version: NaN
[adapter-debug] -> Taking OLD path (wraps localDescription)
```

### 4. 旧路径中的 null 访问错误

OLD 路径会包装 `localDescription` getter（`chrome_shim.js` 第 540-551 行）：

```javascript
// 修复前（原始代码）
Object.defineProperty(window.RTCPeerConnection.prototype, 'localDescription', {
  get() {
    const description = origLocalDescription.get.apply(this);
    if (description.type === '') {  // ← description 为 null 时报错！
      return description;
    }
    return replaceInternalStreamId(this, description);
  }
});
```

当 `localDescription` 为 `null` 时（连接建立前的正常状态），访问 `null.type` 抛出 TypeError。

**日志示例：**

```javascript
[adapter-debug] localDescription getter, value: null  // ← 第一次调用时为 null
[adapter-debug] localDescription getter, value: RTCSessionDescription {type: 'offer', sdp: '...'}
```

## 修复方案

**文件：** `src/js/chrome/chrome_shim.js`

**位置：** `shimAddTrackRemoveTrack` 函数内的 localDescription getter（第 546 行）

```diff
  get() {
    const description = origLocalDescription.get.apply(this);
-   if (description.type === '') {
+   if (!description || description.type === '') {
      return description;
    }
    return replaceInternalStreamId(this, description);
  }
```

增加 `!description` 检查，在 description 为 null 时直接返回，避免访问 null.type。

## 问题本质

这是一个边缘情况：

1. Chrome DevTools 的设备模拟会改变 UA，但不改变浏览器 API
2. webrtc-adapter 通过 API（`navigator.webkitGetUserMedia`）判断浏览器类型为 Chrome
3. 但版本号是从 UA 中解析的，iOS Safari UA 不含 `Chrome/XX`，解析失败得到 NaN
4. NaN 导致走了旧版 shim 路径，该路径缺少 null 检查

## 影响范围

- 仅影响 Chrome DevTools 中模拟 iOS 设备的场景
- 真实 iOS Safari 不受影响（会被正确识别为 Safari）
- 真实 Chrome 浏览器不受影响（版本解析正常）

## 仓库信息

- Fork: https://github.com/user/adapter
- 分支: `fix-localDescription-null-check`
- 基于版本: webrtc-adapter 9.0.3
