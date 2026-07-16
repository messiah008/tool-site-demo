# 工具验证方法设计

> **版本**: v1.1
> **最后更新**: 2026-07-08
> **状态**: 验证规范框架 + PDF转Word 测试用例(已实施)
> **目的**: 统一工具验证标准,自动化生成验证用例

---

## 一、验证目标

每个工具上线前必须验证:
1. **路由可达**:URL 返回 HTTP 200
2. **页面渲染**:Vue 组件正常挂载,无 JS 错误
3. **功能正确**:核心逻辑输出正确结果
4. **边界处理**:空输入、非法输入、超长输入
5. **依赖加载**:第三方库按需加载成功
6. **错误上报**:异常自动上报到日志系统

---

## 二、工具分类验证模板

按 backend 类型分 4 类,每类验证方法不同:

### 2.1 纯前端工具(backend=frontend)

```yaml
验证项:
  - 路由: GET /{route} → 200
  - 渲染: 页面含 tool-action-label 文案
  - 功能: 输入测试数据,断言结果
  - 依赖: 检查 vendor-*.js 加载
  - 边界: 空输入不崩溃

示例(random-password):
  输入: length=16, charset=[a-z,A-Z,0-9]
  断言: result.length === 16 && /^[a-zA-Z0-9]+$/.test(result)
```

### 2.2 数据库工具(backend=db-query)

```yaml
验证项:
  - 路由: GET /{route} → 200
  - API: GET /api/query/{tool_id}?q={test_q} → 200, count >= 1
  - 渲染: 显示查询结果
  - 边界: 空查询返回全部(或提示)
  - 中文: 中文关键词编码正确

示例(id-card-region):
  输入: q=110105
  断言: response.count === 1 && response.results[0].province === '北京市'
```

### 2.3 后端代理工具(backend=proxy)

```yaml
验证项:
  - 路由: GET /{route} → 200
  - API: GET /api/proxy/{action}?{param} → 200
  - 第三方: 确认上游 API 可达
  - 超时: 上游超时返回友好错误
  - 缓存: 可选,相同查询缓存

示例(ip-query):
  输入: ip=8.8.8.8
  断言: response.country === '美国' && response.isp.includes('Google')
```

### 2.4 Worker 工具(backend=wps-worker/stirling)

```yaml
验证项:
  - 路由: GET /{route} → 200
  - 任务创建: POST /api/convert → 200, task_id
  - 状态轮询: GET /api/convert/{task_id} → status
  - 下载: GET /api/convert/{task_id}/download → 302
  - 余额扣减: premium 转换后 balance - 1

示例(pdf-to-word premium):
  前置: 余额 >= 1
  输入: test.pdf, quality=premium
  断言: task.status === 'success' && balance 减少 1
```

---

## 三、验证数据规范

每个工具的验证用例定义在 `schemas/tests/{tool_id}.json`:

```json
{
  "tool_id": "random-password",
  "category": "frontend",
  "test_cases": [
    {
      "name": "默认生成 16 位密码",
      "input": { "length": 16, "charset": ["a-z","A-Z","0-9"] },
      "assert": "result.length === 16"
    },
    {
      "name": "空字符集报错",
      "input": { "length": 16, "charset": [] },
      "assert": "error.includes('字符集')"
    }
  ]
}
```

---

## 四、自动化验证生成

`scripts/verify_tools.py` 自动遍历所有工具,按分类执行验证:

```
python scripts/verify_tools.py                    # 验证全部
python scripts/verify_tools.py --tool random-password  # 验证单个
python scripts/verify_tools.py --category frontend     # 验证一类
```

输出:
```
工具验证报告
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[frontend] random-password        ✓ 路由 ✓ 渲染 ✓ 功能 ✓ 边界
[frontend] uuid-gen               ✓ 路由 ✓ 渲染 ✓ 功能 ✓ 边界
[db-query] id-card-region         ✓ 路由 ✓ API ✓ 功能 ✓ 中文
[proxy]    ip-query               ✓ 路由 ✓ API ✓ 第三方
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
通过: 65/65 (100%)
```

---

## 五、验证检查清单(每工具)

- [ ] 路由 HTTP 200
- [ ] 页面无控制台错误(查 diagnose.py)
- [ ] 核心功能输入→输出正确
- [ ] 空输入有提示或不崩溃
- [ ] 非法输入有错误处理
- [ ] 第三方依赖按需加载(查 Network)
- [ ] 相关工具互链可跳转
- [ ] GEO 元数据(seo.json_ld)渲染
- [ ] 移动端响应式正常

---

## 六、实施计划

1. **本次(预研)**: 定义规范(本文档)
2. **下一步**: 实现 `scripts/verify_tools.py` 自动化验证器
3. **后续**: 为每个工具编写 `schemas/tests/{tool_id}.json` 测试用例
4. **持续**: 新增工具时同步补测试用例,CI 自动执行

---

## 七、PDF 转 Word 测试用例(已实施)

> **脚本**: `scripts/test_pdf_to_word.py`
> **日期**: 2026-07-08
> **结果**: 20/20 通过(100%)

### 7.1 测试范围

| 范围 | 说明 | 需 Worker |
|------|------|-----------|
| 标准(免费)转换 | 任务创建 + 入队验证 | 否 |
| 高质量(付费)转换 | 余额扣减 + 任务入队 | 否 |
| 任务查询/取消/下载 | 生命周期流转 | 否 |
| 错误处理 | 余额不足/无文件/非法参数 | 否 |
| 端到端真实转换 | 真实 PDF→docx 输出 | 是(--e2e) |

### 7.2 测试用例清单(20 项)

**1. 前置 — 测试用户(2 项)**
- 注册/登录测试用户
- 查询余额(不足自动兑换卡密)

**2. 标准转换(4 项)**
- 创建标准任务 → HTTP 200
- 返回 task_id
- 状态 = queued
- quality = standard

**3. 高质量转换(4 项)**
- 创建高质量任务 → HTTP 200
- 返回 task_id
- 状态 = queued
- 余额扣减 1(100→99)

**4. 任务查询与取消(6 项)**
- 查询任务 → 200
- task_id 一致
- tool_id = pdf-to-word
- 取消任务 → cancelled
- 取消后状态 = cancelled
- 取消的任务不能下载 → 404

**5. 错误处理(4 项)**
- 不存在工具 → 404
- 无文件 → 422
- 非法 quality → 404
- 查询不存在的 task_id → 404

### 7.3 端到端模式(--e2e)

需 WPS Worker 在线(`workers_online >= 1`),验证:
- Worker 在线检查
- 创建高质量转换任务
- 轮询 120 秒等待 success
- 下载重定向 → 302

### 7.4 测试脚本特性

| 特性 | 说明 |
|------|------|
| 自动兑换 | 余额=0 时自动取未使用卡密兑换 |
| 内置测试文件 | 121 字节最小合法 PDF,无需准备 |
| 幂等 | 每次注册新用户,可重复运行 |
| 彩色输出 | ✓/✗ 一目了然 |
| 分级 | 默认验证 API 层,--e2e 验真实转换 |

### 7.5 运行方式

```bash
# 基础(API 层,无需 Worker)
python scripts/test_pdf_to_word.py

# 端到端(需 Worker 在线)
python scripts/test_pdf_to_word.py --e2e
```

### 7.6 后续扩展

其他 Worker 类工具可复用此模式:
- `test_pdf_to_excel.py`(同结构,改 tool_id)
- `test_pdf_to_ppt.py`
- `test_id_photo.py`(改 file 类型为 image)
- `test_image_compress.py`

---

## 八、相关文档

- [25-tool-operations-sop.md](./25-tool-operations-sop.md) — 工具操作 SOP
- [27-lessons-learned.md](./27-lessons-learned.md) — 错题集
- [26-log-collection.md](./26-log-collection.md) — 日志收集
