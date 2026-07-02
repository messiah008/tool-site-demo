# 迭代进度记录

> **最后更新**: 2026-07-02
> **当前版本**: v0.2.1
> **阶段**: 原型期(Phase 0-1)

---

## 版本历史

### v0.2.1 (2026-07-02) - Worker 架构调整 + 实施流程总纲

**重大变更 1**: WPS Worker 方案从 pywinauto UI 自动化改为 `kwpsconvert.exe` CLI

**背景**: 用户提供已实际部署的 WPS Worker,使用 WPS 官方 CLI 工具(`kwpsconvert.exe`),比之前设计的 pywinauto UI 自动化更稳定可靠。

**优势对比**:
| 维度 | pywinauto(旧) | CLI(新) |
|------|--------------|---------|
| 桌面会话 | 必须交互式 | 不需要 |
| 显示器 | 必须 | 不需要 |
| 稳定性 | 依赖界面 | 官方工具 |
| 部署 | autologon+NSSM | 普通进程 |
| 多 Worker 并发 | 单实例串行 | 多进程并行 |

**新增文档**:
- `10-worker-integration.md` — Worker 接入标准与解耦设计
- `11-microservice-architecture.md` — 多微服务架构设计
- `12-implementation-playbook.md` — 实施流程总纲(落地+迭代+AI一致性)

**标准化机制(五个统一)**:
1. 统一任务消息格式(基于 OpenAPI schema)
2. 统一错误码(E001-E099,跨工具通用)
3. 统一存储协议(MinIO bucket 命名规范)
4. 统一状态机(queued → processing → success/failed)
5. 统一心跳协议(worker:{id} Redis key)

**解耦机制(五层隔离)**:
1. 消息驱动(API 不直接调用 Worker)
2. 独立部署(每类 Worker 独立进程)
3. 共享契约(不共享代码)
4. 存储隔离(按 Worker 类型分队列)
5. 故障隔离(一类故障不影响其他)

**多微服务设计**:
- 5 大 Worker 类型: wps-worker / stirling / ocr-worker / ai-worker / frontend
- API 根据 `tool.backend` 字段路由到对应队列
- 每类 Worker 独立队列、独立扩缩容、统一监控

**用户现有 Worker 评估**:
- ✅ 任务消息格式、错误码体系符合标准
- ⚠️ 队列名需从 `queue:convert` 调整为 `queue:wps-worker`
- ⚠️ Bucket 名需从 `pdf2word-input` 调整为 `zhiniao-input`
- ⚠️ 需补全心跳上报和 tool_id 字段

### v0.2.0 (2026-07-02) - 卡密兑换闭环

**新增**:
- 卡密兑换页 `redeem.html`
- 用户中心页 `user-center.html`
- PDF转Word 页接入余额检查逻辑
- 首页导航加入"兑换卡密/我的账户"入口
- 完整业务闭环:闲鱼购买 → 卡密兑换 → 高质量转换 → 余额扣减

**演示流程**:
1. `redeem.html` 输入 `ZN-DEMO-2026-TEST` → 余额 100 次
2. 跳转 `pdf-to-word.html` 上传 PDF,选高质量转换
3. 余额扣减到 99,转换完成
4. `user-center.html` 查看记录

**测试卡密**:
- `ZN-DEMO-2026-TEST` (100 次)
- `ZN-ABCD-1234-EFGH` (50 次)

### v0.1.0 (2026-07-02) - 静态原型首版

**新增**:
- 首页 `index.html`(工具目录,5 大分类 30+ 工具卡片)
- PDF转Word 工具页 `pdf-to-word.html`
- 标准/高质量双层质量选择
- 免费转换后付费引导卡片

**设计决策**:
- 借鉴 bmcx.com 的子域名+极简交互+矩阵互链
- 不借鉴 bmcx 的全免费+广告密集模式
- 采用紫色调(#4f46e5)作为主色

### v0.0.1 (2026-07-02) - 战略转向

**决策**: 从"从零自建"转为"Fork Stirling-PDF + WPS Worker 增强"
- 理由: 缩短开发周期,工具矩阵引流
- Stirling-PDF 提供 50+ 工具 + REST API
- WPS Worker 作为高质量付费后端

---

## 当前状态

### 已完成
- [x] 战略方案确定(Fork Stirling-PDF)
- [x] 静态原型 4 个页面
- [x] 卡密兑换闭环演示
- [x] 设计文档全套(本目录)

### 进行中
- [ ] Vue 工程化(原型转 Vue3)
- [ ] 后端 API 实现
- [ ] WPS Worker 开发(已有设计文档)

### 待开始
- [ ] 支付接入
- [ ] 公网上线
- [ ] 工具矩阵扩展

---

## 下一阶段计划 (v0.3.0)

### 目标: Vue 工程化 + 后端骨架

**任务**:
1. 用 Vite 创建 Vue3 + TS 工程
2. 把 4 个静态原型转成 Vue 组件
3. 引入 Design Tokens(从 schemas/design-tokens.json 生成)
4. 实现工具注册表自动生成路由和卡片
5. 搭建 FastAPI 后端骨架
6. 实现 OpenAPI 契约定义的接口
7. 实现卡密生成/校验 API

**预计周期**: 2 周

---

## 关键决策记录

| 日期 | 决策 | 理由 | 影响范围 |
|------|------|------|---------|
| 2026-07-02 | Fork Stirling-PDF 而非从零自建 | 缩短周期,白嫖 50+ 工具 | 全局 |
| 2026-07-02 | WPS Worker 作为高质量付费后端 | 中文 OCR 优势,差异化 | Worker |
| 2026-07-02 | 卡密兑换机制对接闲鱼 | 闲鱼是主要流量来源 | 前端+后端 |
| 2026-07-02 | 采用一致性链接(SSOT+代码生成) | 多 AI 协作一致性 | 全局 |
| 2026-07-02 | WPS Worker 改用 CLI(kwpsconvert) 替代 pywinauto | 官方工具更稳定,无需桌面会话 | Worker |

---

## 待解决问题

| 问题 | 优先级 | 状态 | 备注 |
|------|--------|------|------|
| WPS 商业版 License 报价 | 高 | 待询价 | 联系金山销售 |
| Windows Worker 机器来源 | 高 | 待定 | 本地/云桌面 |
| 域名注册 | 中 | 待定 | .com/.cn |
| 支付通道资质 | 中 | 待定 | 个人/企业 |
| 备案 | 中 | 待定 | 国内服务器需要 |

---

## 文档变更记录

| 日期 | 文档 | 变更 |
|------|------|------|
| 2026-07-02 | 全部 | 初始创建 |
| 2026-07-02 | 10/11/12 + README/CLAUDE/01 | Worker 架构调整 + 实施流程总纲 |
