# 付费工具产品化通用规则(40 号)

> **版本**: v1.0
> **最后更新**: 2026-07-20
> **状态**: 基础设施已落地(扣费退款/进度用时/useConvertTask 升级/sweeper/三处对齐CI)
> **地位**: 每个付费/转换工具上线前**必过本清单**,不过不合并(12 号律三小步快跑)
> **关联**: [32-ui-design-spec.md](./32-ui-design-spec.md)(界面) · [12-implementation-playbook.md](./12-implementation-playbook.md)(P2门禁) · [27-lessons-learned.md](./27-lessons-learned.md)(#020/#027/#035) · [07-consistency-link.md](./07-consistency-link.md)(SSOT)
> **作者**: AI 协作产出

本文件解决"做出来是不能用的 demo,反复迭代耗时":把扣费保障/进度用时/结果展示/余额UX/错误码/SSOT 对齐固化成可执行清单,每个工具照做即成产品。

---

## 0. 适用范围
所有 `is_premium=true` 或 `backend` 为 worker 类(wps-worker/stirling/image-worker/ffmpeg-worker)的工具页。纯前端计算工具(id-photo/image-compress/resume-generator)遵守 §3/§4,扣费遵守 §1。

---

## 1. 扣费保障不变量(核心,后端已落地)

**不变量**: `success ↔ 净扣 1 点`;`failed/timeout/cancelled ↔ 净 0 点`;游客(user_id None)/standard 不扣不退。

- **预扣 + 失败退回**:入队时预扣 1 点(`convert.py` charge 日志)→ worker failed/timeout 退回(`workers/common/refund.py`)→ wps-worker 失败/崩溃由 `refund_sweeper` 兜底退。
- **取消退回(用户主动取消)**:`DELETE /api/convert/{id}` 取消 queued/processing 任务时,premium 退 1 点(action=`cancel_refund`)。queued/processing **都退**(用户没拿到结果↔净0点)。worker 在转换完成后、上传前查 cancelled——若已取消则丢弃结果不上传、不写 success(退款由 cancel 端点处理,worker 不重复退)。sweeper 兜底 cancelled 退款(防 cancel 端点退款失败)。
- **审计成对**:`convert_credit_logs` 表,`UNIQUE(task_id, action)` 幂等。charge/refund/refund_timeout/cancel_refund/recharge_after_late_success,可对账。
- **竞态补扣**:sweeper 退点标 timeout 后 worker 实际成功 → worker `recharge_for_late_success` 补扣 1 点(防白嫖)。
- **退款失败不阻断 worker**:worker 退款异常只记日志,sweeper 兜底。
- **前端取消入口**:`useConvertTask` 暴露 `cancel()` + `cancellable`(queued/processing 时 true);工具页显示"取消(退回次数)"按钮。用户卡住可取消且自动退费。

**验证清单**:造一个故意失败的 premium 转换 → 查 `balances.credits` 入队 -1、失败后 +1 → 查 `convert_credit_logs` 成对 charge+refund;造一个 premium 转换 processing 中调 DELETE 取消 → credits 退回 + 状态 cancelled。

---

## 2. 进度用时真实化(后端已落地)

- **进度**:读后端 Redis `progress` 真值(0→10 下载→30 转换→90 保存→100 完成)。**禁止前端 `setInterval` 假进度**。
- **阶段文案**:读后端 `stage`(排队中/下载文件中/转换中/保存结果中/完成),配 `32 号 §3.8` 审美。
- **用时**:读后端 `duration_ms`(worker success 写)。**禁止前端 `Date.now` 假算**(旧实现量入队耗时毫秒级→恒 0 秒)。
- **get_task 两路径一致**:Redis 快路径 / DB 回退都返回 `started_at/completed_at/duration_ms/file_size/progress/stage`。

**验证清单**:轮询中查 `progress` 单调递增(非 0→100)→ success 后 `duration_ms > 0` → 拔 Redis key 走 DB 回退,字段集一致。

---

## 3. 结果展示(产品级 vs demo 级)

demo 级 = 一个下载链接;产品级 = 结果卡片(状态+用时+文件信息)/对比预览/批量统计。每个转换工具 success 区至少:
- 下载按钮 + **用时**(`duration_ms` 换算秒)+ 文件信息(名/大小)
- 失败/超时标注"已自动退款"(让用户感知额度退回)
- 默认值兜底 + 空状态引导(拖拽区/格式说明,32 号 §3.8.2)

---

## 4. 余额 UX 统一

- **余额=0 拦截**(12 号 P2 门禁):premium 转换前 `useBalanceGuard`/`useConvertTask` 检查。
- **弹窗式**优于硬跳:`useConvertTask(opts.balanceGuard='dialog')` + 页渲染 `<InsufficientDialog>`,引导去 `/pricing` 购买次数(卡密兑换为备选)。
- 默认 redirect 跳 `/pricing`(在线购买次数为主渠道)。

---

## 5. upsell 引导

standard 成功后引导高质量(WPS+OCR),`pdf-to-word` 是样板(双档卡片 + "效果不理想?试试高质量")。高质量档是盈利锚点(38 号 §4.3)。

---

## 6. 错误码(错题 #020/#023)

- worker 内部用字符串错误码(E005/E006/E007/E008/E500),**不调 `raise_error`**,直接写 Redis/DB(无 fallback 风险)。
- API 侧新增错误码**先在 `error_codes.py` ERROR_CODES 定义再用**(`raise_error` fallback 是 E500,会掩盖未定义码)。
- 补码后 `grep -rn "raise_error|E5xx"` 核对所有调用点。E509(退款失败)/E510(超时已退款)已定义。

---

## 7. SSOT 对齐(07 号)

- **改契约先改 `openapi.yaml` → `npm run gen:api`**(get_task 返回字段变更要先落 openapi)。
- **三处对齐**(`scripts/check_worker_alignment.py`,挂 verify_all 第 13 节):`registry.backend ↔ WORKER_QUEUES ↔ file_validator.INPUT_TYPES`。worker 类 backend 的工具必须三处都有,否则上传 E404(#035)。
- **backend 须有消费者**(#027):registry 挂 backend=X → docker-compose 必须有对应 worker 服务。

---

## 8. 上线验证清单(12 号 P2 门禁)

- [ ] 余额=0 拦截(premium 被拒)
- [ ] 造故意失败 → credits 退回 + 审计成对(§1)
- [ ] 进度递增 + 用时>0(§2)
- [ ] 下载得正确文件
- [ ] 三处对齐 `check_worker_alignment.py` 过(§7)
- [ ] 该工具 worker 用到的 E-code 已定义(§6)
- [ ] 转换成功率连续 20 任务 ≥ 90%

---

## 9. 工具页统一改法模板

```
对每个 frontend/src/tools/<tool>/index.vue:
1. 已用 useConvertTask:解构出 progress/stage/duration;模板加进度条+stage;success 区加"用时 X秒";
   删任何 setInterval 假进度/Date.now 假用时。
2. 手写页(如 pdf-to-word):pollTask 读 task.progress/stage/duration_ms;删假进度假用时;余额硬跳可改弹窗。
3. 余额=0 拦截:确认 useBalanceGuard/useConvertTask 接入。
4. 后端:registry.backend↔WORKER_QUEUES↔INPUT_TYPES 三处对齐(check_worker_alignment.py 过)+ 有消费者。
5. 错误码:该 tool worker 用到的 E-code 已定义。
6. 跑 §8 验证清单。
```

---

## 10. 基础设施落地清单(本次,2026-07-20)

| 组件 | 位置 | 作用 |
|------|------|------|
| 审计表 | `backend/alembic/versions/0010_convert_credit_logs.py` | charge/refund 幂等审计 |
| 共享退款 | `workers/common/refund.py` | 三 worker 退款+补扣(行锁+UNIQUE幂等) |
| 三 worker | `workers/{stirling,image,ffmpeg}/worker.py` | 阶段进度/duration_ms/失败退款/success补扣 |
| 扣费入队 | `backend/app/api/convert.py` | charge 日志 + 入队 progress/stage |
| get_task | `backend/app/api/convert.py` | 两路径字段统一(started_at/stage/duration_ms) |
| sweeper | `backend/app/services/refund_sweeper.py` | wps-worker 失败/超时兜底退款(每5min) |
| useConvertTask | `frontend/src/composables/useConvertTask.ts` | progress/stage/duration + 余额弹窗 + onResult |
| openapi | `schemas/api/openapi.yaml` | ConvertTask 加 started_at/stage |
| 三处对齐 CI | `scripts/check_worker_alignment.py` | 防空队列/缺白名单(#035) |
| 错误码 | `backend/app/error_codes.py` | E509/E510 |
