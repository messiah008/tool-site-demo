# Stirling-PDF 接入指南

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **状态**: 接入指南
> **依赖**: [11-microservice-architecture.md](./11-microservice-architecture.md)

---

## 一、Stirling-PDF 的角色

Stirling-PDF 是开源的 PDF 处理工具(Linux 容器,Java),作为本项目的一个 Worker 类型,处理:

| 工具 | backend | Stirling API |
|------|---------|-------------|
| PDF 合并 | `stirling` | `/api/v1/general/merge` |
| PDF 拆分 | `stirling` | `/api/v1/general/split` |
| PDF 压缩 | `stirling` | `/api/v1/general/compress` |
| PDF 加水印 | `stirling` | `/api/v1/general/watermark` |
| PDF 解密 | `stirling` | `/api/v1/security/remove-password` |
| PDF 转图片 | `stirling` | `/api/v1/convert/file-to-img` |
| PDF OCR | `stirling` | `/api/v1/misc/ocr-pdf` |
| Excel 转 PDF | `stirling` | `/api/v1/convert/file-to-pdf` |
| PPT 转 PDF | `stirling` | `/api/v1/convert/file-to-pdf` |

---

## 二、部署架构

```
┌─ Linux VPS ─────────────────────────┐
│ - API 服务 (FastAPI)                │
│ - PostgreSQL / Redis / MinIO        │
│ - stirling-worker (Docker 容器)     │  ← 新增
└─────────────────────────────────────┘
```

Stirling Worker 是 Linux 容器,与 API 共用同一台 VPS,通过 Redis 队列通信。

---

## 三、部署步骤

### 3.1 添加到 docker-compose.yml

```yaml
services:
  stirling-worker:
    image: frooodle/s-pdf:latest
    restart: always
    environment:
      - DOCKER_ENABLE_SECURITY=false
      - LANGS=zh_CN
    volumes:
      - stirling-data:/configs
      - stirling-logs:/logs
      - ./schemas:/app/schemas:ro        # 共享 registry
      - ./scripts/stirling-worker:/app/scripts:ro
    depends_on:
      - redis
      - minio
    networks:
      - zhiniao-net

volumes:
  stirling-data:
  stirling-logs:
```

### 3.2 Stirling Worker 主循环

Worker 端用 Python 包装 Stirling 的 REST API:

```python
# scripts/stirling-worker/worker.py
import redis
import json
import httpx
import os
import time
import threading
from minio import Minio
from datetime import datetime, timezone

# Stirling API 命令映射
STIRLING_API_MAP = {
    "pdf-merge": ("/api/v1/general/merge", "multipart"),
    "pdf-split": ("/api/v1/general/split", "multipart"),
    "pdf-compress": ("/api/v1/general/compress", "multipart"),
    "pdf-watermark": ("/api/v1/general/watermark", "multipart"),
    "pdf-decrypt": ("/api/v1/security/remove-password", "multipart"),
    "pdf-to-image": ("/api/v1/convert/file-to-img", "multipart"),
    "pdf-ocr": ("/api/v1/misc/ocr-pdf", "multipart"),
}

class StirlingWorker:
    def __init__(self):
        self.redis = redis.Redis(
            host=os.getenv("REDIS_HOST", "redis"),
            port=6379, password=os.getenv("REDIS_PASSWORD"),
            decode_responses=True,
        )
        self.minio = Minio(
            os.getenv("MINIO_ENDPOINT", "minio:9000"),
            access_key=os.getenv("MINIO_ACCESS_KEY"),
            secret_key=os.getenv("MINIO_SECRET_KEY"),
            secure=False,
        )
        self.stirling_url = os.getenv("STIRLING_URL", "http://localhost:8080")
        self.worker_id = "stirling-01"
        self.worker_type = "stirling"
        self.queue_name = "queue:stirling"

    def heartbeat_loop(self):
        while True:
            info = {
                "worker_id": self.worker_id,
                "worker_type": self.worker_type,
                "status": "idle",
                "last_heartbeat": datetime.now(timezone.utc).isoformat(),
            }
            self.redis.setex(f"worker:{self.worker_id}", 30, json.dumps(info))
            time.sleep(10)

    def process_task(self, task):
        tool_id = task["tool_id"]
        if tool_id not in STIRLING_API_MAP:
            raise ValueError(f"不支持的工具: {tool_id}")

        api_path, _ = STIRLING_API_MAP[tool_id]

        # 下载输入文件
        input_path = f"/tmp/{task['task_id']}.pdf"
        self.minio.fget_object(task["input_bucket"], task["input_key"], input_path)

        # 调用 Stirling API
        with open(input_path, "rb") as f:
            response = httpx.post(
                f"{self.stirling_url}{api_path}",
                files={"fileInput": (os.path.basename(input_path), f, "application/pdf")},
                timeout=300,
            )
        response.raise_for_status()

        # 上传输出
        output_path = f"/tmp/{task['task_id']}.out"
        with open(output_path, "wb") as f:
            f.write(response.content)
        self.minio.fput_object(task["output_bucket"], task["output_key"], output_path)

        # 清理
        os.unlink(input_path)
        os.unlink(output_path)

    def run(self):
        threading.Thread(target=self.heartbeat_loop, daemon=True).start()
        while True:
            _, raw = self.redis.brpop(f"{self.queue_name}:priority", timeout=1)
            if not raw:
                _, raw = self.redis.brpop(self.queue_name, timeout=30)
            if not raw:
                continue

            task = json.loads(raw)
            task_id = task["task_id"]
            self.redis.hset(f"task:{task_id}", mapping={
                "status": "processing",
                "worker_id": self.worker_id,
                "started_at": datetime.now(timezone.utc).isoformat(),
            })

            try:
                self.process_task(task)
                self.redis.hset(f"task:{task_id}", mapping={
                    "status": "success",
                    "completed_at": datetime.now(timezone.utc).isoformat(),
                })
            except Exception as e:
                self.redis.hset(f"task:{task_id}", mapping={
                    "status": "failed",
                    "error_code": "E003",
                    "error_msg": str(e),
                    "completed_at": datetime.now(timezone.utc).isoformat(),
                })

if __name__ == "__main__":
    StirlingWorker().run()
```

### 3.3 Stirling Worker 配置

```env
# .env 追加
STIRLING_URL=http://stirling-worker:8080
```

---

## 四、API 路由验证

API 端已实现多队列路由(`backend/app/tool_registry.py`):

```python
WORKER_QUEUES = {
    "wps-worker": "queue:wps-worker",
    "stirling": "queue:stirling",      # ← Stirling 任务走这个队列
    "ocr-worker": "queue:ocr-worker",
    "ai-worker": "queue:ai-worker",
}
```

`tool.backend == "stirling"` 的工具,任务自动入 `queue:stirling`。

---

## 五、测试

### 5.1 启动 Stirling Worker

```bash
docker compose up -d stirling-worker
docker compose logs -f stirling-worker
```

### 5.2 手动测试

```bash
# 塞任务到 stirling 队列
python -c "
import redis, json, uuid
from datetime import datetime, timezone
r = redis.Redis(host='localhost', port=6379, decode_responses=True)
task = {
    'task_id': str(uuid.uuid4()),
    'tool_id': 'pdf-merge',
    'input_bucket': 'zhiniao-input',
    'input_key': 'input/test.pdf',
    'output_bucket': 'zhiniao-output',
    'output_key': 'output/test-merged.pdf',
    'quality': 'standard',
    'options': {},
    'created_at': datetime.now(timezone.utc).isoformat(),
    'user_id': 'test',
}
r.lpush('queue:stirling', json.dumps(task))
print(f'任务已入队: {task[\"task_id\"]}')
"

# 通过 API 创建任务(自动路由)
curl -X POST http://localhost:8000/api/convert \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@test.pdf" \
  -F "tool_id=pdf-merge" \
  -F "quality=standard"
```

### 5.3 验证心跳

```bash
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" GET worker:stirling-01
```

---

## 六、扩容

Stirling Worker 横向扩容:

```bash
docker compose up -d --scale stirling-worker=3
```

每个实例用不同 worker_id(通过环境变量):
```yaml
stirling-worker:
  environment:
    WORKER_ID: stirling-${HOSTNAME}
```

---

## 七、相关文档

- [10-worker-integration.md](./10-worker-integration.md) — Worker 接入标准
- [11-microservice-architecture.md](./11-microservice-architecture.md) — 微服务架构
- [13-operations.md](./13-operations.md) — 运维手册
