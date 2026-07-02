# 多微服务架构设计

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **状态**: 设计完成
> **目的**: 多 Worker 类型的统一调度、扩缩容、监控

---

## 一、为什么需要多微服务

### 1.1 单一 Worker 的局限

如果所有工具用一个 Worker:
- WPS 任务卡住,Stirling 任务也卡住(同进程)
- WPS 必须跑 Windows,Stirling 跑 Linux,无法共存
- 升级一个工具要重启全部
- 一个工具内存泄漏,拖垮所有工具

### 1.2 多微服务的价值

| 价值 | 说明 |
|------|------|
| 故障隔离 | WPS Worker 挂了,图片压缩照常 |
| 独立扩容 | PDF 转换高峰时只扩 WPS Worker |
| 技术栈自由 | WPS 用 Python+Windows,Stirling 用 Java+Linux |
| 独立部署 | 升级 Stirling 不影响 WPS |
| 资源优化 | 重计算任务用 GPU 机器,轻任务用小机器 |

---

## 二、Worker 类型分类

### 2.1 五大 Worker 类型

```
┌──────────────────────────────────────────────────────┐
│                    API 服务                          │
│         (根据 tool.backend 路由到对应队列)            │
└────────┬─────────┬──────────┬──────────┬────────────┘
         │         │          │          │
    ┌────▼───┐ ┌──▼─────┐ ┌──▼─────┐ ┌──▼─────┐ ┌────────┐
    │queue:  │ │queue:  │ │queue:  │ │queue:  │ │ 无队列  │
    │wps     │ │stirling│ │ocr     │ │ai      │ │前端处理 │
    └────┬───┘ └──┬─────┘ └──┬─────┘ └──┬─────┘ └────────┘
         │        │          │          │
    ┌────▼───┐ ┌──▼─────┐ ┌──▼─────┐ ┌──▼─────┐
    │WPS     │ │Stirling│ │PaddleOCR│ │LLM API │
    │Worker  │ │Worker  │ │Worker  │ │Worker  │
    │(Win)   │ │(Linux) │ │(Linux) │ │(Linux) │
    └────────┘ └────────┘ └────────┘ └────────┘
```

### 2.2 各类型详情

| 类型 | 后端标识 | 平台 | 技术栈 | 处理工具 |
|------|---------|------|--------|---------|
| WPS Worker | `wps-worker` | Windows | Python + kwpsconvert CLI | PDF↔Word/Excel/PPT, CAJ, 证件照 |
| Stirling Worker | `stirling` | Linux 容器 | Java (Stirling-PDF) | PDF 合并/拆分/压缩/水印/解密 |
| OCR Worker | `ocr-worker` | Linux 容器 | Python + PaddleOCR | 图片 OCR, 发票识别 |
| AI Worker | `ai-worker` | Linux 容器 | Python + LLM API | 批量翻译, 文档摘要 |
| 前端处理 | `frontend` | 浏览器 | JS | JSON 格式化, Base64, 密码生成 |

### 2.3 工具到 Worker 的映射

**单一事实来源**: `schemas/tools.registry.json` 的 `backend` 字段

```json
[
  { "id": "pdf-to-word", "backend": "wps-worker", "is_premium": true },
  { "id": "pdf-merge", "backend": "stirling", "is_premium": false },
  { "id": "pdf-ocr", "backend": "stirling", "is_premium": true },
  { "id": "image-compress", "backend": "frontend", "is_premium": false },
  { "id": "batch-translate", "backend": "ai-worker", "is_premium": true },
  { "id": "id-photo", "backend": "wps-worker", "is_premium": true }
]
```

API 启动时加载 registry,根据 `backend` 路由到对应队列。

---

## 三、统一调度层

### 3.1 API 端路由逻辑

```python
# backend/app/api/convert.py
from app.utils.tool_registry import TOOLS

WORKER_QUEUES = {
    "wps-worker": "queue:wps-worker",
    "stirling": "queue:stirling",
    "ocr-worker": "queue:ocr-worker",
    "ai-worker": "queue:ai-worker",
    # frontend 不入队,前端直接处理
}

@router.post("/api/convert")
async def create_convert(..., tool_id: str = Form(...)):
    tool = TOOLS.get(tool_id)
    if not tool:
        raise HTTPException(400, {"error_code": "E404"})

    # 纯前端工具,无需入队
    if tool["backend"] == "frontend":
        raise HTTPException(400, {"error_code": "E400",
                                   "message": "此工具在前端处理"})

    # 付费工具检查余额
    if tool["is_premium"] and quality == "premium":
        await check_and_consume_balance(user)

    # 上传文件到 MinIO
    input_key = await upload_to_minio(file)

    # 创建任务记录
    task = await create_task_record(tool_id, quality, input_key)

    # ★ 根据 backend 路由到对应队列
    queue = WORKER_QUEUES[tool["backend"]]
    await redis.lpush(queue, json.dumps(task))

    return {"task_id": task["task_id"], "status": "queued"}
```

### 3.2 队列设计

```
queue:wps-worker        # WPS Worker 消费
queue:wps-worker:priority  # 高优先级(付费用户)
queue:stirling          # Stirling Worker 消费
queue:stirling:priority
queue:ocr-worker        # OCR Worker 消费
queue:ai-worker         # AI Worker 消费
```

**优先级策略**:
- 付费用户任务入 `:priority` 队列,Worker 优先消费
- 免费用户任务入普通队列
- Worker 消费时先 `BRPOP :priority`,再 `BRPOP` 普通

```python
# Worker 消费逻辑
def get_next_task(redis, queue_base):
    # 优先消费高优先级
    result = redis.brpop(f"{queue_base}:priority", timeout=1)
    if not result:
        result = redis.brpop(queue_base, timeout=30)
    return result
```

---

## 四、独立扩缩容

### 4.1 扩容策略

每类 Worker 独立扩容,根据队列长度自动决策:

```python
# backend/app/services/autoscaler.py
async def check_and_scale():
    for worker_type, queue_name in WORKER_QUEUES.items():
        backlog = await redis.llen(queue_name)
        online_workers = await count_online_workers(worker_type)

        # 每个 Worker 处理能力(假设 5 任务/分钟)
        capacity = online_workers * 5
        demand = backlog  # 简化:积压数

        if demand > capacity * 2:
            await scale_up(worker_type)   # 扩容
        elif demand < capacity * 0.5 and online_workers > 1:
            await scale_down(worker_type)  # 缩容
```

### 4.2 各类 Worker 扩容方式

| Worker 类型 | 扩容方式 | 启动时间 |
|------------|---------|---------|
| WPS Worker | 启动新 Windows 机器/进程 | 慢(分钟级) |
| Stirling Worker | Docker scale | 快(秒级) |
| OCR Worker | Docker scale | 中(GPU 模型加载) |
| AI Worker | Docker scale | 快(调 API) |

```bash
# Stirling Worker 扩容到 3 实例
docker compose up -d --scale stirling-worker=3

# WPS Worker 扩容(手动启动新机器)
# 配置 worker_id=wps-worker-02,启动即可
```

### 4.3 扩容触发条件

| 指标 | 阈值 | 动作 |
|------|------|------|
| 队列积压 | > 50 | 扩容 1 实例 |
| 队列积压 | > 100 | 扩容 2 实例 |
| 处理耗时 P95 | > 5 分钟 | 扩容 |
| Worker 离线 | 心跳过期 | 告警 + 任务重排 |
| 成功率 | < 90% | 告警(不扩容,先排查) |

---

## 五、统一监控

### 5.1 心跳聚合

所有 Worker 上报相同格式的心跳,API 端聚合:

```python
# backend/app/api/workers.py
@router.get("/api/workers")
async def list_workers():
    # 扫描所有 worker:* key
    keys = await redis.keys("worker:*")
    workers = []
    for key in keys:
        data = await redis.get(key)
        ttl = await redis.ttl(key)
        worker = json.loads(data)
        worker["online"] = ttl > 0
        workers.append(worker)

    # 按 worker_type 分组
    grouped = {}
    for w in workers:
        grouped.setdefault(w["worker_type"], []).append(w)
    return grouped
```

返回示例:
```json
{
  "wps-worker": [
    { "worker_id": "wps-01", "status": "busy", "current_task_id": "..." },
    { "worker_id": "wps-02", "status": "idle" }
  ],
  "stirling": [
    { "worker_id": "stir-01", "status": "busy" }
  ],
  "ocr-worker": [
    { "worker_id": "ocr-01", "status": "error" }
  ]
}
```

### 5.2 Prometheus 指标

所有 Worker 暴露相同指标格式:

```python
# Worker 端(通用 metrics 模块)
from prometheus_client import Counter, Histogram, Gauge

# 通用指标(所有 Worker 实现)
tasks_processed = Counter('worker_tasks_total',
    'Total tasks processed',
    ['worker_type', 'status'])
task_duration = Histogram('worker_task_duration_seconds',
    'Task duration',
    ['worker_type'])
queue_size = Gauge('queue_size', 'Queue size', ['queue_name'])
worker_online = Gauge('worker_online', 'Worker online',
    ['worker_type', 'worker_id'])
```

### 5.3 Grafana 面板

统一面板展示所有 Worker:
- 按 worker_type 分组的实时状态
- 各队列积压趋势
- 各 Worker 成功率对比
- 任务耗时分布

---

## 六、跨语言/跨平台考虑

### 6.1 Worker SDK

为不同语言提供统一 SDK,封装通用逻辑:

```
worker-sdk/
├─ python/          # Python Worker SDK(WPS/OCR/AI)
├─ java/            # Java Worker SDK(Stirling)
└─ node/            # Node Worker SDK(可选)
```

**Python SDK 示例**:
```python
# worker-sdk/python/zhiniao_worker/base.py
from zhiniao_worker import BaseWorker

class MyWorker(BaseWorker):
    worker_type = "my-worker"

    def process_task(self, task, input_path, output_path):
        # 子类只实现业务逻辑
        # 心跳/队列/状态/上传/下载 都由基类处理
        ...

if __name__ == "__main__":
    MyWorker().run()
```

**好处**:
- 新 Worker 只关心业务逻辑
- 通用逻辑(心跳/状态/重试)统一实现
- 升级 SDK 即升级所有 Worker

### 6.2 契约同步

不同语言从同一 schema 生成契约代码:

```
schemas/task-message.schema.json
    ├─→ Python: pydantic 模型
    ├─→ Java: DTO 类
    └─→ Node: TypeScript 接口
```

详见 [07-consistency-link.md](./07-consistency-link.md)。

---

## 七、容错与降级

### 7.1 Worker 故障处理

```python
# API 端定时扫描心跳
async def check_worker_health():
    for worker_key in redis.scan("worker:*"):
        ttl = redis.ttl(worker_key)
        if ttl < 0:  # 心跳过期
            worker = json.loads(redis.get(worker_key))
            if worker["current_task_id"]:
                # 该 Worker 有未完成任务,重新入队
                task = reconstruct_task(worker["current_task_id"])
                queue = WORKER_QUEUES[worker["worker_type"]]
                redis.lpush(queue, json.dumps(task))
                redis.delete(worker_key)  # 标记离线
                alert(f"Worker {worker['worker_id']} 离线,任务已重排")
```

### 7.2 降级链路

某类 Worker 故障时的降级:

```
WPS Worker 故障
    ↓
高质量转换不可用
    ↓
提示用户:"高质量版暂不可用,已为您降级到标准版"
    ↓
走 Stirling(LibreOffice) 兜底
```

### 7.3 熔断

某类 Worker 错误率过高时熔断:

```python
if error_rate(worker_type) > 0.3:  # 30% 错误率
    # 熔断 5 分钟,不再分配任务
    redis.setex(f"circuit:{worker_type}", 300, "open")
    # 已入队任务标记为 failed,允许重试到其他 Worker
```

---

## 八、部署拓扑示例

### 8.1 小规模(初期)

```
┌─ Linux VPS (4C8G) ────────────────┐
│ - API 服务                         │
│ - PostgreSQL / Redis / MinIO       │
│ - Stirling Worker × 2              │
│ - OCR Worker × 1                   │
│ - AI Worker × 1                    │
└────────────────────────────────────┘

┌─ Windows 机器 (本地) ──────────────┐
│ - WPS Worker × 1                   │
└────────────────────────────────────┘
```

### 8.2 中规模(增长期)

```
┌─ Linux VPS-A (API + 数据库) ───────┐
└────────────────────────────────────┘
┌─ Linux VPS-B (计算) ───────────────┐
│ - Stirling Worker × 5              │
│ - OCR Worker × 2                   │
└────────────────────────────────────┘
┌─ Windows 机器-A (WPS Worker) ──────┐
└────────────────────────────────────┘
┌─ Windows 机器-B (WPS Worker) ──────┐
└────────────────────────────────────┘
```

### 8.3 大规模(成熟期)

- K8s 集群跑 Linux Worker(自动扩缩容)
- Windows 机器池跑 WPS Worker(手动/半自动扩容)
- 数据库主从分离
- MinIO 分布式集群

---

## 九、与现有 WPS Worker 的集成

### 9.1 用户的 WPS Worker 在架构中的位置

```
schemas/tools.registry.json
  └─ pdf-to-word: backend="wps-worker"
       │
       ▼
API 路由层
  └─ 检测 backend="wps-worker"
       │
       ▼
LPUSH 到 queue:wps-worker
       │
       ▼
用户的 WPS Worker (BRPOP)
  ├─ 解析任务
  ├─ 下载输入(MinIO)
  ├─ 调用 kwpsconvert.exe
  ├─ 上传输出(MinIO)
  └─ 更新状态(Redis)
```

### 9.2 接入步骤

1. **调整队列名**: `queue:convert` → `queue:wps-worker`
2. **调整 bucket 名**: `pdf2word-input` → `zhiniao-input`
3. **补全心跳**: 实现心跳上报到 `worker:{id}`
4. **补全 task_id 字段**: 任务消息加 `tool_id`
5. **接入监控**: 暴露 Prometheus 指标

详见 [10-worker-integration.md](./10-worker-integration.md) 的 5.2 节。

### 9.3 短期兼容方案

不调整也能用,但建议在 API 层做适配:

```python
# API 端兼容旧队列名
LEGACY_QUEUE_MAP = {
    "queue:wps-worker": "queue:convert"  # 临时映射
}

# 启动时检测,旧名存在则同时入两个队列
async def enqueue(task, worker_type):
    new_queue = WORKER_QUEUES[worker_type]
    legacy_queue = LEGACY_QUEUE_MAP.get(new_queue)
    if legacy_queue and await redis.exists(legacy_queue):
        await redis.lpush(legacy_queue, json.dumps(task))
    else:
        await redis.lpush(new_queue, json.dumps(task))
```

---

## 十、关键结论

1. **多微服务通过 backend 字段路由**,单一来源是 `tools.registry.json`
2. **每类 Worker 独立队列**,故障隔离
3. **统一调度在 API 层**,根据 backend 字段分发
4. **独立扩缩容**,按队列积压决策
5. **统一监控**,所有 Worker 上报相同格式心跳
6. **跨语言用 SDK**,封装通用逻辑
7. **容错有降级链路**,WPS 故障可降级 Stirling
8. **用户现有 Worker 可无缝接入**,小幅调整命名即可

---

## 十一、相关文档

- [03-backend-architecture.md](./03-backend-architecture.md) — 后端架构
- [10-worker-integration.md](./10-worker-integration.md) — Worker 接入标准
- [07-consistency-link.md](./07-consistency-link.md) — 一致性链接
- [09-deployment.md](./09-deployment.md) — 部署方案
