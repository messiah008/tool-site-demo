# API 契约 (OpenAPI 3.0)

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **文件位置**: `schemas/api/openapi.yaml`
> **状态**: 设计完成,待实现
> **作用**: 前后端接口的**单一事实来源**,前端自动生成 API Client,后端自动校验

---

## 一、契约的作用

这份 OpenAPI 契约是前后端的**唯一接口真相**:

```
openapi.yaml (单一来源)
    │
    ├─→ 前端: openapi-typescript 生成 TS 类型 + API Client
    ├─→ 后端: FastAPI 自动生成(比对一致性)
    └─→ 测试: schema 验证 mock 数据
```

**核心原则**:
1. 接口变更**先改 openapi.yaml**,再改代码
2. 前端不手写 API 调用,全部从契约生成
3. 后端 FastAPI 路由必须与契约一致,CI 校验

---

## 二、完整 OpenAPI 规范

```yaml
openapi: 3.0.3
info:
  title: 知鸟工具箱 API
  version: 1.0.0
  description: |
    知鸟工具箱后端 API。
    - 工具转换(PDF/图片/文本等)
    - 卡密兑换
    - 用户/余额管理
    - 订单/支付
servers:
  - url: https://api.zhiniao.tools
    description: 生产
  - url: http://localhost:8000
    description: 开发

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    # ===== 通用 =====
    Error:
      type: object
      required: [error_code, message]
      properties:
        error_code:
          type: string
          description: 错误码,见错误码表
        message:
          type: string
          description: 用户友好提示

    # ===== 用户 =====
    User:
      type: object
      properties:
        id: { type: string, format: uuid }
        username: { type: string }
        balance:
          type: object
          properties:
            credits: { type: integer, description: 高质量转换剩余次数 }
            total_redeemed: { type: integer }
            total_consumed: { type: integer }

    # ===== 卡密 =====
    RedeemRequest:
      type: object
      required: [card_key]
      properties:
        card_key:
          type: string
          pattern: '^ZN-[A-Z0-9]{4}-[A-Z0-9]{4}-[A-Z0-9]{4}$'
          description: 卡密,格式 ZN-XXXX-XXXX-XXXX
          example: ZN-AB12-CD34-EF56

    RedeemResponse:
      type: object
      required: [success, credits_added, balance]
      properties:
        success: { type: boolean }
        credits_added: { type: integer, description: 本次兑换次数 }
        balance: { type: integer, description: 兑换后总余额 }
        package_type: { type: string, description: 套餐类型 }
        expires_at: { type: string, format: date-time }

    # ===== 转换 =====
    ConvertRequest:
      type: object
      required: [tool_id, quality]
      properties:
        tool_id:
          type: string
          description: 工具 ID,见 tool-registry
          example: pdf-to-word
        quality:
          type: string
          enum: [standard, premium]
          description: standard=免费LibreOffice, premium=付费WPS+OCR
        options:
          type: object
          description: 工具特定选项
          properties:
            ocr: { type: boolean, default: true }
            format: { type: string, default: docx }

    ConvertTask:
      type: object
      required: [id, status]
      properties:
        id: { type: string, format: uuid }
        tool_id: { type: string }
        quality: { type: string }
        status:
          type: string
          enum: [queued, processing, success, failed, timeout, cancelled]
        progress: { type: integer, minimum: 0, maximum: 100 }
        error_code: { type: string }
        error_message: { type: string }
        download_url: { type: string, description: 预签名下载URL,仅success有 }
        file_size: { type: integer }
        duration_ms: { type: integer }
        created_at: { type: string, format: date-time }
        completed_at: { type: string, format: date-time }

    # ===== 订单 =====
    CreateOrderRequest:
      type: object
      required: [package_type, channel]
      properties:
        package_type:
          type: string
          enum: [single_50, monthly_100, yearly_unlimited]
        channel:
          type: string
          enum: [wechat, alipay]

    Order:
      type: object
      properties:
        id: { type: string, format: uuid }
        order_no: { type: string }
        amount: { type: number, format: float }
        status: { type: string, enum: [pending, paid, delivered, refunded] }
        pay_url: { type: string, description: 支付链接 }

security:
  - BearerAuth: []

paths:
  # ===== 认证 =====
  /api/auth/register:
    post:
      summary: 用户注册
      security: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [username, password]
              properties:
                username: { type: string }
                password: { type: string, minLength: 8 }
                email: { type: string, format: email }
      responses:
        '201':
          description: 注册成功
          content:
            application/json:
              schema:
                type: object
                properties:
                  user: { $ref: '#/components/schemas/User' }
                  token: { type: string }

  /api/auth/login:
    post:
      summary: 用户登录
      security: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [username, password]
              properties:
                username: { type: string }
                password: { type: string }
      responses:
        '200':
          description: 登录成功
          content:
            application/json:
              schema:
                type: object
                properties:
                  user: { $ref: '#/components/schemas/User' }
                  token: { type: string }

  # ===== 卡密 =====
  /api/auth/redeem:
    post:
      summary: 卡密兑换
      description: 输入卡密,激活高质量转换次数
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/RedeemRequest' }
      responses:
        '200':
          description: 兑换成功
          content:
            application/json:
              schema: { $ref: '#/components/schemas/RedeemResponse' }
        '400':
          description: 卡密无效或已使用
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }
        '429':
          description: 兑换频率过高
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }

  /api/auth/balance:
    get:
      summary: 查询余额
      responses:
        '200':
          description: 余额信息
          content:
            application/json:
              schema:
                type: object
                properties:
                  credits: { type: integer }
                  total_redeemed: { type: integer }
                  total_consumed: { type: integer }

  # ===== 转换 =====
  /api/convert:
    post:
      summary: 创建转换任务
      description: 上传文件,创建转换任务,返回 task_id
      requestBody:
        required: true
        content:
          multipart/form-data:
            schema:
              type: object
              required: [file, tool_id, quality]
              properties:
                file:
                  type: string
                  format: binary
                tool_id: { type: string }
                quality: { type: string, enum: [standard, premium] }
                options: { type: string, description: JSON 字符串 }
      responses:
        '200':
          description: 任务已创建
          content:
            application/json:
              schema:
                type: object
                properties:
                  task_id: { type: string, format: uuid }
                  status: { type: string }
        '402':
          description: 余额不足(高质量转换)
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }
        '429':
          description: 配额已用完

  /api/convert/{task_id}:
    get:
      summary: 查询任务状态
      parameters:
        - name: task_id
          in: path
          required: true
          schema: { type: string, format: uuid }
      responses:
        '200':
          description: 任务详情
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ConvertTask' }
        '404':
          description: 任务不存在

    delete:
      summary: 取消任务
      parameters:
        - name: task_id
          in: path
          required: true
          schema: { type: string, format: uuid }
      responses:
        '200':
          description: 已取消

  /api/convert/{task_id}/download:
    get:
      summary: 下载转换结果
      parameters:
        - name: task_id
          in: path
          required: true
          schema: { type: string, format: uuid }
      responses:
        '302':
          description: 重定向到 MinIO 预签名 URL

  # ===== 订单 =====
  /api/payment/order:
    post:
      summary: 创建订单
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/CreateOrderRequest' }
      responses:
        '200':
          description: 订单已创建
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Order' }

  /api/payment/callback/wechat:
    post:
      summary: 微信支付回调
      security: []
      responses:
        '200':
          description: 处理成功

  # ===== 工具 =====
  /api/tools:
    get:
      summary: 获取工具列表
      description: 返回工具注册表的全部工具
      security: []
      responses:
        '200':
          description: 工具列表
          content:
            application/json:
              schema:
                type: array
                items:
                  type: object
                  properties:
                    id: { type: string }
                    name: { type: string }
                    category: { type: string }
                    is_premium: { type: boolean }
                    backend: { type: string }

  # ===== 健康 =====
  /api/health:
    get:
      summary: 健康检查
      security: []
      responses:
        '200':
          description: 服务正常
          content:
            application/json:
              schema:
                type: object
                properties:
                  api: { type: string }
                  redis: { type: string }
                  postgres: { type: string }
                  minio: { type: string }
                  workers_online: { type: integer }
```

---

## 三、错误码表

| 错误码 | HTTP | 含义 | 用户提示 |
|--------|------|------|---------|
| E001 | 500 | WPS 启动失败 | 服务暂时不可用,请稍后重试 |
| E002 | 400 | PDF 打开失败 | 文件可能损坏,请检查后重新上传 |
| E003 | 500 | 转换菜单定位失败 | 服务升级中,请稍后重试 |
| E004 | 500 | OCR 选项设置失败 | 服务异常,请稍后重试 |
| E005 | 504 | 转换超时 | 文件较大或复杂,转换超时 |
| E006 | 500 | 输出文件校验失败 | 转换异常,请稍后重试 |
| E007 | 500 | 文件下载失败 | 服务异常,请稍后重试 |
| E008 | 500 | 文件上传失败 | 服务异常,请稍后重试 |
| E101 | 400 | 卡密格式错误 | 卡密格式不正确 |
| E102 | 400 | 卡密无效 | 卡密无效或已被使用 |
| E103 | 400 | 卡密已过期 | 卡密已过期 |
| E201 | 402 | 余额不足 | 余额不足,请兑换或购买 |
| E202 | 429 | 配额已用完 | 今日免费次数已用完,明日再来 |
| E301 | 429 | 频率过高 | 操作过于频繁,请稍后重试 |
| E401 | 401 | 未登录 | 请先登录 |
| E403 | 403 | 无权限 | 无权访问此资源 |
| E404 | 404 | 资源不存在 | 资源不存在或已删除 |
| E500 | 500 | 未知错误 | 服务异常,请稍后重试 |

---

## 四、代码生成

### 4.1 前端生成 TypeScript Client

```bash
# 安装工具
npm install -D openapi-typescript openapi-typescript-codegen

# 生成类型
npx openapi-typescript schemas/api/openapi.yaml -o frontend/src/api/types.ts

# 生成 Client
npx openapi-typescript-codegen --input schemas/api/openapi.yaml \
  --output frontend/src/api/client --client axios
```

生成的代码可直接使用:
```typescript
import { ConvertService } from '@/api/client'

const task = await ConvertService.createConvert({
  file, tool_id: 'pdf-to-word', quality: 'premium'
})
```

### 4.2 后端校验一致性

FastAPI 自动生成 OpenAPI,与 `openapi.yaml` 比对:

```bash
# 启动后端
uvicorn app.main:app

# 导出实际 OpenAPI
curl http://localhost:8000/openapi.json > actual.json

# 比对
python scripts/compare_openapi.py schemas/api/openapi.yaml actual.json
```

CI 中执行此比对,不一致则构建失败。

---

## 五、版本管理

- 接口变更**必须**先改 `openapi.yaml`
- 重大变更 bump major 版本(v1 → v2),保留旧版本路由
- 小变更 bump minor 版本(v1.0 → v1.1)
- 变更记录在 [01-progress.md](./01-progress.md)

---

## 六、Mock 与测试

### 6.1 前端 Mock

```bash
# 用 Prism 启动 Mock 服务
npx prism mock schemas/api/openapi.yaml --port 4010
```

前端开发时指向 Mock,不依赖后端。

### 6.2 契约测试

```bash
# 后端测试用 schema 验证响应
pytest tests/test_contract.py
```

详见 [07-consistency-link.md](./07-consistency-link.md)。
