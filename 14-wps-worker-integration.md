# WPS Worker 对接指南

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **状态**: 对接指南,基于用户已部署的 WPS Worker
> **依赖**: [10-worker-integration.md](./10-worker-integration.md), [11-microservice-architecture.md](./11-microservice-architecture.md)

---

## 一、用户现有 Worker 评估

用户已部署的 WPS Worker 基于 `kwpsconvert.exe` CLI,具备:

| 能力 | 现状 |
|------|------|
| WPS CLI 调用 | ✅ 已实现(kwpsconvert.exe) |
| 任务消息消费 | ✅ BRPOP `queue:convert` |
| MinIO 文件读写 | ✅ input/output bucket |
| 错误码体系 | ✅ E001-E099 |
| 配置文件 | ✅ worker_config.yaml |
| 心跳上报 | ❌ 需补充 |
| 多队列支持 | ⚠️ 需调整队列名 |
| tool_id 字段 | ⚠️ 需补充 |

---

## 二、对接调整清单

### 2.1 必须调整(影响功能)

| 项 | 现状 | 调整为 | 原因 |
|----|------|--------|------|
| 队列名 | `queue:convert` | `queue:wps-worker` | 多 Worker 类型区分 |
| Bucket 名 | `pdf2word-input` | `zhiniao-input` | 通用化(与 API 一致) |
| Bucket 名 | `pdf2word-output` | `zhiniao-output` | 同上 |
| 任务消息 | 无 tool_id | 加 `tool_id` 字段 | 一个 Worker 支持多工具 |

### 2.2 建议补充(增强可观测性)

| 项 | 现状 | 补充 | 原因 |
|----|------|------|------|
| 心跳上报 | 无 | `worker:{id}` Redis key | API 监控 Worker 状态 |
| 状态回写 | task:{id} Hash | 补全 started_at/completed_at | 标准化 |
| 截图上传 | 无 | 失败时上传到 `zhiniao-screenshots` | 诊断 |

### 2.3 短期兼容方案(不调整也能用)

API 层已实现兼容:任务入队时同时推送到新旧两个队列名(`backend/app/redis_client.py` 的 `LEGACY_QUEUE_MAP`)。用户现有 Worker 无需立即改动,但建议中期迁移到新命名。

---

## 三、对接实施步骤

### 3.1 第一步:确认 Worker 配置

用户现有 `config/worker_config.yaml` 需对齐以下字段:

```yaml
# 用户现有配置(示例)
worker:
  worker_id: "wps-worker-01"
  work_dir: "C:/wps-worker/work"

wps:
  cli_path: "C:/Users/.../kwpsconvert.exe"

redis:
  host: "<API服务器IP>"
  port: 6379
  password: "${REDIS_PASSWORD}"

minio:
  endpoint: "<API服务器IP>:9000"
  access_key: "${MINIO_ACCESS_KEY}"
  secret_key: "${MINIO_SECRET_KEY}"
  # ↓ 需调整为通用名(与 API 一致)
  input_bucket: "zhiniao-input"     # 原 pdf2word-input
  output_bucket: "zhiniao-output"   # 原 pdf2word-output
  screenshot_bucket: "zhiniao-screenshots"  # 新增

task:
  timeout: 300
  max_retry: 2
  heartbeat_interval: 10  # 新增
```

### 3.2 第二步:补充心跳上报

在 Worker 主循环中加心跳线程:

```python
import threading
import json
from datetime import datetime, timezone

class WPSWorker:
    def __init__(self, config):
        # ... 原有初始化 ...
        self.worker_id = config['worker']['worker_id']
        self.worker_type = "wps-worker"  # 新增
        self.processed_count = 0
        self.success_count = 0
        self.failed_count = 0
        self.current_status = "idle"
        self.current_task_id = None
        self.started_at = datetime.now(timezone.utc).isoformat()

    def heartbeat_loop(self):
        """心跳上报(每 10 秒)"""
        while True:
            info = {
                "worker_id": self.worker_id,
                "worker_type": self.worker_type,
                "ip": self._get_local_ip(),
                "version": "1.0.0",
                "wps_version": self._get_wps_version(),
                "status": self.current_status,
                "current_task_id": self.current_task_id,
                "processed_count": self.processed_count,
                "success_count": self.success_count,
                "failed_count": self.failed_count,
                "started_at": self.started_at,
                "last_heartbeat": datetime.now(timezone.utc).isoformat(),
            }
            self.redis.setex(
                f"worker:{self.worker_id}",
                30,  # TTL 30 秒
                json.dumps(info),
            )
            time.sleep(10)

    def start(self):
        # 启动心跳线程
        threading.Thread(target=self.heartbeat_loop, daemon=True).start()
        # 原有主循环
        self.main_loop()

    def _get_local_ip(self):
        import socket
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        try:
            s.connect(("8.8.8.8", 80))
            return s.getsockname()[0]
        except:
            return "127.0.0.1"
```

### 3.3 第三步:调整任务消费(支持多队列)

```python
def main_loop(self):
    """主循环 - 优先消费高优先级队列"""
    while True:
        # 优先级队列
        _, raw = self.redis.brpop("queue:wps-worker:priority", timeout=1)
        if not raw:
            # 兼容旧队列名(过渡期)
            _, raw = self.redis.brpop("queue:wps-worker", timeout=1)
            if not raw:
                _, raw = self.redis.brpop("queue:convert", timeout=30)  # 兼容
        if not raw:
            continue

        task = json.loads(raw)
        task_id = task["task_id"]
        self.current_status = "busy"
        self.current_task_id = task_id

        # 更新状态为 processing
        self.redis.hset(f"task:{task_id}", mapping={
            "status": "processing",
            "worker_id": self.worker_id,
            "started_at": datetime.now(timezone.utc).isoformat(),
        })

        try:
            # 原有转换逻辑
            result = self.process_task(task)
            self.processed_count += 1
            self.success_count += 1

            self.redis.hset(f"task:{task_id}", mapping={
                "status": "success",
                "completed_at": datetime.now(timezone.utc).isoformat(),
            })
        except Exception as e:
            self.failed_count += 1
            error_code = self._map_error_code(e)
            self.redis.hset(f"task:{task_id}", mapping={
                "status": "failed",
                "error_code": error_code,
                "error_msg": str(e),
                "completed_at": datetime.now(timezone.utc).isoformat(),
            })
            # 失败时上传截图
            self._upload_screenshot(task_id)

        self.current_status = "idle"
        self.current_task_id = None
```

### 3.4 第四步:工具路由(支持多工具)

用户的 `kwpsconvert.exe` 支持多种转换,根据 `task.tool_id` 路由:

```python
# 工具 ID → WPS CLI 命令映射
TOOL_COMMANDS = {
    "pdf-to-word": "pdf2word",
    "pdf-to-excel": "pdf2excel",
    "pdf-to-ppt": "pdf2ppt",
    "pdf-to-txt": "pdf2txt",
    "word-to-pdf": "word2pdf",
}

def process_task(self, task):
    tool_id = task["tool_id"]
    command = TOOL_COMMANDS.get(tool_id)
    if not command:
        raise ValueError(f"不支持的工具: {tool_id}")

    # 下载输入
    input_path = self._download(task["input_bucket"], task["input_key"])

    # 输出路径
    output_ext = ".docx" if "word" in tool_id else ".xlsx" if "excel" in tool_id else ".pdf"
    output_path = os.path.join(self.work_dir, f"{task['task_id']}{output_ext}")

    # 调用 WPS CLI
    options = task.get("options", {})
    cmd = [
        self.wps_cli_path, command,
        "-i", input_path,
        "-o", output_path,
        "--json",
    ]
    if options.get("ocr"):
        cmd.extend(["--scanned", "true", "--ai-fix", "true"])

    result = subprocess.run(cmd, capture_output=True, timeout=self.timeout)

    if result.returncode != 0:
        raise WPSConvertError(self._map_error_code(result.returncode))

    # 上传输出
    self._upload(task["output_bucket"], task["output_key"], output_path)

    return {"status": "success", "output": output_path}
```

---

## 四、Worker 接入测试

### 4.1 手动塞任务测试

```bash
# 在 API 服务器上塞一个测试任务
python scripts/test_enqueue_wps.py
```

测试脚本(`scripts/test_enqueue_wps.py`):

```python
"""手动塞任务测试 WPS Worker"""
import redis
import json
import uuid
from datetime import datetime, timezone

r = redis.Redis(host='localhost', port=6379, password='xxx', decode_responses=True)

task = {
    "task_id": str(uuid.uuid4()),
    "tool_id": "pdf-to-word",
    "input_bucket": "zhiniao-input",
    "input_key": "input/2026/07/02/test.pdf",  # 需先上传到 MinIO
    "output_bucket": "zhiniao-output",
    "output_key": "output/2026/07/02/test.docx",
    "quality": "premium",
    "options": {"ocr": True, "format": "docx"},
    "priority": 0,
    "created_at": datetime.now(timezone.utc).isoformat(),
    "expires_at": datetime.now(timezone.utc).isoformat(),
    "user_id": "test",
}

r.lpush("queue:wps-worker", json.dumps(task))
r.hset(f"task:{task['task_id']}", mapping={
    "status": "queued",
    "tool_id": "pdf-to-word",
    "created_at": task["created_at"],
})

print(f"任务已入队: {task['task_id']}")
print(f"查询: redis-cli HGETALL task:{task['task_id']}")
```

### 4.2 验证 Worker 心跳

```bash
# 在 API 服务器上
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" KEYS "worker:*"
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" GET worker:wps-worker-01
```

### 4.3 端到端验证

```bash
# 通过 API 创建任务(自动入队)
TOKEN=<登录后获取>
curl -X POST http://localhost:8000/api/convert \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@test.pdf" \
  -F "tool_id=pdf-to-word" \
  -F "quality=premium" \
  -F "options={\"ocr\": true}"

# 查询任务状态
curl http://localhost:8000/api/convert/<task_id> \
  -H "Authorization: Bearer $TOKEN"
```

---

## 五、Worker 部署步骤(Windows)

### 5.1 环境准备

```powershell
# 1. 安装 Python 3.11+
winget install Python.Python.3.11

# 2. 安装依赖
pip install redis minio pyyaml pillow

# 3. 安装 WPS Office 商业版(需 VIP 用于 PDF 转 Word)
# 手动安装,登录 VIP 账号

# 4. 找到 kwpsconvert.exe 路径
# 通常在: C:\Users\<用户名>\AppData\Local\Kingsoft\WPS Office\<版本>\office6\kwpsconvert.exe
```

### 5.2 部署 Worker 代码

```powershell
# 假设 Worker 代码在 C:\wps-worker\
cd C:\wps-worker

# 配置
Copy-Item config\worker_config.example.yaml config\worker_config.yaml
# 编辑 config\worker_config.yaml 填入 Redis/MinIO 地址
```

### 5.3 注册为服务(NSSM)

```powershell
# 下载 NSSM: https://nssm.cc/
nssm install zhiniao-wps-worker "C:\Python311\python.exe" "C:\wps-worker\worker.py"
nssm set zhiniao-wps-worker AppDirectory "C:\wps-worker"
nssm set zhiniao-wps-worker AppStdout "C:\wps-worker\logs\stdout.log"
nssm set zhiniao-wps-worker AppStderr "C:\wps-worker\logs\stderr.log"
nssm set zhiniao-wps-worker Start SERVICE_AUTO_START

nssm start zhiniao-wps-worker

# 查看状态
nssm status zhiniao-wps-worker
```

### 5.4 配置 autologon(如需桌面会话)

```powershell
# CLI 方案理论上不需要桌面会话,但部分 WPS 功能可能需要
# 配置开机自动登录
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" -Name "AutoAdminLogon" -Value "1"
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" -Name "DefaultUserName" -Value "worker"
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" -Name "DefaultPassword" -Value "<密码>"
```

### 5.5 屏蔽 WPS 广告(可选)

```powershell
# 编辑 hosts
Add-Content C:\Windows\System32\drivers\etc\hosts @"
127.0.0.1 ad.wps.cn
127.0.0.1 info.wps.cn
127.0.0.1 push.wps.cn
"@
```

---

## 六、监控 Worker

### 6.1 API 端查看

```bash
# 通过健康检查(包含在线 Worker 数)
curl http://localhost:8000/api/health

# 未来会有 /api/workers 端点查看详情
```

### 6.2 直接查 Redis

```bash
# 所有 Worker
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" KEYS "worker:*"

# 单个 Worker 详情
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" GET worker:wps-worker-01 | python -m json.tool

# 队列积压
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" LLEN queue:wps-worker
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" LLEN queue:wps-worker:priority
```

### 6.3 故障排查

| 现象 | 排查 |
|------|------|
| Worker 心跳消失 | 1. 查 Worker 进程 `nssm status` 2. 查日志 `logs.sh` 3. 查网络 |
| 任务一直 queued | 1. 查队列 `LLEN queue:wps-worker` 2. 查 Worker 是否在线 |
| 任务 failed | 1. 查 `HGETALL task:{id}` 看 error_code 2. 查截图 bucket |
| WPS VIP 失效 | 1. 检查 WPS 登录状态 2. 重新登录 VIP 账号 |

---

## 七、扩容 Worker

新增第二台 WPS Worker:

1. 在新 Windows 机器重复 5.1-5.3
2. 修改 `worker_id` 为 `wps-worker-02`
3. 启动后自动注册到 Redis,API 自动发现
4. 两台 Worker 同时 BRPOP,Redis 自动负载均衡

---

## 八、相关文档

- [10-worker-integration.md](./10-worker-integration.md) — Worker 接入标准
- [11-microservice-architecture.md](./11-microservice-architecture.md) — 多微服务架构
- [04-api-contract.md](./04-api-contract.md) — API 契约
- [13-operations.md](./13-operations.md) — 运维手册
