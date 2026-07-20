# 竞品分析 + 转换能力缺口与落地(38 号)

> **版本**: v1.1
> **最后更新**: 2026-07-16
> **状态**: stirling-worker P0 已落地(2026-07-16),12 工具后端接通;P1 补 11 前端页
> **依据**: 实地抓取 filetools.com.cn + 本地实测 Stirling-PDF v2.14.2(259 端点)
> **关联**: [01-progress.md](./01-progress.md)(Stirling 转换链路实测条目) + [CLAUDE.md](./CLAUDE.md) 工具矩阵路线图 P3(Stirling 集成,实测为假就绪) + [27-lessons-learned.md](./27-lessons-learned.md)(#027-#031) + [32-ui-design-spec.md](./32-ui-design-spec.md)
> **作者**: AI 协作产出
>
> **本文件解决两个问题**: ① 竞品「格式智造」(filetools.com.cn) 如何盈利、我们是否缺竞争力;② 用开源成熟方案补齐转换能力(免费增粘性、付费做盈利),含 stirling-worker 落地设计。

---

## 〇、TL;DR(先看结论)

1. **竞品「格式智造」靠 SEO 流量农场 + 后置变现**,免费声明为主、无定价页。它是四川小主体(蜀ICP备2025164628 + 个人邮箱 ycmail@189.cn)。
2. **我们不在同一赛道**: 它走"免费+广覆盖+标准质量",我们走"高质量付费+垂直"。硬碰格式广度我们必输,但 **WPS 高质量转换是我们独有护城河,它追不上**。
3. **最大缺口不是格式广度,而是 Stirling 集成从未接通**: registry 挂了 12 个 `backend=stirling` 工具,但**无任何队列消费者** → 任务永远卡在 queued。这是 P0。
4. **本地实测 Stirling v2.14.2**: latest 镜像**自带 unoserver(LibreOffice)**,Office 转换原生支持;259 端点里**无任何音频/视频端点**(audio-convert 是假工具,需独立 FFmpeg worker)。
5. **合规**: Stirling License = Open-Core MIT;**别碰 H.264**(音频用 MP3/FLAC/OGG 免专利,视频暂不做)。

---

## 一、竞品分析:格式智造(filetools.com.cn)

### 1.1 身份指纹(实证)

| 证据 | 内容 | 来源 |
|------|------|------|
| 站点 | 5 类导航(文档/图像/音频/视频/3D),副标题"兼容 190+/100+/100+/80+/50+ 种格式" | 截图 7 张 |
| 页面规模 | sitemap **386 个 URL**,几乎全是 `X-to-Y.html` 转换页,**无 /pricing /about** | 抓 sitemap.xml |
| UI 范式 | 4 源卡片(本地/URL/Dropbox/GoogleDrive) + 格式网格 + "点击转换文件" + "我的文件"角标 + "登录" | 截图 |
| 备案 | 蜀ICP备2025164628号 + 川公网安备51010802033094号 | 截图页脚 |
| 联系 | ycmail@189.cn(电信个人邮箱) | 截图 |
| 自述 | "免费""不使用广告或分析类追踪Cookie""仅供小文件体验,请勿上传大文件或批量处理" | 抓 tool-wps.html |

> 小主体 + 个人邮箱 + 全免费声明 → 四川小团队/个人开发者,非大公司。

### 1.2 它如何盈利(基于证据的判断)

证据三条:免费、无追踪 Cookie、禁止大文件/批量。sitemap 无定价/会员页。盈利模式按可能性排序:

1. **SEO 流量农场 + 后置变现**(最可能): 386 个 `pdf转word`/`heic转jpg`/`mp4转mp3` 长尾页各锁一个搜索词,先养流量,后续接广告/卖权重/转手。online-convert、123apps、bmcx 经典打法。
2. **登录后 freemium**: 有"登录","禁止大文件/批量"逼用户登录,登录后或有大文件/批量付费档,藏在账户内、首页不挂价。
3. **试水期未变现**: 小主体 + 全免费声明,可能处于"先把矩阵和 SEO 做起来"的早期。

> 它说"不使用追踪类 Cookie"≠ 绝对无广告,且更像给用户/备案看的隐私姿态。无论如何,**它没把"付费转换"摆到台面**,与我们不同。

### 1.3 竞争力对比

| 维度 | 格式智造 | 知鸟工具箱 | 谁赢 |
|------|----------|-----------|------|
| 格式转换广度 | 5类/数百格式 | 仅 4 个 PDF(pdf-to-word/excel/ppt/scanned) | 它碾压 |
| 转换质量定位 | 免费、标准质量(FFmpeg/LibreOffice级) | **高质量** PDF转Word(WPS+OCR) | **我们赢**(唯一真护城河) |
| 工具矩阵 | 纯转换 | 110 个(个税/房贷/黄历/成语等) | 我们赢(不同关键词池) |
| SEO 长尾 | 386 转换页 | 110 个多为计算/查询词 | 各占不同搜索池,正面冲突少 |
| 合规托管 | ✅ICP+公安双备案 | ❌域名/备案"待定" | **它赢,硬门槛** |
| 变现清晰度 | 模糊 | 清晰(付费转换+广告+卡密) | 各有取舍 |

**核心判断**:
- 泛格式转换赛道我们**严重缺竞争力**——它数百转换页吃长尾,我们只 4 个 PDF。
- **差异化是对的**: 它免费+广覆盖+标准质量,我们高质量付费+垂直。FFmpeg/LibreOffice 出的 PDF转Word 一塌糊涂(版式乱/表格丢/无 OCR),WPS 高质量是它给不了的。**质量这条线它追不上,我们不该拼它广度。**
- **真正风险是备案**: 它国内合法双备案,我们域名/备案仍"待定"。ICP 备案是中文转换站的生死线。

### 1.4 开源合规:它那套是不是开源?我们能抄吗?

**它能转这么多格式,后端几乎必然是公开开源组件拼的**(这才是"190+格式"真相):

| 转换类型 | 开源引擎 | 许可证 | 商用 |
|----------|----------|--------|------|
| 音频/视频/H.264 | FFmpeg | LGPL/GPL | ✅(子进程调用) |
| 图片(jpg/png/heic/psd) | ImageMagick/libvips | 各自宽松 | ✅ |
| 文档(doc/xls/ppt/ofd) | LibreOffice headless | MPL 2.0 | ✅ |
| 文档(md/epub/xml) | Pandoc | GPL-2 | ✅(子进程) |
| 3D模型(obj/gltf/stl) | Assimp | BSD | ✅ |
| PDF 全套 | **Stirling-PDF** | **Open-Core MIT** | ✅(实测,见下) |

**三件事分清**:
- ✅ **开源工具链本身可放心用**: 用同一批公开开源工具不叫"抄它",CloudConvert、online-convert 全世界都这么干,filetools 自己也是。我们项目已有 Stirling-PDF 基因(进度表"Fork Stirling-PDF"),方向一致。
- ⚠️ **不能抄代码/UI/品牌**: "格式智造"商标、其 UI 代码若有版权/商标不能照搬。但 4源卡片+格式网格是 CloudConvert 式通用范式,布局思路不受保护,按我们 32-ui-design-spec 实现即可。
- 🚨 **H.264 专利 ≠ FFmpeg 许可证**: 它声明"H.264/AVC 基于 FFmpeg 开源协议"是**误导性话术**——FFmpeg 开源(许可证免费)≠ H.264 编码格式本身被 MPEG-LA 专利池覆盖。它"不另行收专利费"是说它不向你收钱,**不等于它有专利授权**。**别照抄这句当合规背书**。
  - 做文档/图片转换基本不碰 H.264 专利。
  - 做视频输出 H.264 才踩线:走免专利编码(AV1/VP9)或"只转封装不编码"规避。**本计划暂不做视频。**

---

## 二、我们转换能力现状(实地核对,必读)

### 2.1 后端类工具真实状态

| 引擎 | 已挂工具 | 前端页? | 后端通? |
|------|----------|---------|---------|
| wps-worker(6) | pdf-to-word/excel/ppt, photo-restore, caj-to-pdf, batch-translate | ✅全有 | ✅(独立 Worker 进程,代码不在本仓库) |
| stirling(12) | pdf-merge/split/compress/ocr/decrypt/to-image/watermark/scanned, excel-to-pdf, ppt-to-pdf, ebook-convert, audio-convert | ❌**11个缺前端页** | ❌**全断**(见 2.2) |
| db-query(5) | id-card/phone/idiom/poetry/history-today | ✅ | ✅ |
| proxy(3) | ip/dns/whois | ✅ | ✅ |

### 2.2 🚨 硬伤:Stirling 链路从未接通(2026-07-16 实测发现)

追完整条链路:

```
用户上传 → convert.py → 入队 queue:stirling (Redis) → ❌ 无消费者 → 永远 queued
```

- `redis_client.py` 定义 `queue:stirling` 队列
- `convert.py` 确实把 stirling 工具任务推到 `queue:stirling`
- **全项目 grep 队列消费代码(brpop/blpop/consume) = 零结果**
- docker-compose 的 `stirling` 容器是 Stirling-PDF 的 **HTTP API 服务**,不是队列消费者
- (历史状态: 截至 2026-07-16 容器从未启动过,本文件首次 `docker compose up -d stirling` 起来)

**结论**: registry 挂 `backend=stirling` 的 12 个工具提交后**任务永远卡 queued,永不完成**——包括唯一有前端页的 pdf-to-scanned。这是"假就绪",不只是缺前端页。

> 进度文档 P3"Stirling+10个工具"一直待做,与本次实地核对一致。本文件把 P3 拆成可执行任务。

### 2.3 这正好是点3"补齐转换能力"的真正形态

补齐路径清晰: **写一个 stirling-worker(队列消费者)** = 从 queue:stirling 取任务 → 调 Stirling HTTP API → 结果存 MinIO → 回写 Redis 状态。一旦写出,**12 个已设计工具一次性全活**(前端页补 11 个即可)。性价比高于新增 image-format-convert 等。

---

## 三、Stirling-PDF 实测端点(2026-07-16,v2.14.2)

> **全部本地实测**,非文档推测。回归测试脚本: `scripts/test_stirling_endpoints.py`(改 Stirling 配置后重跑)。
> 实测环境: `docker compose up -d stirling`,容器内配 `SECURITY_CUSTOMGLOBALAPIKEY=zhiniao_stirling_key_2026`。

### 3.1 部署事实

| 项 | 实测结果 |
|----|----------|
| 镜像 | `frooodle/s-pdf:latest` = Stirling **v2.14.2** |
| License | **Open-Core MIT**(可商用,见 openapi `info.license`) |
| 默认端口 | 8080(仅内部 zhiniao-net,不暴露宿主) |
| **LibreOffice** | ✅ **latest 自带 unoserver**(日志 `Created LibreOffice temp directory` + `Starting unoserver on 127.0.0.1:2003`)→ Office 转换原生支持,**无需换 full/ultra 镜像** |
| 认证 | 新版默认 enableLogin,**API 必须带 `X-API-KEY`**(即使 DOCKER_ENABLE_SECURITY=false)→ 用 `SECURITY_CUSTOMGLOBALAPIKEY` 环境变量配全局 key |
| OpenAPI | `GET /v1/api-docs`(注意是 **v1** 非 v3) |
| 端点总数 | **259** |
| License 限制 | `requiresPaid=true, hasPaid=false` + `GRANDFATHERING LOCKED: 5 users` → SaaS 模式需付费 license,免费版锁 5 用户(⚠️ 商用需评估,见 §六) |

### 3.2 端点清单(转换相关 58 个,列关键项)

全部 POST、`multipart/form-data`、文件字段名 `fileInput`、认证头 `X-API-KEY`。

**转 PDF**:
- `/api/v1/convert/file/pdf` — 通用文件→PDF(LibreOffice/unoserver,**Excel/PPT/Word 转 PDF 走这里**) ✅实测200
- `/api/v1/convert/html/pdf` — HTML/ZIP→PDF ✅实测200
- `/api/v1/convert/markdown/pdf` — Markdown→PDF ✅实测200
- `/api/v1/convert/img/pdf` — 图片→PDF
- `/api/v1/convert/url/pdf` — URL→PDF(抓网页存档)
- `/api/v1/convert/svg/pdf` — SVG→PDF
- `/api/v1/convert/ebook/pdf` — 电子书→PDF ✅
- `/api/v1/convert/eml/pdf` — 邮件→PDF
- `/api/v1/convert/cbr|cbz/pdf` — 漫画→PDF

**PDF 转出**:
- `/api/v1/convert/pdf/word` — PDF→Word ✅实测200(`outputFormat=docx`)
- `/api/v1/convert/pdf/xlsx` — PDF→Excel(**⚠️ 实测 204 空输出**,需查参数)
- `/api/v1/convert/pdf/presentation` — PDF→PPT
- `/api/v1/convert/pdf/text` — PDF→Text/RTF ✅实测200(**必传 `outputFormat`**,否则 400)
- `/api/v1/convert/pdf/html` — PDF→HTML ✅实测200
- `/api/v1/convert/pdf/markdown` — PDF→Markdown
- `/api/v1/convert/pdf/epub` — PDF→EPUB/AZW3 ✅实测200
- `/api/v1/convert/pdf/csv` — PDF→CSV(**⚠️ 实测 204 空输出**)
- `/api/v1/convert/pdf/xml` — PDF→XML
- `/api/v1/convert/pdf/img` — PDF→图片 ✅实测200(**必传 `dpi`**,否则 500;多页返回 zip)
- `/api/v1/convert/pdf/pdfa` — PDF→PDF/A

**PDF 处理**:
- `/api/v1/general/merge-pdfs` — 合并
- `/api/v1/general/split-pages` — 拆分
- `/api/v1/general/split-by-size-or-count` — 按大小/数量拆
- `/api/v1/general/rotate-pdf` — 旋转
- `/api/v1/general/rearrange-pages` — 重排页
- `/api/v1/general/remove-pages` — 删页
- `/api/v1/misc/compress-pdf` — 压缩 ✅实测200
- `/api/v1/misc/scanner-effect` — 扫描效果(=pdf-to-scanned) ✅实测200
- `/api/v1/misc/ocr-pdf` — OCR(**⚠️ 实测 500**,ocrmypdf exit 6,需查 tesseract 中文语言包)
- `/api/v1/security/add-watermark` — 加水印
- `/api/v1/security/add-password` | `/remove-password` — 加解密
- `/api/v1/misc/extract-images` — 提取图片

### 3.3 实测暴露的端点细节坑(worker 实现必看)

| 端点 | 坑 | 正确用法 |
|------|----|----------|
| `pdf/img` | 不传 `dpi` → 500(`getDpi() is null`) | `-F dpi=150 -F imageFormat=png` |
| `pdf/text` | 不传 `outputFormat` → 400 | `-F outputFormat=txt`(或 rtf) |
| `pdf/word` | 需指定格式 | `-F outputFormat=docx` |
| `html/pdf` | 传 PDF 文件 → 400 | 传 .html 或 .zip |
| `file/pdf` | 传 PDF → 500;传 .md/.html/.docx → 200 | 传源文档(office/markdown/html) |
| `pdf/xlsx` `pdf/csv` | 实测 204 空输出 | 待查(可能需表格型 PDF 或额外参数) |
| `ocr-pdf` | 实测 500(ocrmypdf exit 6) | 查 tesseract 语言包是否装 `chi_sim` |
| `convert/pdf/img` | 多页 → 返回 **zip** | worker 按 zip 处理,非单文件 |

### 3.4 ❌ audio-convert 是假工具(实测结论)

259 端点 grep `audio|video|mp3|wav` = **零**。Stirling 是纯 PDF/文档工具,无任何音视频能力。registry 的 `audio-convert`(backend=stirling)是假工具,需:
- 要么改 `backend=ffmpeg-worker`(独立 FFmpeg worker,见 §五 P2)
- 要么先 deprecated 掉

---

## 四、转换能力全量缺口清单(双层质量分层)

> 分层策略(已定): **Stirling/开源 = 免费标准档(引流粘性),WPS = 付费高质量档(盈利)**。同一路由如 `/pdf-to-word` 免费走 Stirling、付费走 WPS。

### 4.1 第0波:已有后端、补前端页(stirling-worker 通了之后,零后端成本)

worker 一通,这 11 个工具后端即活,只需补前端页(走 SSOT:registry 已有→gen:tools→构建):

| 工具 | Stirling 端点 | 付费? | SEO | 成本 |
|------|---------------|-------|-----|------|
| pdf-merge | `/general/merge-pdfs` | 免费 | ⭐⭐⭐⭐ | 极低 |
| pdf-split | `/general/split-pages` | 免费 | ⭐⭐⭐⭐ | 极低 |
| pdf-compress | `/misc/compress-pdf` | 免费 | ⭐⭐⭐⭐⭐ | 极低 |
| pdf-to-image | `/convert/pdf/img` | 免费 | ⭐⭐⭐⭐ | 极低 |
| pdf-watermark | `/security/add-watermark` | 免费 | ⭐⭐⭐ | 极低 |
| pdf-decrypt | `/security/remove-password` | 免费 | ⭐⭐ | 极低 |
| excel-to-pdf | `/convert/file/pdf` | 免费 | ⭐⭐⭐⭐ | 极低 |
| ppt-to-pdf | `/convert/file/pdf` | 免费 | ⭐⭐⭐ | 极低 |
| ebook-convert | `/convert/pdf/epub`+`/convert/ebook/pdf` | 免费 | ⭐⭐⭐ | 极低 |
| pdf-ocr | `/misc/ocr-pdf` | 付费 | ⭐⭐⭐⭐ | 极低(先修 500) |
| pdf-to-scanned | `/misc/scanner-effect` | 免费 | ⭐⭐ | 已有页 |

### 4.2 第1波:Stirling 白送端点 → 新工具(对标 filetools 长尾,中等成本)

实测通过、registry 没有、可直接做:

| 新工具 | 端点 | 付费 | SEO | 成本 |
|--------|------|------|-----|------|
| word-to-pdf(免费档) | `/convert/file/pdf` | 免费 | ⭐⭐⭐⭐⭐ | 中 |
| pdf-to-text | `/convert/pdf/text` | 免费 | ⭐⭐⭐ | 低 |
| html-to-pdf | `/convert/html/pdf` | 免费 | ⭐⭐⭐⭐ | 中 |
| url-to-pdf | `/convert/url/pdf` | 免费 | ⭐⭐⭐ | 中 |
| markdown-to-pdf | `/convert/markdown/pdf` | 免费 | ⭐⭐ | 低 |
| pdf-to-html | `/convert/pdf/html` | 免费 | ⭐⭐ | 低 |
| pdf-to-epub | `/convert/pdf/epub` | 免费 | ⭐⭐ | 低 |
| image-to-pdf | `/convert/img/pdf` | 免费 | ⭐⭐⭐⭐ | 低 |
| pdf-to-ppt | `/convert/pdf/presentation` | 免费 | ⭐⭐⭐ | 低 |

### 4.3 第2波:独立开源引擎 → 新工具(对标 filetools 长尾,需新 worker)

| 新工具 | 引擎 | 付费 | SEO | 成本 | 专利 |
|--------|------|------|-----|------|------|
| **image-format-convert**(jpg/png/webp/heic互转) | ImageMagick/libvips | 免费 | ⭐⭐⭐⭐⭐ | 中 | 无 |
| audio-convert(真版) | FFmpeg | 免费 | ⭐⭐⭐ | 中 | **MP3/FLAC/OGG 免专利,AAC 有专利池** |
| doc/markdown 互转 | Pandoc | 免费 | ⭐⭐ | 低 | 无 |
| 3D模型转换 | Assimp | 免费 | ⭐⭐ | 中 | 无 |
| ~~video-convert~~ | ~~FFmpeg~~ | — | — | — | **🚫 暂不做(H.264 专利)** |

### 4.4 第3波:WPS 高质量付费档(盈利锚点)

延续 wps-worker 护城河,与免费档形成"质量对比":

| 工具 | 引擎 | 付费 | 说明 |
|------|------|------|------|
| pdf-to-word(高质量,已有) | wps-worker | ✅ | 核心卖点 |
| pdf-to-excel(高质量,已有) | wps-worker | ✅ | |
| pdf-to-ppt(高质量,已有) | wps-worker | ✅ | |
| word-to-pdf(高质量) | wps-worker | ✅ | 与 Stirling 免费档对比 |
| excel-to-pdf(高质量) | wps-worker | ✅ | |
| photo-restore(已有) | wps-worker | ✅ | |
| caj-to-pdf(已有) | wps-worker | ✅ | |

---

## 五、stirling-worker 详细设计(P0 核心)

### 5.1 架构(与现有 wps-worker 同构)

```
Redis (queue:stirling)
   │ brpop
   ▼
stirling-worker (Python 进程,新服务)
   │ 1. 取任务消息
   │ 2. 从 MinIO 拉输入文件(input_bucket/input_key)
   │ 3. 构造 multipart → POST Stirling HTTP API(X-API-KEY)
   │ 4. 收响应(单文件 or zip)→ 存 MinIO(output_bucket/output_key)
   │ 5. 回写 Redis task:{id} status=success/failed + 进度
   ▼
convert.py GET /api/convert/{task_id} 轮询状态 → download_url
```

### 5.2 服务定义(docker-compose 新增)

```yaml
  stirling-worker:
    build: ./workers/stirling
    restart: always
    environment:
      - REDIS_HOST=redis
      - REDIS_PASSWORD=${REDIS_PASSWORD}
      - MINIO_ENDPOINT=minio:9000
      - MINIO_ACCESS_KEY=${MINIO_ACCESS_KEY}
      - MINIO_SECRET_KEY=${MINIO_SECRET_KEY}
      - STIRLING_URL=http://stirling:8080
      - STIRLING_API_KEY=${STIRLING_API_KEY:-zhiniao_stirling_key_2026}
    depends_on:
      redis: { condition: service_started }
      minio: { condition: service_healthy }
      stirling: { condition: service_healthy }
    networks: [zhiniao-net]
```

### 5.3 端点路由表(SSOT: tool_id → Stirling 端点 + 参数)

`workers/stirling/route_map.py`(单一事实来源,worker 与 registry 共用):

```python
# tool_id -> (stirling_endpoint, form_builder)
# form_builder(input_path, options) -> list of (field, value) tuples
ROUTE = {
    "pdf-to-image":   ("/api/v1/convert/pdf/img", lambda o: [("dpi", o.get("dpi",150)), ("imageFormat", o.get("format","png"))]),
    "pdf-merge":      ("/api/v1/general/merge-pdfs", None),  # 多文件
    "pdf-split":      ("/api/v1/general/split-pages", None),
    "pdf-compress":   ("/api/v1/misc/compress-pdf", None),
    "pdf-ocr":        ("/api/v1/misc/ocr-pdf", lambda o: [("languages", o.get("lang","chi_sim+eng"))]),
    "pdf-decrypt":    ("/api/v1/security/remove-password", lambda o: [("password", o.get("password",""))]),
    "pdf-watermark":  ("/api/v1/security/add-watermark", None),
    "pdf-to-scanned": ("/api/v1/misc/scanner-effect", None),
    "excel-to-pdf":   ("/api/v1/convert/file/pdf", None),
    "ppt-to-pdf":     ("/api/v1/convert/file/pdf", None),
    "ebook-convert":  ("/api/v1/convert/pdf/epub", None),  # 或 /convert/ebook/pdf 看方向
    # audio-convert: 不在此表(Stirling 无音视频),见 §3.4
}
```

### 5.4 状态机(复用现有 task:{id} Redis 结构)

```
queued → processing → success
                   └→ failed (error_code/error_message)
超时:expires_at 到 → timeout
```

回写字段(与 convert.py 读取一致): `status/tool_id/quality/user_id/guest_token/created_at/started_at/completed_at/progress/error_code/error_msg/worker_id`。

### 5.5 消费循环(伪代码)

```python
while True:
    _, raw = redis.brpop("queue:stirling", timeout=30)  # 优先级队列另处理
    msg = json.loads(raw)
    task_id, tool_id = msg["task_id"], msg["tool_id"]
    update_status(task_id, status="processing", started_at=now, worker_id=WORKER_ID)
    try:
        in_file = minio_get(msg["input_bucket"], msg["input_key"])
        endpoint, builder = ROUTE[tool_id]
        fields = builder(msg["options"]) if builder else []
        resp = stirling_post(endpoint, in_file, fields)  # multipart + X-API-KEY
        out_bytes = resp.content  # 注意 zip 情况(pdf/img 多页)
        minio_put(msg["output_bucket"], msg["output_key"], out_bytes)
        update_status(task_id, status="success", completed_at=now, progress=100)
    except Exception as e:
        update_status(task_id, status="failed", error_code=map_err(e), error_msg=str(e), completed_at=now)
```

### 5.6 关键实现点(踩坑预防)

1. **zip 输出处理**: `pdf/img` 多页返回 zip,worker 需识别 Content-Type 或按端点约定存 zip(MinIO key 加 .zip)。
2. **超时**: Stirling 大文件/OCR 可能 >60s,curl 设 `--max-time 300`,worker 异步不阻塞队列。
3. **优先级队列**: convert.py 用 `{queue}:priority`,worker 需先 brpop priority 再 brpop 普通(见 redis_client.WORKER_QUEUES)。
4. **临时文件清理**: Stirling 容器自己清 /tmp,worker 侧下载的输入文件用完即删(防磁盘涨)。
5. **错误码映射**: Stirling 4xx/5xx → 映射到 error_codes.py E2xx 段(新增 stirling 专用码)。
6. **不要同步直调**: 别图省事改 convert.py 同步调 Stirling(大文件超时、阻塞 API 进程)——坚持异步队列,与 wps-worker 一致。

---

## 六、合规与成本红线

### 6.1 Stirling License 风险(⚠️ 必须评估)

实测日志 `License check: type=NORMAL, requiresPaid=true, hasPaid=false` + `GRANDFATHERING LOCKED: 5 users`。新版 Stirling 对 **SaaS 模式(对外提供转换服务)要求付费 license**,免费版锁 5 用户。

- **行动**: 上线前必须确认 Stirling v2.14.2 的 SaaS license 条款。**选项**: ① 购买 Stirling 商用 license;② 锁定用旧版(MIT 无此限制,但功能少);③ 评估是否触发"SaaS"定义(若仅自用/内部可能不触发)。
- **建议**: P0 用当前版本跑通验证,**商业化上线前**解决 license,**别上线后被告侵权**。记入 27 错题集待办。

### 6.2 开源合规总表

| 组件 | 许可证 | 合规要点 |
|------|--------|----------|
| Stirling-PDF | Open-Core MIT(但 SaaS 限 5 用户) | 见 6.1 |
| LibreOffice/unoserver | MPL 2.0 | 可商用,保留版权声明 |
| FFmpeg | LGPL/GPL | 子进程调用避传染;**H.264 专利另算** |
| ImageMagick | ImageMagick License(类 BSD) | 可商用 |
| Pandoc | GPL-2 | 子进程调用避传染 |

### 6.3 专利红线(再次强调)

- ✅ 文档/图片/3D 转换: 无专利问题
- ⚠️ 音频: MP3(专利已过期)/FLAC/OGG 免专利;**AAC 有专利池**,优先免专利编码
- 🚫 视频 H.264/AVC: **暂不做**。要做走 AV1/VP9 或"转封装不编码"
- ❌ **别照抄格式智造"H.264 基于 FFmpeg 开源协议"的话术**——那是姿态不是授权

---

## 七、落地路线(P0-P3)

> 遵循 [12-implementation-playbook.md](./12-implementation-playbook.md) 三律一循 8 步。每个工具走 SSOT: registry → gen:tools → 前端页 → 构建验证。

### P0:接通 Stirling(最高优先,解锁 12 工具)✅ 已落地 2026-07-16
1. [x] docker-compose 加 stirling-worker 服务(§5.2)+ STIRLING_API_KEY 变量化(stirling 与 worker 共用)
2. [x] 建 `workers/stirling/`(worker.py 消费循环+心跳+DB更新 + route_map.py 11 工具映射 + requirements + Dockerfile)
3. [x] 端点路由表 §5.3 落地(覆盖 11 个 stirling 工具,排除假工具 audio-convert)
4. [ ] 修复 pdf-ocr 500(查 tesseract chi_sim 语言包)——**待办**:需自建 stirling 镜像装语言包,worker 已优雅失败 E004 不阻塞
5. [x] 处理 pdf-to-image zip 输出(worker 按魔数 PK\x03\x04 识别,原样存 MinIO)
6. [x] 端到端:POST /api/convert pdf-compress → worker 消费 → 轮询 success → 下载 %PDF(全链路通,见 scripts/test_stirling_worker_e2e.py 2/2)
7. [x] 回归: `python3 scripts/test_stirling_endpoints.py` 10/10 + `test_stirling_worker_e2e.py` 2/2 + `test_stirling_unsupported.py` 假工具 E006 + verify_all 119/119

> P0 落地附带修复:convert.py expires_at 23点崩溃(#034)、file_validator 补 4 工具 INPUT_TYPES(#035)、.env SMTP 错位还原(#033);新坑 httpx multipart(#032)。详见 01-progress.md「stirling-worker P0 落地完成」+ 27 号 #032-#035

### P1:补 11 个前端页(零后端成本)
- [ ] pdf-merge/split/compress/to-image/watermark/decrypt/excel-to-pdf/ppt-to-pdf/ebook-convert/pdf-ocr 按 32-ui-design-spec 写前端页
- [ ] 每个配 seo 字段(33-seo-optimization.md,title/description/keywords/FAQ≥3/how_to/description_for_ai)
- [ ] gen:tools → 构建验证 → deploy_sync

### P2:第1波新工具(Stirling 白送端点)
- [ ] word-to-pdf / pdf-to-text / html-to-pdf / url-to-pdf / markdown-to-pdf / pdf-to-html / image-to-pdf 等
- [ ] 先加 registry(backend=stirling)→ route_map 加映射 → 前端页 → SEO 长尾页

### P3:独立引擎新工具(需新 worker)
- [ ] image-format-convert worker(ImageMagick,SEO 最高 ⭐⭐⭐⭐⭐)
- [ ] audio-convert 真版(FFmpeg,MP3/FLAC/OGG 免专利)
- [ ] 暂不做 video-convert

### P4:WPS 高质量档对比(盈利)
- [ ] word-to-pdf/excel-to-pdf 高质量档(wps-worker),与免费档 UI 上做"标准/高质量"选择

---

## 八、错题集(已记入 27 号 #027-#031)

- [x] **#027 Stirling 链路从未接通** → 27 号 #027(根因:convert.py 入队 queue:stirling 无消费者;修复:写 stirling-worker §五)
- [x] **#028 Stirling API 需 X-API-KEY 即使 SECURITY=false** → 27 号 #028(修复:配 SECURITY_CUSTOMGLOBALAPIKEY)
- [x] **#029 Stirling SaaS license 限 5 用户** → 27 号 #029(行动:商业化上线前解决,见 §6.1)
- [x] **#030 audio-convert 假工具** → 27 号 #030(Stirling 无音视频端点;修复:改 backend=ffmpeg-worker 或 deprecated)
- [x] **#031 deploy_sync --delete 删 WSL-only 脚本** → 27 号 #031(修复:脚本回写 Windows 权威侧)

---

## 九、关联文件清单

| 文件 | 作用 |
|------|------|
| `scripts/test_stirling_endpoints.py` | Stirling 端点回归测试(改配置后重跑) |
| `schemas/tools.registry.json` | 工具注册表(SSOT),stirling 工具的 backend 字段 |
| `backend/app/redis_client.py` | WORKER_QUEUES(queue:stirling 定义) |
| `backend/app/api/convert.py` | 任务入队逻辑(L155-200) |
| `workers/stirling/`(已落地) | stirling-worker 消费者(worker.py + route_map.py) |
| `docker-compose.yml`(已改) | 加 SECURITY_CUSTOMGLOBALAPIKEY + stirling-worker 服务 |
| `scripts/test_stirling_worker_e2e.py` | worker 端到端回归(POST→轮询→mc 验证 MinIO 输出) |
| `scripts/test_stirling_unsupported.py` | 假工具优雅失败回归(audio-convert → E006) |

---

> **下一步行动**: P0 已落地。下一步 **P1 补 11 个 stirling 工具前端页**(后端已活,走 SSOT:registry → gen:tools → 前端页 + seo 字段 → 构建验证)。商用上线前必解决 §6.1 Stirling SaaS license(#029);pdf-ocr tesseract chi_sim 待自建镜像(#P0.4)。
