# 实时日志收集与 AI 调试

> **版本**: v1.0
> **最后更新**: 2026-07-03
> **状态**: 已上线,前端错误自动上报
> **目的**: AI 无需用户手动 F12,主动获取前端报错进行调试

---

## 一、工作原理

```
浏览器发生异常
   │
   ├─ 未捕获 JS 异常(window.error)
   ├─ Promise rejection(unhandledrejection)
   ├─ 资源加载失败(link/img/script)
   └─ Vue 组件错误(errorHandler)
        │
        ▼ (sendBeacon,不阻塞)
   POST /api/logs/frontend
        │
        ▼
   Redis List (logs:frontend,最近 500 条,24h TTL)
        │
        ▼ AI 查询
   GET /api/logs/frontend?limit=20
   GET /api/logs/frontend/latest
```

**关键特性**:
- 前端**自动捕获**,无需用户操作
- 用 `sendBeacon` 上报,页面关闭也能发出去
- 批量上报(500ms 内合并,减少请求)
- 后端存 Redis(内存,快),最多 500 条
- AI 用 `scripts/diagnose.py` 一键查询

---

## 二、AI 调试流程

当用户报告"页面异常/白屏/功能不工作"时,AI 按此流程:

### 2.1 第一步:查最新错误

```bash
python scripts/diagnose.py --latest
```

输出示例:
```
============================================================
  最新错误  10:57:59
============================================================
  类型: error
  消息: Cannot access 'z' before initialization
  位置: vendor-element-xxx.js:1:3836
  页面: https://localhost/
  堆栈:
    at Object.<anonymous> (vendor-element-xxx.js:1:3836)
    at ...
============================================================
```

### 2.2 第二步:查最近多条

```bash
python scripts/diagnose.py --limit 20
```

输出含错误分类统计:
```
  [1] 10:57:59  error
      Cannot access 'z' before initialization
      @ vendor-element-xxx.js:1:3836
      URL: https://localhost/

  [2] 10:55:12  resource
      资源加载失败: SCRIPT /assets/old.js
      URL: https://localhost/random-password

============================================================
  错误分类: error×3, resource×1
============================================================
```

### 2.3 第三步:根据错误定位

| 错误类型 | 典型原因 | 排查方向 |
|---------|---------|---------|
| `error` + `Cannot access 'X' before initialization` | 循环依赖/chunk 分割错误 | 检查 vite.config.ts manualChunks |
| `error` + `is not defined` | 依赖未安装或 import 错误 | 检查 package.json + 工具页 import |
| `resource` + 资源加载失败 | 文件名 hash 变化,浏览器缓存旧版 | 强制刷新 / 清缓存 |
| `unhandledrejection` | Promise 未 catch | 工具页 async 函数加 try-catch |
| `vue` + 组件错误 | Vue 组件渲染异常 | 检查组件 props/template |

### 2.4 第四步:修复后清空

```bash
python scripts/diagnose.py --clear
```

### 2.5 完整 AI 调试命令

```bash
# 1. 看最新错误
python scripts/diagnose.py --latest

# 2. 看最近 50 条
python scripts/diagnose.py --limit 50

# 3. 清空(修复后)
python scripts/diagnose.py --clear
```

---

## 三、已修复的典型问题

### 3.1 "Cannot access 'z' before initialization"(2026-07-03)

**现象**: 首页白屏,Console 报 `vendor-element-xxx.js` 初始化错误

**根因**: vite manualChunks 强制把 Element Plus 分到同一 chunk,但其内部有循环依赖,初始化顺序错乱

**修复**(`frontend/vite.config.ts`):
```typescript
// 旧(错误):
manualChunks(id) {
  if (id.includes('element-plus')) return 'vendor-element'  // ← 强制分块破坏循环依赖
}

// 新(正确):
manualChunks(id) {
  // element-plus 不手动分块,让 Vite 自动处理内部依赖
  // 只精确分 vue/router/pinia(用 /vue/ 精确匹配,避免误捕 @vue/)
  if (id.includes('/vue/') || id.includes('/vue-router/') || id.includes('/pinia/')) {
    return 'vendor-vue'
  }
  // element-plus 和其他库返回 undefined(自动分包)
}
```

**教训**: 大型库(Element Plus/Lodash 等)有内部循环依赖时,不要强制 manualChunks,让 Vite 自动处理。

---

## 四、API 参考

### 4.1 上报日志(前端调用)

```
POST /api/logs/frontend
Content-Type: application/json

{
  "logs": [
    {
      "type": "error",           // error/unhandledrejection/resource/vue
      "message": "错误消息",
      "stack": "堆栈(可选)",
      "filename": "文件名(可选)",
      "lineno": 1,                // 行号
      "colno": 1,                 // 列号
      "url": "https://localhost/",
      "userAgent": "Mozilla/5.0...",
      "timestamp": "2026-07-03T11:00:00Z"
    }
  ]
}
```

响应: `{"received": 1}`

### 4.2 查询日志(AI/调试用)

```
GET /api/logs/frontend?limit=20
```

响应:
```json
{
  "total": 5,
  "limit": 20,
  "logs": [...]
}
```

### 4.3 查最新一条

```
GET /api/logs/frontend/latest
```

响应:
```json
{
  "has_error": true,
  "log": {...}
}
```

### 4.4 清空

```
DELETE /api/logs/frontend
```

---

## 五、前端实现

### 5.1 错误捕获模块

文件: `frontend/src/utils/errorTracker.ts`

- `initErrorTracking()` — 初始化,在 main.ts 调用
- `vueErrorHandler()` — Vue 错误处理器
- 捕获 4 类异常:JS error / Promise rejection / 资源加载 / Vue 错误
- 用 `sendBeacon` 批量上报(500ms 合并)

### 5.2 接入

`frontend/src/main.ts`:
```typescript
import { initErrorTracking, vueErrorHandler } from './utils/errorTracker'

initErrorTracking()  // 必须在 mount 前
const app = createApp(App)
app.config.errorHandler = vueErrorHandler
app.mount('#app')
```

---

## 六、相关文档

- [25-tool-operations-sop.md](./25-tool-operations-sop.md) — 工具操作 SOP
- [19-runbook.md](./19-runbook.md) — 运维故障处理
- [09-deployment.md](./09-deployment.md) — 部署方案
