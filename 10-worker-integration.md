# Worker 接入标准与解耦设计

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **状态**: 核心规范,所有 Worker 必须遵循
> **参考实现**: 用户的 WPS Worker(`kwpsconvert.exe` CLI 方案)

---

## 一、重大架构调整:CLI 方案替代 pywinauto

### 1.1 背景

之前的设计文档(`wps-pdf2word-task-B-worker.md`)基于 **pywinauto UI 自动化**,需要:
- Windows 交互式桌面会话(Session 0 隔离问题)
- 物理显示器或虚拟显示适配器
- autologon 配置
- WPS 升级界面改版会导致自动化失效

**实际部署的 WPS Worker** 用了 `kwpsconvert.exe`(WPS 官方 CLI),这是**更优方案**:

| 维度 | pywinauto UI 自动化 | kwpsconvert CLI(采用) |
|------|--------------------|-----------------------|
| 稳定性 | 低(依赖界面) | 高(官方工具) |
| 桌面会话 | 必须交互式 | 不需要 |
| 显示器 | 必须物理/虚拟 | 不需要 |
| 部署 | autologon + NSSM | 普通进程 |
| WPS 升级影响 | 界面改版即失效 | CLI 接口稳定 |
| JSON 输出 | 不支持 | 支持 |
| 多 Worker 并发 | 单实例串行 | 多进程并行 |
| 跨平台 | 仅 Windows | Windows(Wine 可试 Linux) |

### 1.2 调整后的 Worker 架构

```
┌──────────────────────────────────────────────────────┐
│ Windows Worker (普通进程,无需桌面会话)                │
│                                                      │
│  ┌─────────────┐    ┌──────────────┐    ┌────────┐ │
│  │ worker.py   │───▶│ wps_converter│───▶│ WPS    │ │
│  │ (主循环)    │    │ _cli.py      │    │ CLI    │ │
│  │             │    │ (封装层)     │    │kwpsconv│ │
│  └─────┬───────┘    └──────────────┘    └────────┘ │
│        │                                             │
│        │ BRPOP                                       │
└────────┼─────────────────────────────────────────────┘
         │
    ┌────▼────┐         ┌──────────┐
    │ Redis   │         │  MinIO   │
    │ 队列+状态│         │ 文件存储 │
    └─────────┘         └──────────┘
```

**关键改进**:
- Worker 是普通 Python 进程,不需要桌面会话
- 可以跑在 Windows Server Core、Windows 容器,甚至(实验性)Linux + Wine
- 多 Worker 实例并行运行,互不干扰

---

## 二、标准化保障机制

如何让不同工具的 Worker 保持标准化?通过**五个统一**:

### 2.1 统一任务消息格式

**单一事实来源**: `schemas/api/openapi.yaml` 的 `ConvertTask` schema

```json
{
  "task_id": "uuid",
  "tool_id": "pdf-to-word",
  "input_bucket": "zhiniao-input",
  "input_key": "input/2026/07/02/uuid.pdf",
  "output_bucket": "zhiniao-output",
  "output_key": "output/2026/07/02/uuid.docx",
  "quality": "premium",
  "options": {"ocr": true, "format": "docx"},
  "priority": 0,
  "created_at": "2026-07-02T10:00:00Z",
  "expires_at": "2026-07-02T11:00:00Z",
  "user_id": "uuid"
}
```

**所有 Worker 用同一格式解析**,不管它是 WPS Worker、Stirling Worker 还是 OCR Worker。

校验: `ajv validate -s schemas/task-message.schema.json -d <消息>`

### 2.2 统一错误码

**单一事实来源**: `schemas/error-codes.json`(参见 [04-api-contract.md](./04-api-contract.md))

| 错误码 | 含义 | 适用 Worker |
|--------|------|-----------|
| E001 | 启动失败 | WPS / OCR |
| E002 | 输入文件打开失败 | 所有 |
| E003 | 转换流程失败 | 所有 |
| E005 | 超时 | 所有 |
| E006 | 输出校验失败 | 所有 |
| E007 | 文件下载失败 | 所有 |
| E008 | 文件上传失败 | 所有 |
| E099 | 未知错误 | 所有 |

**用户的 WPS Worker 已采用此错误码体系**,一致性链接已生效。

### 2.3 统一存储协议

**单一事实来源**: [03-backend-architecture.md](./03-backend-architecture.md) 的 MinIO 章节

| Bucket | 用途 | 读写方 |
|--------|------|--------|
| `zhiniao-input` | 上传的原始文件 | API 写,Worker 读 |
| `zhiniao-output` | 转换结果 | Worker 写,API 读 |
| `zhiniao-screenshots` | 失败诊断 | Worker 写 |

**Key 命名规范**(所有 Worker 遵循):
```
{type}/{year}/{month}/{day}/{task_id}.{ext}
```

**用户 Worker 的现状**: 用 `pdf2word-input` / `pdf2word-output` bucket。
**调整建议**: 改为通用名 `zhiniao-input` / `zhiniao-output`,因为后续会有图片、文本等多种工具,不应按工具名分 bucket(否则 bucket 数量爆炸)。

### 2.4 统一状态机

**单一事实来源**: [04-api-contract.md](./04-api-contract.md) 的 `ConvertTask.status`

```
queued → processing → success
                  ↘ failed (重试2次后)
                  ↘ timeout
                  ↘ cancelled
```

**所有 Worker 必须实现这套状态转换**,通过 Redis Hash `task:{id}` 共享:

```python
# 通用状态更新函数(所有 Worker 复用)
def update_task_status(redis, task_id, status, **extra):
    redis.hset(f"task:{task_id}", mapping={
        "status": status,
        **extra,
        "updated_at": datetime.utcnow().isoformat()
    })
```

### 2.5 统一心跳协议

**单一事实来源**: [03-backend-architecture.md](./03-backend-architecture.md) 的 Worker 心跳章节

```json
{
  "worker_id": "wps-worker-01",
  "worker_type": "wps",
  "ip": "192.168.1.100",
  "version": "1.0.0",
  "status": "idle|busy|error|maintenance",
  "current_task_id": null,
  "processed_count": 123,
  "success_count": 118,
  "failed_count": 5,
  "last_heartbeat": "2026-07-02T10:00:00Z"
}
```

**所有 Worker 上报相同格式**,API 端用统一逻辑监控。

---

## 三、解耦保障机制

如何保证工具间无耦合、不互相影响?通过**五层隔离**:

### 3.1 消息驱动(不直接调用)

```
API 服务 ──(LPUSH 任务)──▶ Redis 队列 ──(BRPOP)──▶ Worker
       ◀──(HSET 状态)───                   ──(HSET)──▶
```

- API **不直接调用** Worker(无 HTTP/RPC)
- Worker **不直接回调** API(只写 Redis)
- 双方只与 Redis/MinIO 通信

**好处**:
- Worker 重启不影响 API
- API 重启不影响 Worker(任务在队列里等着)
- 网络分区时任务不丢(队列持久化)

### 3.2 独立部署(进程隔离)

每个 Worker 类型**独立部署、独立进程、独立配置**:

```
┌─────────────────────────┐
│ wps-worker (Windows)    │  独立机器/进程
│ - kwpsconvert.exe        │
│ - 独立 worker_config     │
└─────────────────────────┘

┌─────────────────────────┐
│ stirling-worker (Linux) │  Docker 容器
│ - Stirling-PDF Java     │
│ - 独立配置              │
└─────────────────────────┘

┌─────────────────────────┐
│ ocr-worker (Linux)      │  Docker 容器
│ - PaddleOCR             │
│ - 独立配置              │
└─────────────────────────┘
```

**好处**:
- 一个 Worker 崩溃,其他 Worker 不受影响
- 不同 Worker 用不同语言/平台(WPS 用 Windows+Python,Stirling 用 Linux+Java)
- 独立升级,不互相阻塞

### 3.3 共享契约(不共享代码)

Worker 之间**不共享代码**,但共享**契约定义**:

| 共享 | 不共享 |
|------|--------|
| `schemas/api/openapi.yaml` | 业务实现 |
| `schemas/task-message.schema.json` | 内部逻辑 |
| 错误码定义 | 配置文件 |
| 状态机定义 | 依赖库版本 |

**实现方式**: 每种 Worker 用各自语言生成契约代码:
- Python Worker: 用 `pydantic` 从 JSON Schema 生成模型
- Java Worker(Stirling): 用 `jsonschema2pojo` 生成 DTO
- Node Worker: 用 `json-schema-to-typescript`

### 3.4 存储隔离(不共享状态)

**Bucket 隔离**(可选,按需):
```
zhiniao-input/         # 所有工具共用输入
zhiniao-output/        # 所有工具共用输出
  ├─ pdf-to-word/      # 按工具分前缀(可选)
  ├─ image-compress/
  └─ ...
```

**队列隔离**(必须,按 Worker 类型):
```
queue:wps              # WPS Worker 消费
queue:stirling         # Stirling Worker 消费
queue:ocr              # OCR Worker 消费
queue:ai               # AI Worker 消费
```

**好处**:
- WPS Worker 挂了,Stirling 任务照常处理
- 某类任务积压,不影响其他类
- 可独立扩容某类 Worker

### 3.5 故障隔离

| 故障场景 | 影响范围 | 隔离机制 |
|---------|---------|---------|
| WPS Worker 崩溃 | 仅 WPS 任务 | 独立队列 + 心跳检测 |
| WPS License 失效 | 仅 WPS 任务 | 状态置 error,其他 Worker 正常 |
| Stirling 容器 OOM | 仅 Stirling 任务 | Docker 重启策略 |
| Redis 故障 | 全局(降级) | RDB 持久化 + 哨兵 |
| MinIO 故障 | 全局 | 集群模式 |
| API 服务重启 | 任务在队列等待 | 队列持久化 |

---

## 四、Worker 接入标准(新工具如何接入)

新增一个工具的 Worker,按此模板:

### 4.1 步骤

1. **在 registry 标注 backend 类型**:
```json
// schemas/tools.registry.json
{
  "id": "pdf-to-excel",
  "backend": "wps-worker",   // 复用已有 Worker
  // 或
  "backend": "ocr-worker",   // 新 Worker 类型
}
```

2. **如果复用已有 Worker**:只需在 Worker 的工具映射表加一行
```python
# wps_worker/tool_handlers.py
TOOL_HANDLERS = {
    "pdf-to-word": handle_pdf_to_word,
    "pdf-to-excel": handle_pdf_to_excel,   # 新增
}
```

3. **如果是新 Worker 类型**:按 4.2 模板实现

### 4.2 新 Worker 类型模板

```python
# my_worker/worker.py
import redis, json, os
from minio import Minio
from datetime import datetime

class MyWorker:
    def __init__(self, config):
        self.redis = redis.Redis(**config['redis'])
        self.minio = Minio(**config['minio'])
        self.worker_id = config['worker_id']
        self.worker_type = "my-worker"   # ★ 标识 Worker 类型
        self.queue_name = f"queue:{self.worker_type}"  # ★ 独立队列

    def heartbeat_loop(self):
        """统一心跳(所有 Worker 必须实现)"""
        while True:
            info = {
                "worker_id": self.worker_id,
                "worker_type": self.worker_type,
                "status": self.current_status,
                "current_task_id": self.current_task_id,
                "processed_count": self.processed_count,
                "success_count": self.success_count,
                "failed_count": self.failed_count,
                "last_heartbeat": datetime.utcnow().isoformat()
            }
            self.redis.setex(f"worker:{self.worker_id}", 30, json.dumps(info))
            time.sleep(10)

    def process_task(self, task):
        """工具特定逻辑(子类实现)"""
        raise NotImplementedError

    def run(self):
        """统一主循环(所有 Worker 复用)"""
        while True:
            _, raw = self.redis.brpop(self.queue_name, timeout=30)
            if not raw: continue

            task = json.loads(raw)
            task_id = task['task_id']

            # 检查过期
            if self._is_expired(task['expires_at']):
                self._update_status(task_id, "failed", error_code="E099",
                                    error_msg="任务过期")
                continue

            # 下载输入
            input_path = self._download(task)
            output_path = os.path.join(self.work_dir, f"{task_id}.out")

            # 处理
            self._update_status(task_id, "processing",
                               worker_id=self.worker_id,
                               started_at=datetime.utcnow().isoformat())
            try:
                result = self.process_task(task, input_path, output_path)
                self._upload(task, output_path)
                self._update_status(task_id, "success",
                                    completed_at=datetime.utcnow().isoformat())
            except Exception as e:
                self._handle_failure(task, e)

    # 通用辅助方法(所有 Worker 复用)
    def _update_status(self, task_id, status, **extra): ...
    def _download(self, task): ...
    def _upload(self, task, path): ...
    def _handle_failure(self, task, error): ...
    def _is_expired(self, expires_at): ...
```

### 4.3 校验清单

新 Worker 接入前自检:

- [ ] 消费正确的队列 `queue:{worker_type}`
- [ ] 任务消息符合 `task-message.schema.json`
- [ ] 错误码使用 `E001-E099` 体系
- [ ] 实现心跳上报到 `worker:{id}`
- [ ] 输出文件上传到 `zhiniao-output` bucket
- [ ] 失败时上传截图到 `zhiniao-screenshots`(可选)
- [ ] 状态更新符合统一状态机

---

## 五、用户的 WPS Worker 接入评估

按上述标准评估用户已部署的 WPS Worker:

### 5.1 符合的部分

| 标准 | 用户 Worker | 状态 |
|------|-----------|------|
| 任务消息格式 | `{task_id, input_bucket, input_key, output_bucket, output_key, options}` | ✅ 符合 |
| 错误码体系 | E001/E002/E005/E006/E099 | ✅ 符合 |
| Redis 队列 | `queue:convert` + BRPOP | ⚠️ 队列名需调整 |
| MinIO 存储 | input/output bucket 分离 | ⚠️ bucket 名需调整 |
| 配置文件 | `worker_config.yaml` | ✅ 合理 |

### 5.2 需调整的部分

| 项 | 现状 | 建议调整 | 原因 |
|----|------|---------|------|
| 队列名 | `queue:convert` | `queue:wps-worker` | 多 Worker 类型时区分 |
| Bucket 名 | `pdf2word-input` | `zhiniao-input` | 通用化,支持多工具 |
| 任务消息 | 缺 `tool_id` | 加 `tool_id` 字段 | 一个 Worker 支持多工具 |
| 心跳 | 未实现 | 加心跳上报 | API 监控 Worker 状态 |
| 状态机 | task:{id} Hash | 补全状态字段 | 标准化 |

### 5.3 调整后的配置

```yaml
# config/worker_config.yaml (调整后)
worker:
  worker_id: "wps-worker-01"
  worker_type: "wps-worker"           # ★ 新增,标识类型
  queue_name: "queue:wps-worker"      # ★ 独立队列
  work_dir: "C:/wps-worker/work"
  log_dir: "C:/wps-worker/logs"

wps:
  cli_path: ".../kwpsconvert.exe"

redis:
  host: "redis.internal"
  port: 6379
  password: "${REDIS_PASSWORD}"

minio:
  endpoint: "minio.internal:9000"
  access_key: "${MINIO_ACCESS_KEY}"
  secret_key: "${MINIO_SECRET_KEY}"
  input_bucket: "zhiniao-input"       # ★ 通用名
  output_bucket: "zhiniao-output"     # ★ 通用名
  screenshot_bucket: "zhiniao-screenshots"

# 工具映射(一个 Worker 支持多工具)
tool_handlers:
  pdf-to-word: "pdf2word"
  pdf-to-excel: "pdf2excel"
  pdf-to-ppt: "pdf2ppt"
  word-to-pdf: "word2pdf"

task:
  timeout: 300
  max_retry: 2
  heartbeat_interval: 10
```

### 5.4 兼容性说明

由于用户的 Worker 已实际部署,**调整应渐进**:
1. **短期**: 保留现有 bucket/队列名,在 API 层做适配
2. **中期**: 新增工具时使用新命名,旧名共存
3. **长期**: 统一到新命名,下线旧名

---

## 六、与一致性链接的关系

Worker 的标准化通过一致性链接保障:

```
schemas/api/openapi.yaml ──→ ConvertTask schema ──→ 所有 Worker 解析任务用
schemas/error-codes.json ──→ 错误码定义 ──→ 所有 Worker 上报错误用
schemas/tools.registry.json ──→ backend 字段 ──→ API 路由到对应队列
```

**任何 schema 变更**,所有 Worker 重新生成代码,保持一致。详见 [07-consistency-link.md](./07-consistency-link.md)。

---

## 七、关键结论

1. **用户的 CLI 方案优于 pywinauto**,架构已调整采用 CLI
2. **标准化通过五个统一**:任务消息/错误码/存储/状态机/心跳
3. **解耦通过五层隔离**:消息驱动/独立部署/共享契约/存储隔离/故障隔离
4. **新 Worker 接入有标准模板**,见 4.2
5. **用户现有 Worker 基本符合**,需小幅调整命名和补全心跳

详见 [11-microservice-architecture.md](./11-microservice-architecture.md) 了解多 Worker 类型的架构设计。
