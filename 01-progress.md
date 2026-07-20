# 迭代进度记录

> **最后更新**: 2026-07-16
> **当前版本**: v1.5.1 (新增 38-竞品分析+Stirling实测,暴露 stirling-worker P0 硬伤)
> **上一版本**: v1.5.0 (新增 36-用户支付卡密三模块补全方案)
> **阶段**: 39 个工具按新规范重写,3 大业务模块补全方案已设计完成

---

## 🔄 当前状态(实时更新,防止记忆丢失)

**正在执行**: 补全 WPS 工具完成 ✅,累计 74 个工具

**最新(2026-07-16)**: 🚨 Stirling 转换链路实测发现硬伤——12 个 `backend=stirling` 工具**无队列消费者**(任务永远卡 queued),P0 需写 stirling-worker 接通。详见下方「竞品分析 + Stirling 转换链路实测」+ [38-competitor-and-conversion-gap.md](./38-competitor-and-conversion-gap.md)

**WPS 工具补全**(v1.0.0-rc2,基于 wps实现功能.png 截图):
- pdf-to-excel PDF 转 Excel(¥2,WPS 引擎,表格还原)
- pdf-to-ppt PDF 转 PPT(¥3,WPS 引擎,页转幻灯片)
- pdf-to-scanned 转扫描型 PDF(免费,Stirling,文字变图片防复制)

**已实现工具**(71 个,v0.9.0):

第 1 批(30 个,纯前端):
- PDF/付费(1): pdf-to-word
- 电脑网络/加密(11): random-password/uuid-gen/qrcode/md5/sha/hmac/aes/file-hash/base64/url/html-escape
- 文本工具(6): json-format/text-count/case-convert/text-dedup/text-diff/regex-test
- 计算工具(7): timestamp-convert/base-convert/unit-convert/storage-unit/bmi/color-convert/morse
- 时间工具(3): online-alarm/countdown-timer/world-clock
- 符号收集(2): special-symbols/emoji-collection

第 2 批(30 个,纯前端+静态数据):
- 日常生活(5): lunar-calendar万年历/solar-terms节气/zodiac生肖/age-calculator年龄/constellation星座
- 网站建设(5): html-format/css-format/js-format/sql-format/http-status
- 其他计算(2): placeholder-image占位图/chinese-number数字转中文大写
- 文化数据(8): periodic-table元素周期表/hundred-surnames百家姓/birthday-code生日密码/twenty-eight-stars二十八星宿/dynasties历史朝代/iching易经/luban-ruler鲁班尺/birthday-book生日书
- 趣味文本(5): couplets对联/riddles谜语/brain-teasers脑筋急转弯/famous-quotes名人名言/two-part-allegorical歇后语
- 互动游戏(5): reaction-test反应测试/lottery抽奖/random-number-gen随机数/user-agent查看/mind-reading读心术

第 3 批(5 个,数据库查询,backend=db-query):
- id-card-region 身份证归属查询(tool_id_cards 表,320 条)
- phone-region 手机归属查询(tool_phone_numbers 表,62 条)
- idiom-dict 成语大全(tool_idioms 表,45 条)
- poetry 诗词大全(tool_poetry 表,57 条)
- history-today 历史上的今天(tool_history_today 表,26 条)

第 4 批(3 个,后端代理,backend=proxy):
- ip-query IP 地址查询(代理 ip-api.com,返回归属/运营商/经纬度)
- dns-query DNS 查询(代理 Cloudflare DoH,支持 A/AAAA/MX/TXT/CNAME/NS)
- whois-query WHOIS 查询(占位,待接入付费 API)

第 5 批(3 个,付费工具,is_premium=true):
- id-photo 证件照制作(Canvas 换底色+裁剪,¥2/次)
- image-compress 图片批量压缩(browser-image-compression,¥2/次)
- resume-generator 简历模板生成(填写+实时预览+打印,¥5/次)

**deprecated**(2): timestamp/base64(被新工具替代)

**验证结果**(v0.9.0):
- ✅ registry 95 个工具(93 活跃 + 2 deprecated),一致性校验通过
- ✅ 前端构建成功(8.21s),71 个工具页各自独立 chunk
- ✅ Nginx 部署后 71 个路由全部 HTTP 200
- ✅ 数据库扩充:身份证 320/手机 62/成语 45/诗词 57/历史 26 条
- ✅ 代理 API 工作正常(IP 查 8.8.8.8→美国 Google,DNS 查 baidu.com→A 记录)
- ✅ 付费工具余额扣减逻辑就绪(检查余额→不足跳转兑换页)
- ✅ 工具路由+渲染验证 77/77 通过(100%)
- ✅ 数据库/代理功能验证 7/7 通过(100%)

**验证用例**(v0.9.0 新增,B 任务完成):
- schemas/tests/ 目录 — 77 个工具的测试用例 JSON
- 31 个工具含功能验证用例(输入+断言)
- scripts/verify_tools.py v2 — 支持读取测试用例执行功能验证
- 断言类型:json_path/json_path_gte/list_not_empty/contains/equals/route_ok
- 用法: `python scripts/verify_tools.py --functional`(db/proxy 自动验证,frontend 标记需浏览器)

**待执行**(对齐 22 号文档 12 周计划):
- [x] 第 1 批:30 个纯前端工具 ✅
- [x] 第 2 批:30 个纯前端+静态数据 ✅
- [x] 第 3 批:5 个数据库工具 ✅(数据已扩充至生产规模)
- [x] 第 4 批:3 个后端代理工具 ✅(IP/DNS/WHOIS)
- [x] 付费工具:证件照/图片压缩/简历 ✅(第 5 批)
- [x] 工具验证:77 个测试用例已生成 ✅(B 任务完成)
- [x] 上线准备:检查清单+生产配置+部署脚本 ✅

**上线准备完成**(v1.0.0-rc):
- [x] 29-go-live-checklist-exec.md — 上线检查清单(执行版,38 项)
- [x] backend/.env.prod.example — 生产环境配置模板(强密码/JWT/CORS/支付)
- [x] scripts/deploy_prod.sh — 生产一键部署脚本(9 步:检查→构建→启动→迁移→数据→验证)
- [x] main.py 生产模式(DEBUG=false 时关闭 /docs /redoc /openapi.json)

**可维护性+易用性优化**(v1.0.0-rc3,2026-07-07):
- [x] useConvertTask composable — 4 个转换工具复用,消除 pollTask 重复(行数 -38%)
- [x] useBalanceGuard composable — 3 个付费工具复用,余额不足改弹窗确认
- [x] 实时搜索 — 首页输入实时下拉建议(最多 8 个,按 name/keywords/desc 匹配)
- [x] 占位标记 — 首页卡片显示"即将上线"角标 + 半透明
- [x] 互链扩展 — ToolLayout 从 6 个扩到 70 个(同类 35 + 随机 35),bmcx 模式
- [x] 健康度 6.5/10 → 7.2/10

**万年历产品化优化**(v1.0.0-rc4,2026-07-08,对标 bmcx):
- [x] 从"输入框查单日(6 行信息)"重构为"月历网格 + 点击展开完整黄历"
- [x] 月历网格:7列42格,每格显示公历+农历+节气/节日标注
- [x] 完整黄历详情:农历/干支/生肖/星座/星宿/十二神/六曜/冲煞/宜忌/节日(10 项)
- [x] 月份/年份切换(1900-2100),今天按钮
- [x] 节气标注(从 lunar-javascript 获取精确节气)
- [x] 公历节日标注(元旦/国庆/圣诞等)
- [x] 移动端响应式适配
- [x] 错题集记录 2 个新问题(#010 lunar API 名不匹配 / #011 input_schema 需同步)

**选品重新评估 + 产品设计审查**(v1.0.0-rc5,2026-07-08):
- [x] 诊断选品问题:开发者工具占比 40%,生活工具仅 20%,bmcx 高需求工具缺失 17 个
- [x] 产出改进方案(详见 31 号文档第六章):
  - 第一阶段:补 8 个高搜索量生活工具(个税/房贷/退休年龄/简繁/拼音/老黄历/亲戚/利息,月搜索 200万+)
  - 第二阶段:扩数据(成语→500+/诗词→500+/身份证→3000+)
  - 第三阶段:开发者工具降级 + 合并(HMAC/AES→★1,CSS/JS/HTML 格式化合并)
- [x] 产品设计审查:14 个工具存在"平庸"问题(数据少/无详情/无引导)
- [x] 界面设计规范(32 号文档):一屏可见/双栏布局/视觉层次标准

**选品改进第一阶段完成**(v1.1.0,2026-07-08):
- [x] P0:补 8 个高搜索量生活工具 ✅(月搜索覆盖 200万+)
  - tax-calculator 个税计算器(50万+/月,七级累进税率+五险一金)
  - mortgage-calculator 房贷计算器(40万+/月,等额本息/本金)
  - retirement-age 退休年龄计算器(20万+/月,渐进式延迟)
  - simplified-traditional 简繁互转(20万+/月,opencc-js)
  - pinyin-convert 汉字转拼音(15万+/月,pinyin-pro)
  - old-almanac 老黄历(30万+/月,宜忌冲煞值神)
  - relatives-calculator 亲戚计算器(10万+/月,关系链计算)
  - interest-calculator 存款利息计算器(15万+/月,定期/活期/复利)
- [x] 全量验证 110/110 通过(100%)
- [x] 遵守 SSOT(registry→gen:tools→构建)、设计规范(通用组件)、无耦合

**选品改进第二阶段-纯算法生活工具**(v1.1.1,2026-07-09,对标 bmcx):
- [x] 补 4 个高搜索量纯算法/库工具(无后端/API/数据库,与点1数据扩充零冲突)
  - solar-to-lunar 公历转农历(30万+/月,lunar-javascript,含干支/生肖/节气/节日)
  - auspicious-day 黄道吉日查询(30万+/月,lunar-javascript 宜忌筛选,8 类事项)
  - due-date 预产期计算器(8万+/月,末次月经/受孕日双算法+周期校正+孕周进度)
  - safe-period 安全期排卵期(8万+/月,排卵日/易孕期/前后安全期)
- [x] 生肖取值修复:solar-to-lunar 用 lunar.getYearShengXiao()(非 getAnimal(),后者返回二十八宿动物,见错题集 #015)
- [x] 每个工具配 about 知识区(features/scenarios/faq),响应"去平庸"改进方案1
- [x] 遵守 SSOT:add_life_tools2.py 改 registry → gen:tools → 构建验证,无耦合
- [x] 全量验证 114/114 通过(100%),工具页 87/87 可访问
- [x] 算法逻辑 node 验证通过(预产期/排卵日/吉日筛选均与手算一致)

**部署同步**(v1.1.1,2026-07-09):
- [x] 发现 Windows 与 WSL(/root/tool-site-project)两份副本脱节:WSL 侧源码残缺(只剩2页面)、无 node、registry 停在90、线上工具路由一度全404
- [x] diff 结论:WSL 侧无独有可保留内容(Windows 是超集+更新版),以 Windows 为权威单向同步
- [x] 同步 frontend/src(2→88 页面+composables)、registry(90→110)、dist(新构建 C6KbY8An),未动 backend/.env/数据库
- [x] `docker compose restart nginx`,线上 HTTPS 验证:solar-to-lunar/auspicious-day/due-date/safe-period + zodiac/lunar 修复全部 HTTP 200
- [x] 旧 WSL 文件备份至 `/root/tool-site-project/_backup_20260709_105628`

**选品改进第三阶段-纯算法生活工具**(v1.2.2,2026-07-09,SSG 架构下):
- [x] 补 2 个纯算法零数据工具(SSG 安全、不碰数据库、避开数据扩充)
  - blood-type 血型遗传计算器(ABO 规则表,父母血型→孩子可能/不可能血型,16 组合全验)
  - days-between 日期天数计算器(两日期间隔+工作日/周末分离+周月年换算)
- [x] 按新 SEO 规范(33 号文档)配齐 seo 字段(title/description/keywords/FAQ≥3/how_to/description_for_ai)
- [x] SSG 构建成功:每页预渲染独立 HTML(title/h1/FAQ/JSON-LD/canonical 齐全,非空壳)
- [x] 全量同步 WSL(因 WSL 又落后整个 SSG 基础设施):frontend(src+dist+config+package.json+main.ts+router)+schemas
- [x] 修复 SSG 部署关键:WSL nginx.conf 落后(无 $uri.html),同步后 reload,/blood-type 等预渲染页 title 正确生效
- [x] verify_all 116/116(100%):修复 robots/llms(部署同步后恢复)+ Vue 挂载点断言适配 SSG(data-server-rendered)

**选品改进第四阶段-健康类工具**(v1.2.3,2026-07-09):
- [x] 补 2 个纯算法健康工具(无数据/API依赖,SSG安全)
  - blood-pressure 血压评估器(中国高血压防治指南7级分级+单纯收缩期判断+对照表)
  - standard-weight 标准体重计算器(BMI 18.5-23.9区间+Broca/Devine/Robinson公式)
- [x] 算法 node 验证:血压6种典型值分级正确,标准体重男175/女162计算合理
- [x] 全程用 `scripts/deploy_sync.sh` 一键部署(SKIP_BUILD=1),5/5验证全绿
- [x] verify_all 118/118(100%);2新工具线上SSG title专属正确

**黄历三件套产品力优化**(v1.2.4,2026-07-09,对标 huangli.com):
- [x] 抽 `components/AlmanacDetail.vue` 共享详情组件(万年历/老黄历复用,SSOT 详情逻辑)
- [x] 详情深度对标 huangli 首屏:宜忌左右分栏 + 今日方位(财/喜/福神)+吉神宜趋/凶煞宜忌标签 + 彭祖百忌 + 胎神 + 建除/纳音
- [x] 十二时辰吉凶表(12时辰干支/吉凶/冲煞/时宜时忌,当前时辰高亮,可折叠)— lunar-javascript 全 API 验证可用
- [x] 万年历接入 AlmanacDetail(右栏,16KiB→46KiB 预渲染 HTML 信息密度大增)
- [x] 老黄历改单日详情版(日期切换器+AlmanacDetail,与万年历"翻月历"分工,消除重复)
- [x] 黄道吉日加吉时(每个吉日显示当天吉时时辰)
- [x] 修复 zodiac getAnimal bug 回归(错题集 #015,改 getYearShengXiao,2020-2031全验证)
- [x] 产出 34-product-power-optimization.md(高流量工具清单+迭代方案)
- [x] SSG 预渲染含新模块(非空壳);verify_all 118/118;线上三件套新模块全在线

**P1 计算类可视化**(v1.2.5,2026-07-10,纯 CSS/SVG 不引图表库):
- [x] 房贷 mortgage-calculator:加本金/利息占比堆叠条(用户最关心"利息占多少"),等额本金补末月月供+每月递减额
- [x] 个税 tax-calculator:加"月薪去向"分段条(五险一金/个税/到手占比),钱去哪了一目了然
- [x] 退休 retirement-age:加生涯时间轴(出生→原退休→延迟退休,标记当前位置,在职/已退休状态)
- [x] 算法 node 验证:房贷100万30年本金62%/利息38%,个税15000到手79%,退休1985男时间轴百分比合理
- [x] SSG build 成功(交互后渲染,不影响预渲染);deploy_sync 部署 5/5;verify_all 118/118

**P2 文化类深度核查+iching补全**(v1.2.6,2026-07-10):
- [x] 核查文化工具真实状态(纠正34号文档过时判断):constellation/solar-terms/dynasties/hundred-surnames 已补深度(有data.ts+完整内容)
- [x] iching 补全:16卦→64卦完整数据(data.ts),每卦9维释义(卦辞/象/事业/经商/求名/外出/婚恋/决策),对标huangli今日卦象
- [x] iching 页面重写:起卦随机64卦+多维卡片展示,SSG安全(crypto仅在点击触发)
- [x] SSG build成功;deploy_sync部署5/5;verify_all 118/118

**扩数据 3 批全部完成**(v1.3.0,2026-07-10):

第 1 批 — 数据库类(4 个):
- [x] 成语 45→218(扩 173 条,常用成语)
- [x] 诗词 57→353(扩 296 首,唐宋名篇)
- [x] 身份证 320→2543(扩 2223 条,全国区县)
- [x] 手机 62→294(扩 232 条,三大运营商)
- 脚本: scripts/expand_idioms.sql / expand_poetry.sql / gen_expand_id_cards.py / gen_expand_phone.py

第 2 批 — 前端静态(5 个):
- [x] 谜语 8→102(扩 94 条,动物/植物/物品/脑筋)
- [x] 脑筋急转弯 8→110(扩 102 条)
- [x] 歇后语 10→103(扩 93 条,三国/西游/水浒/神话类)
- [x] 对联 18→55(扩 37 副,春联/婚联/寿联/乔迁/通用 5 类)
- [x] 名人名言 10→103(扩 93 条,诸子百家/近现代)
- 数据外置: frontend/src/tools/{id}/data.ts,页面 import 引用

第 3 批 — 补详情(5 个):
- [x] 生肖 12 项→完整详情(性格/优缺点/配对/幸运/名人)
- [x] 星座 12 项→完整详情(性格/优缺点/配对/职业/恋爱)
- [x] 节气 24 项→完整详情(含义/习俗/养生/农事,可点击查看)
- [x] 朝代 17 项→完整详情(大事/文化/帝王,点击展开)
- [x] 百家姓 50 姓→完整详情(起源/名人/排名)

**P2续:仍薄文化工具补深**(v1.2.7,2026-07-10):
- [x] birthday-code 修复:原仅8日期有数据(357天fallback失效)→ 改生命灵数算法(生年月日各位相加至单数/大师数),12灵数释义+12月主题,覆盖所有生日
- [x] birthday-book 补深:原仅花/石/树+拼字符串 → 12月生日花语/寓意/性格象征
- [x] twenty-eight-stars 补深:原每宿一句"主XX" → 四象(青龙朱雀白虎玄武)归属+吉凶+释义+运势(28宿全)
- [x] luban-ruler 补知:加鲁班尺背景(门公尺8段吉凶/丁兰尺/用途说明)
- [x] birthday-code registry input_schema 同步(month/day→date),生命灵数需完整生日
- [x] SSG build成功;deploy_sync部署5/5;verify_all 118/118

**🔥 修复全站表单bug**(v1.2.8,2026-07-10,错题集#016):
- [x] 用户报"黄道吉日查询一次后按钮失效",精确为"改参数后查询无反应"
- [x] 根因:ToolInput 只 emit `update:modelValue`,但72个工具页用 `@update=` 监听 → Vue3两事件不同 → handleUpdate不触发 → form不更新 → 改参数无效(首查靠默认值成功)
- [x] Chrome headless 最小复现确认(修复前form_x=a不更新,修复后=c正确更新)
- [x] 修复:ToolInput update/toggleCheckboxGroup 同时 emit `update`+`update:modelValue`(一处修复全站72工具生效)
- [x] deploy_sync部署;verify_all 118/118无回归

**黄历吉日查询二级功能升级**(v1.2.9,2026-07-10,对标 huangli.com):
- [x] auspicious-day 吉日列表项可点击 → 跳转 `/old-almanac?date=YYYY-MM-DD` 看完整黄历(对标 huangli 吉日→单日详情)
- [x] 加查询模式切换:未来N天 / **指定月份**(对标 huangli 按月查吉日,选年月查该月全部吉日)
- [x] old-almanac 支持 URL query `?date=` 参数(供跳转定位,SSG 构建期 query 空fallback今天,hydration后读query)
- [x] Chrome headless 实测闭环:点查询→11吉日列表→首项链接/old-almanac?date=2026-07-10→old-almanac显示2026-07-17(query读取生效)
- [x] SSG build成功;deploy_sync部署5/5;verify_all 118/118

**新增AI修图提示词库**(v1.2.10,2026-07-10,形态A模板库):
- [x] ai-retouch-prompt:6分类(换背景/人像美化/风格转换/滤镜调色/智能扩图/抠图换底)×共46条中文修图提示词
- [x] 参考小红书流行趋势改写(非照搬),每条含场景名/适用平台/提示词/使用贴士
- [x] 一键复制(navigator.clipboard + execCommand兜底,SSR安全),Chrome实测复制反馈生效
- [x] 分类标签切换 + 卡片网格 + 使用说明,无表单纯内容展示(同special-symbols/emoji结构)
- [x] 配seo(title/description/keywords/FAQ≥3/how_to/description_for_ai)+about
- [x] SSG预渲染含全部提示词内容(非空壳,爬虫可抓);deploy_sync部署5/5;verify_all 119/119
- [x] 产出35-short-video-tools-plan.md(短视频工具方案,含提示词库思路)

**AI图像提示词库产品化升级**(v1.2.11,2026-07-13):
- [x] 重构为按模型分组:通用(46修图)+豆包+即梦Seedream+Nano banana,模型筛选标签+计数
- [x] 支持长结构化提示词(如Nano banana穿搭拆解600字多段框架),pre滚动展示+一键复制
- [x] 新增来源标注字段(source:官方示例/社区分享/原创整理),每条可追溯
- [x] 入库用户提供的Nano banana穿搭拆解示例(来源:小红书#nanobanana)+即梦官方公式示例(海边日落/东北虎)+豆包修图句式
- [x] 加搜索(标题/内容/分类)+模型色标(豆包蓝/即梦橙/Nano粉)
- [x] 名称扩为"AI图像提示词库"(含修图+穿搭拆解+文生图),id/route不变保链接,SEO扩范围
- [x] 工作流:用户贴收集的提示词→我结构化入库+标来源(无法自动抓取小红书APP内容)
- [x] SSG预渲染46KiB含长提示词;deploy_sync部署5/5;Chrome实测模型筛选+长词复制生效;verify_all 119/119

**接入开源豆包提示词合集**(v1.2.12,2026-07-13):
- [x] 入库 github langgptai/awesome-doubao-prompts(Apache-2.0许可,允许商用)29条中文豆包提示词
- [x] 覆盖7类:生图2/工作效率5/内容创作5/编程开发5/教育学习5/创意设计5/生活服务2
- [x] 含{{变量}}占位符结构化模板,每条带使用贴士,独立存 doubao-prompts.ts
- [x] 豆包分组从2条增至31条,Chrome实测筛选31卡片全豆包、分类多样
- [x] 页面底部加Apache-2.0许可证声明+仓库链接(合规:保留版权声明)
- [x] SSG预渲染73.99KiB含29条豆包提示词;verify_all 119/119

**接入开源通用对话提示词**(v1.2.13,2026-07-13):
- [x] 入库 github PlexPt/awesome-chatgpt-prompts-zh(MIT许可)精选41条通用对话角色提示词
- [x] git clone拉取prompts-zh.json(124条)→脚本精选41条(排除技术开发/擦边/学术,保留创作/生活/起名/咨询)
- [x] 覆盖翻译/讲故事/小说家/编剧/诗人/标题生成器/导游/厨师/营养师/造型师/起名等,适用豆包/Kimi/通义
- [x] 新增"通用对话"模型分组(绿色色标),独立存 general-chat-prompts.ts
- [x] 页面许可证声明补MIT来源;gen_general_prompts.py生成脚本(可复用扩充)
- [x] 提示词库总计120条(通用44+通用对话41+豆包31+即梦3+Nano1),6分组
- [x] Chrome实测6模型计数正确+通用对话筛选41卡片全验;verify_all 119/119

**提示词产品力提质(按"交付难/复杂/有门槛"标准)**(v1.2.14,2026-07-13):
- [x] 诊断:现有120条大量低门槛(纯角色扮演"用户自己也会说"),修图8换背景8滤镜同质,缺"难自写"的高门槛提示词
- [x] 删低价值:通用对话41→13条(删28条纯角色扮演营养师/讲故事/心理等,留标题生成/起名/编剧/文案等有套路的)
- [x] 补高门槛:新增6条用户难自写的高阶提示词(角色一致性约束/电影感光影/面料材质/负面约束排除/Character Sheet套路/商业精修)
- [x] 加difficulty字段+难度色标(入门灰/进阶黄/高阶红),呼应"门槛"分析
- [x] 顺手修预存问题:hundred-surnames/data.ts Duplicate key "程"(删重复条目)
- [x] 提示词库120→98条,质量提升;高阶徽章6个;Chrome实测分组计数+difficulty正确
- [x] SSG build干净无error无warning;deploy_sync部署5/5;verify_all 119/119

**P0 计算器对标改造**(v1.3.1,2026-07-13,对标 AuxTool/房天下):
- [x] 个税计算器:加年终奖单独/合并计税对比 + 7级税率档位可视化 + 到手/社保/个税分段占比条
- [x] 房贷计算器:加还款明细表(每月本金/利息/剩余) + 提前还款计算(省多少利息) + 本金/利息占比堆叠条
- [x] 验证:两工具 HTTP 200,无前端错误

**产品设计规范 v2.0 沉淀**(v1.4.0,2026-07-13):
- [x] 32-ui-design-spec.md 升级为产品设计规范(6条铁律+5类工具模板+检查清单)
- [x] 核心原则:一进来就是完整产品(默认值+自动计算+结果始终可见+功能可见性)
- [x] 沉淀教训:功能改了界面没变/新功能用户看不到/日历太大/API名漏改

**第1批产品优化全部完成**(v1.4.0,7个工具按新规范重写):
- [x] 个税计算器:Tab切换(月薪/年终奖) + 预设默认值 + 到手占比条 + 税率档高亮 + 明细卡片 + 单独vs合并对比
- [x] 房贷计算器:预设默认值 + 三卡片(月供/利息/总额) + 本息占比条 + 可展开明细表 + 可展开提前还款 + 提示
- [x] 退休年龄:预设默认值 + 退休时间渐变卡片 + 三卡片(原年龄/延迟年龄/延迟月数) + 时间轴(已过/剩余) + 社保提示
- [x] 存款利息:预设默认值 + 利息渐变卡片 + 四卡片(本金/利息/本息/年化) + 占比条 + 参考利率表(8种) + 提示
- [x] BMI计算器:预设默认值 + BMI大数字+分类+建议 + 仪表盘指针(偏瘦/正常/超重/肥胖) + 四卡片(理想体重/体脂率/基础代谢)
- [x] 单位换算:8类分类(长度/重量/温度/面积/体积/速度/数据/时间) + 全单位同时显示 + 点击切换输入源
- [x] 颜色转换:颜色选择器+HEX输入 + 三格式(HEX/RGB/HSL)一键复制 + 配色方案(互补/类似/三角) + 明暗渐变

**第2批产品优化全部完成**(v1.4.1,2026-07-13,7个工具按新规范重写):
- [x] 时间戳转换:双面板(时间戳→日期 / 日期→时间戳) + 当前时间实时显示 + 6个常用时间戳快捷预设
- [x] 字数统计:8指标实时统计 + 阅读时间/朗读时间/写作时间 + 字数占比条(中文/英文/标点/其他)
- [x] JSON格式化:实时美化/压缩 + 语法高亮暗色主题 + 错误显示 + 字节数+压缩率
- [x] 进制转换:四主卡片(BIN/OCT/DEC/HEX 渐变色) + 二进制位可视化 + 展开更多进制(Base 3-36) + 进制小知识
- [x] 大小写转换:8种模式(全大写/全小写/首字母/标题/驼峰/下划线/短横线/点分隔) + 暗色结果 + 实时字符/单词/行统计 + 用例说明
- [x] 中文数字:三模式(大写/小写/口语) + 渐变结果卡片 + 数字拆解可视化 + 数字对照表(0-9 三种写法) + 应用场景
- [x] 存储单位:渐变主结果卡片 + 全单位对照(1024进制) + 十/二进制差异说明(硬盘容量少多少%) + 实用容量对照
- [x] URL编码:实时编码/解码 + 错误提示 + 12个特殊字符编码对照表 + 应用场景
- [x] MD5哈希:四算法实时计算(MD5/SHA1/SHA256/SHA512 渐变色卡) + MD5字节可视化(16字节方格) + 算法对比表 + 一键复制
- [x] CSS格式化:美化/压缩 + 缩进选项(2/4/Tab) + 语法高亮(选择器/属性/值/注释四色) + 字节数+压缩率+规则数

**修复**(v1.4.1):
- [x] 修复 json-format 高亮正则有多余 `)` 导致 esbuild 解析失败

**布局反直觉问题修复**(v1.4.2,2026-07-14,产品设计规范 v2.1):
- [x] 写入设计规范:§1.0 新增"反面教训:触发按钮在上方"、铁律第7条、§3.6 占卜/随机/抽奖类模板、§3.7 实时转换类模板、检查清单新增2条
- [x] 全项目审计所有工具,识别 5 个违规:iching/lottery/random-number-gen(占卜/随机类) + base64-encode/morse-code(转换类)
- [x] 易经起卦:卦象舞台在顶部,起卦按钮在下方 + 800ms 摇卦滚动动画 + scale+opacity 出现动画 + 计数
- [x] 抽奖:中奖结果舞台在顶部,开始按钮在下方 + 800ms 滚动动画 + 中奖号码逐个弹出动画 + 计数
- [x] 随机数生成:结果舞台在顶部,生成按钮在下方 + 700ms 滚动动画 + 数字逐个弹出动画 + 计数
- [x] Base64 编码:移除"转换"按钮,改为实时转换 + Tab切换 + 暗色结果 + 知识区(用途/字符集/体积/场景)
- [x] 摩斯电码:移除"转换"按钮,改为实时转换 + Tab切换 + 暗色结果 + 完整对照表(字母+数字+标点) + 应用场景

**审计中合理的布局**(保留):
- 查询类(idiom/poetry/phone/id-card/ip/dns):输入框+查询按钮+结果,搜索引擎式,合理
- 上传类(pdf-to-word):上传区在上,合理
- 流程类(mind-reading/reaction-test):分步引导,合理

**通用审美规范沉淀**(v1.4.3,2026-07-14,设计规范 v2.2):
- [x] 设计规范 §3.6 改名为"生成/占卜/抽奖类",扩展适用范围到密码/UUID 等所有"点击生成"工具
- [x] 新增 §3.8 通用审美规范(7小节):首屏视觉重心 / 空白处理 / 配色层次 / 字号层级 / 圆角阴影 / 动画反馈 / 一屏完整度
- [x] 新增问题7:生成器按钮上方留空白 / 问题8:首屏没有视觉重心
- [x] 检查清单新增 8 条(3产品+5审美)
- [x] 随机密码生成:顶部密码舞台(渐变+大字+强度色标) + 中部配置(滑块+字符集卡片) + 底部按钮 + 滚动生成动画 + 强度参考表
- [x] UUID 生成:顶部 UUID 舞台(列表渐入动画) + 中部配置(v1/v4 Tab+数量滑块+大写开关) + 底部按钮 + UUID 知识区

**全项目审计 + P0-P2 批量改造**(v1.4.4,2026-07-15,共 15 个工具):

**P0 布局反直觉(3个)**:
- [x] 二维码生成:顶部二维码舞台(渐变+实时生成+防抖) + 配置区(滑块) + 操作区(复制/下载) + 二维码知识
- [x] 占位图生成:顶部占位图舞台 + 配置区(宽高滑块/文字/双色) + 6 个快捷尺寸预设 + 操作区
- [x] 黄道吉日:顶部结果摘要舞台(渐变+大数字) + 配置区(事项/模式/日期/天数滑块) + 吉日列表卡片网格(宜/吉时标签)

**P1 转换类应实时(8个)**:
- [x] HTML 转义:实时转义/反转义 + Tab 切换 + 5 字符转义对照表 + 4 应用场景
- [x] 简繁转换:实时简繁 + Tab 切换 + 6 用词差异示例(软件→軟體等)
- [x] 拼音转换:实时拼音 + 声调(带/数字/无)+ 格式(带空格/不带/首字母)Tab + 拼音知识
- [x] 文本去重:实时去重 + 4 选项(去空格/忽略大小写/排序/只保留唯一)+ 4 统计卡片(原/去/删/压缩率)
- [x] HMAC 生成:实时 HMAC + 4 算法(MD5/SHA1/SHA256/SHA512)+ 大写开关 + 算法对比
- [x] AES 加密:实时加解密(防抖)+ Tab 切换 + 错误提示 + AES 知识(密钥长度/模式/填充)
- [x] SHA 哈希:实时 SHA-1/256/512 + 大写开关 + 算法对比表
- [x] 文件哈希:顶部文件信息舞台 + 拖拽上传 + 3 算法渐变卡片 + 加载进度条 + 哈希用途
- [x] HTML 格式化:实时格式化(防抖)+ 语法高亮(标签/属性/值/注释四色)+ 4 知识卡片
- [x] JS 格式化:实时格式化(防抖)+ 语法高亮(关键字/字符串/数字/注释)+ 统计(字节数/行数)
- [x] SQL 格式化:实时格式化 + 6 种 SQL 方言选择 + 语法高亮(关键字/函数/字符串/注释)
- [x] 正则测试:实时匹配 + 3 标志 Tab + 匹配高亮预览 + 匹配列表 + 8 常用正则预设
- [x] 文本对比:双栏输入 + 2 选项(忽略大小写/空格)+ 5 统计卡片(总/同/增/删/变化率)+ 行号 diff

**P2 计算器类(5个)**:
- [x] 年龄计算:顶部年龄大数字舞台 + 4 时间单位卡片(月/天/时/分)+ 下次生日进度条 + 生肖/星座/生命天数
- [x] 日期间隔:顶部天数大数字舞台(负差红底)+ 8 单位卡片(工作日/周末/周/月/年/时/分/秒)
- [x] 血压评估:顶部分级舞台(6 种颜色渐变)+ 双滑块(收缩/舒张)+ 7 级血压分级表(高亮当前)
- [x] 血型遗传:顶部可能血型舞台(父×母→子)+ 4 血型按钮 + 不可能血型 + ABO 遗传规律说明
- [x] 预产期:顶部预产期舞台(进度条)+ 计算方式 Tab + 4 详情卡片 + 周期滑块
- [x] 安全期:顶部排卵日舞台 + 4 阶段彩色时间条(经期/前安全/易孕/后安全)+ 详细信息表
- [x] 标准体重:顶部标准体重范围舞台 + 性别 Tab + 身高滑块 + 3 公式卡片(Broca/Devine/Robinson)+ BMI 区间表

**待执行**(后续优先级,共 27 个工具待审):
- [ ] **P3 详情类**(10个):birthday-book/birthday-code/constellation/hundred-surnames/luban-ruler/periodic-table/solar-terms/solar-to-lunar/twenty-eight-stars/zodiac — 需要预设值+卡片布局
- [ ] **P4 生成类**(2个):countdown-timer/online-alarm — 需要"舞台+按钮"模式
- [ ] **P5 查询类**(8个):idiom-dict/poetry/phone-region/id-card-region/history-today/ip-query/dns-query/whois-query — 搜索引擎式合理,但需要优化默认值和首屏
- [ ] **P6 PDF 付费类**(5个):id-photo/image-compress/resume-generator/pdf-to-excel/pdf-to-ppt/pdf-to-scanned — 暂缓,Worker 未接
- [ ] **P7 其他**:couples(对联)/http-status/password-gen(可能与 random-password 重复,考虑 deprecated)

**用户/支付/卡密三模块补全方案**(v1.5.0,2026-07-15,设计完成):
- [x] 新增 [36-user-payment-cardkey-modules.md](./36-user-payment-cardkey-modules.md) 作为三模块 SSOT
- [x] 现状盘点:后端模型/API/服务齐全(注册/登录/卡密兑换/支付回调占位),前端 4 页(Login/Redeem/UserCenter/Pricing)旧式待改造
- [x] 模块边界:三模块单向依赖(A→B→C→A 闭环),后端 modules/user|payment|card_key 三层结构
- [x] 数据模型增量:5 张新表(user_sessions/user_profiles/email_verifications/payment_logs/card_redeem_logs)+ 现有表字段追加
- [x] API 契约增量:模块 A 11 路径 + 模块 B 6 路径 + 模块 C 7 路径 + 14 个新错误码
- [x] 模块 A 用户系统:账号/邮箱/手机/微信扫码 4 种登录 + 找回密码 + 资料 + 设备列表 + 注销
- [x] 模块 B 支付系统:渠道适配器策略模式 + 微信 Native 扫码(V3 RSA-SHA256)+ 虎皮椒/PayJS + 订单状态机 + 退款 + 对账
- [x] 模块 C 卡密系统:生成/兑换(行锁+事务+日志)/作废/统计/闲鱼订单关联
- [x] 前端页面:6 改造 + 11 新增(含管理员后台 4 页)
- [x] 路线图:P0 MVP(8.5d 闲鱼可发卡)→ P1 在线支付(10.5d)→ P2 用户系统(8d)→ P3 卡密运营(4.5d)→ P4 对账风控(5d),合计 36.5 人日
- [x] 踩坑预防:基于错题集列出 5 类 20 项雷区(数据库/支付/卡密/前端/一致性)
- [x] 上线检查清单:后端 10 项 + 前端 7 项 + 安全 7 项 + 监控 3 项
- [x] 迭代记录机制:本文件第十二章为后续每次迭代追加记录的模板

**待执行**(下一步按 37 开发计划落地,36 号为设计 SSOT):
- [x] 36 号方案升级到 v1.2(多收款方式 + 对外卡密 API + 订单状态机闭环 + 兑换校验)
- [x] 37-development-plan.md 产出 v1.1(开发前评审 4 硬伤 + 5 建议已修)
- [x] 🚨 git commit 保命(6 提交,工作区 clean)
- [x] P0 M0:openapi.yaml 补全(31 路径,6 admin + 5 对外 Open API,ApiKeyAuth + 8 schema components,YAML 验证通过)
- [x] P0 M1a:modules/card_key 骨架(models/schemas,CardKeyDistributor/CardKeyBatch)
- [x] P0 M1b:迁移 0003-0006 全部跑通(head=0006_open_api,7 新表建好,card_keys 加 valid_from/external_order_no/batch_id 改 FK)
- [x] P0 M1c:卡密兑换强化(service.redeem 唯一实现,E105/E106/兑换日志;端到端验证:兑换0→50,重复E102,日志落表)
- [x] P0 M1d:对外 Open API(HMAC+Fernet+nonce 防重放,5 接口;验证:claim/redeem-notify/幂等E506/跨入口互斥E102/错签E502/query 全通过)
- [x] P0 M1e 后端:卡密管理后台(admin generate/list/export/revoke/stats/bind-order + get_current_admin;端到端验证:admin 生成卡密成功)
- [x] P0 M1e 前端:CardKeys.vue(生成/列表/导出/作废/统计,§3.6舞台+§3.8审美)+ 路由守卫 requiresAuth/requiresAdmin(setupRouterGuards)+ admin 导航入口 + robots 屏蔽 /admin/
- [x] P0 M2:Login/Redeem/UserCenter/Pricing 现状已符合 P0 规范(完整5Tab留P2,在线购买留P1,小步边界);部署 deploy_sync 5/5
- [x] P0 验收:verify_all 119/119 全绿 + Docker 全服务 Up + admin 后台端到端(页面200/admin接口OK/非admin拦403)
- [x] 闲鱼录单接口(§6.0/§6.3):admin 录入闲鱼订单(channel=xianyu,pending,买家可无账号)+ 手动确认收款(走 pay_order_success,与在线支付同一发卡逻辑不分叉)+ admin 订单列表。端到端:录单→确认发卡→兑换余额100→幂等全通过
- [ ] P1-P4 按 37 号 §三推进(总 37.5 人日)

> **🎉 P0 全部完成**(2026-07-16,按 37 号 v1.1):
> - 后端:openapi契约(34路径)+ modules/card_key(兑换/管理/对外OpenAPI)+ 迁移0003-0007 + 闲鱼录单
> - 前端:CardKeys.vue后台 + 路由守卫 + admin导航 + robots屏蔽
> - **端到端闭环全验证**:兑换/生成/对外API(claim+redeem-notify+幂等+跨入口互斥)/闲鱼录单(录单→确认发卡→兑换)/admin鉴权
> - verify_all 119/119,Docker 全服务 Up
> - **新增迁移 0007**:orders.user_id 改 nullable(闲鱼录单无买家账号)
> - **新踩坑**(27号 #021):pay_order_success batch_id 旧值触发 FK 违反(硬伤2发卡路径遗漏),改 batch_id=None + 清 __import__ hack

**P1 在线支付进行中**(2026-07-16):
- [x] P1a 渠道适配器策略模式:modules/payment/channels/(base抽象+wechat_native+xunhupay+payjs),CHANNELS注册表,get_channel工厂。3渠道注册验证OK
- [x] P1b 订单状态机:pay_order_success 补 paid→delivered(卡密就绪即交付),幂等扩为 status in (paid,delivered)。验证:确认收款→delivered
- [x] P1c 退款↔卡密联动(§6.3防双重得利):refund_order unused→revoke/used→扣余额(已消耗E507)。验证:unused退款卡密E104+used退款余额150→100扣减
- [x] checkout在线下单+订单状态查询接口(渠道适配器发起支付)
- [x] P1d 前端:Checkout.vue(选套餐→选渠道→扫码→轮询)+useOrderPolling(超时停止)+Pricing加在线购买按钮跳checkout
- **新踩坑**(27号 #022):create_order channel白名单漏新渠道(wechat_native等),checkout报E404,扩白名单
- **P1后端闭环验证通过**:checkout→delivered→退款unused(revoke)/used(扣余额)全过,verify_all 119/119
- **P1前端验证通过**:/payment/checkout SSG 200 + 守卫(无token401/有token200) + validate 0警告 + deploy 5/5
- **🎉 P1 全部完成**,进 P2(用户系统:找回密码/微信扫码/资料/设备/注销)或对账风控

**P2 用户系统进行中**(2026-07-16):
- [x] P2a modules/user骨架 + JWT黑名单:create_access_token加JTI,is_jti_revoked(Redis黑名单+user_sessions库兜底,评审断点1),登出吊销JTI。验证:登出后旧token 401
- [x] P2b 设备列表+强制下线:登录落user_sessions(IP/UA/设备),/api/user/sessions查设备,/api/user/sessions/{jti}吊销(E408不能吊销自己),全部下线。验证:强制下线后token立即401
- [x] P2c 用户资料+改密码:GET/PUT /api/user/profile(nickname/avatar/bio),改密码(需当前密码+吊销所有会话)
- [x] P2d 找回密码邮箱验证码:forgot-password(发6位码,SMTP占位控制台输出)+verify-code+reset-password(5分钟有效,错5次锁,防邮箱枚举)。验证:重置后旧密码401
- [x] P2e 注销(软删banned+吊销所有会话,token立即失效)+微信扫码骨架(占位,需WECHAT_APPID)
- [x] P2e 前端:UserCenter 改造为资料/登录设备/安全3Tab(改密码/强制下线/注销)+client.ts加userApi(P2接口)
- **新踩坑**(27号 #023):E302-E305/E408/E409 漏定义致fallback E500(同#020复发,补错误码要全表核对§4.2)
- **P2后端闭环验证通过**:登录落session→设备列表→强制下线(401)→登出(401)→找回密码→改资料→注销(token失效),verify_all 119/119
- **🎉 P2 全部完成**(后端+前端),进 P3(卡密运营)或P4(对账风控)

**P3 卡密运营完成**(2026-07-16):
- [x] P3a 批次管理:admin/batches(列表含使用率/过期率)+ PUT status(停用/启用,激活E106完整);过期清理端点(cleanup-expired,dry-run+执行)
- [x] P3b 闲鱼CSV批量发货:admin/xianyu-batch-ship(订单列表→批量生成卡密+关联external_order_no→返回发货清单);generate_cards.py加--db/--from-orders(本地脚本)
- [x] P3c 作废退款联动:revoke_card查关联订单,有则落日志提示需退款(§6.3);统计增强(使用率)
- [x] P3c 前端:CardKeys.vue批次管理区(批次列表+使用率+停用/启用+清理过期按钮)+client.ts加批次/清理/发货接口
- **新踩坑**(27号 #024):运维脚本依赖app模块但容器未挂载scripts目录→运维逻辑做成admin API端点(无路径依赖)
- **P3全部验证通过**:批次列表(使用率)/停用/过期清理dry-run/闲鱼批量发货(关联external_order_no)/前端build+deploy 5/5/verify_all 119/119/openapi 39路径
- **🎉 P3 全部完成**(后端+前端),进 P4(对账风控)

**P4 对账风控后端完成**(2026-07-16):
- [x] P4a 收入看板:admin/dashboard(今日/本周/本月收入+订单数,按channel分类,闲鱼+在线统一统计)+退款30日汇总
- [x] P4a 每日对账:admin/reconcile(按日期核对订单收入vs卡密生成,各渠道paid/refunded统计,discrepancy差异标记)
- [x] P4b 退款记录:admin/refunds(退款列表查询,P1 refund直执行基础上加记录查询)
- [x] P4c 风控告警:admin/risk-alerts(同IP短时多次兑换失败聚合,超阈值告警,企业微信机器人占位日志)
- [x] P4 前端:admin/Dashboard.vue(收入看板4卡片+渠道分布+每日对账+风控告警查询)+路由 /admin+导航📊后台+robots屏蔽/admin
- **新踩坑**(27号 #025):新增端点用Query但顶部未import致api crash loop(NameError,同#020/#023类)
- **P4全部验证通过**:dashboard(按渠道分类)/reconcile(差异标记)/refunds/risk-alerts/前端build+deploy 5/5/verify_all 119/119/openapi 43路径
- **🎉 P4 全部完成(后端+前端)。三模块方案 P0-P4 全部落地!**

> **🎉 三模块方案全部完成**(2026-07-16,36号 v1.8 落地):
> - P0 MVP(兑换/管理/对外API/闲鱼录单/卡密后台) ✅
> - P1 在线支付(渠道适配器/delivered/退款联动/Checkout) ✅
> - P2 用户系统(JWT黑名单/设备/找回密码/资料/注销) ✅
> - P3 卡密运营(批次管理/过期清理/闲鱼发货/作废联动) ✅
> - P4 对账风控(收入看板/对账/退款/风控告警) ✅
> - 总 19 提交,踩坑 #017-#025 共 9 个新坑记入 27 号,verify_all 始终 119/119
> - **下一步**:真实外部资源(域名/SSL/备案/VPS/微信支付资质/SMTP/企业微信webhook),见 01 进度"待用户准备外部资源"

**接入清单产出**(2026-07-16):
- [x] 38-placeholder-integration-checklist.md:8 个占位的接入地图(文件:行/当前实现/替换目标/所需资源/接入步骤/顺序)
  - 微信Native下单+回调验签 / 虎皮椒 / PayJS / SMTP邮件 / 微信扫码登录 / 企业微信告警 / 回调路由接线 / config配置项
  - 每个 §独立接入互不阻塞,接入后跑 verify_all + §11 检查清单
- **接入顺序**:域名SSL备案 → SMTP → 虎皮椒/PayJS(个人可用) → 企业微信告警 → 微信支付V3(企业资质) → 微信扫码(可选)

**支付接入后台页面完成**(2026-07-16):
- [x] 后端 config.py 补18个渠道/邮件/告警字段定义(修复隐性bug:getattr读不到.env值→settings.X可读)
- [x] modules/config/{service,api}:read_env_status(掩码,不返回完整密钥)+write_env(原地写,空值不覆盖,字段白名单防改基础配置)+test_channel(连通测试,从.env重读最新值)+restart端点(无docker.sock提示手动)
- [x] docker-compose.yml 挂载 ./.env:/app/.env:rw(后端可写宿主.env)
- [x] error_codes 加 E508;main 注册 config 路由;.env.prod.example 字段名统一 XUNHUPAY_SECRET+补渠道字段
- [x] 前端 PaymentConfig.vue(6渠道分组,状态+掩码+编辑password框+测试连通+重启生效)+路由+Dashboard入口
- **安全**:密钥只写不读(GET永不返回完整值,POST不回显);白名单拒改 JWT/POSTGRES 等基础配置;空值不覆盖;商户私钥只配路径不输内容
- **新踩坑**(27号 #026):挂载的.env用os.replace跨inode报busy→改原地覆盖写+内存备份兜底
- **验证通过**:get(掩码)/write(写入.env不破坏其他键)/空值不覆盖/白名单E403/test连通/restart提示手动 + 前端build+deploy 5/5+verify_all 119/119+/admin/payment-config 200
- **下一步**:渠道真实API接入(微信V3下单/SMTP发送/wecom告警)按38号清单替换占位,密钥现可在后台页配置

**竞品分析 + Stirling 转换链路实测**(2026-07-16,新增 [38-competitor-and-conversion-gap.md](./38-competitor-and-conversion-gap.md)):
- [x] 竞品「格式智造」filetools.com.cn 实证:四川小主体(蜀ICP备2025164628 + ycmail@189.cn),sitemap 386 个 X转Y 长尾页、无定价/会员页 → 判为 SEO 流量农场 + 后置变现,非付费转换
- [x] 竞争力结论:不同赛道——它"免费+广覆盖+标准质量",我们"高质量付费+垂直";WPS 高质量转换是独有护城河(它追不上);**最大缺口是备案**(它双备案、我们待定)而非格式广度
- [x] 开源合规:FFmpeg/LibreOffice/Pandoc/ImageMagick 均可商用;**勿照抄其"H.264 基于 FFmpeg 开源协议"声明**(FFmpeg 许可证 ≠ H.264/MPEG-LA 专利授权),仅做文档/图片不碰 H.264
- [x] 🚨 Stirling 链路硬伤实测:`convert.py` 入队 `queue:stirling` 后**无消费者**(docker-compose 无 stirling-worker,Stirling 容器此前从未启动)→ 任务永远卡 queued。registry 挂的 12 个 `backend=stirling` 工具全是"假就绪"
- [x] Stirling v2.14.2 实测(起容器 + 拿 OpenAPI 259 端点 + 跑真实转换):latest 镜像**自带 unoserver(LibreOffice)**,Office 转换原生支持(无需另装);配全局 key `SECURITY_CUSTOMGLOBALAPIKEY=zhiniao_stirling_key_2026`(走 \\wsl.localhost UNC Edit 避 CRLF)
- [x] 端点实测三发现:① **audio-convert 是假工具**(259 端点无任何音视频端点,需独立 FFmpeg worker);② excel-to-pdf/ppt-to-pdf 实测可转(靠内置 unoserver);③ pdf-ocr 500(ocrmypdf 报错,需查 tesseract 中文语言包)
- [x] 白送端点(registry 没有、Stirling 现成能转):pdf-to-text / html-to-pdf / markdown-to-pdf / pdf-to-html / pdf-to-epub 等
- [x] 产出回归脚本 `scripts/test_stirling_endpoints.py`(10/10 端点全 200,见 38 号 §四);清理临时脚本 diag_stirling_errors*.py + _stirling_openapi.json
- [x] 产出 38-competitor-and-conversion-gap.md(竞品分析 / 缺口清单 / 实测端点表 / stirling-worker 设计 / 落地路线)
- **修正后优先级**:原"先补前端页"判断有误(后端无消费者,补了也没用)→ **P0 写 stirling-worker**(Python 队列消费者:queue:stirling 取任务 → 调 Stirling HTTP API → 存 MinIO → 回写状态,一次性激活 12 工具)→ P1 补 11 前端页 → P2 新增 image-format-convert / word-to-pdf(对标长尾)→ P3 WPS 高质量档对比
- ⚠️ Stirling License:SaaS 模式需付费 license,免费版锁 5 用户(GRANDFATHERING LOCKED),商用前需评估
- ⚠️ 双副本:38doc 原仅 Windows、回归脚本原仅 WSL——已同步(脚本 → Windows 防 deploy_sync `--delete` 丢失;docker-compose key → Windows)

**stirling-worker P0 落地完成**(2026-07-16,见 [38-competitor-and-conversion-gap.md](./38-competitor-and-conversion-gap.md) §七 P0):
- [x] 建 `workers/stirling/`(worker.py 消费循环+心跳+DB更新 + route_map.py 11 工具端点映射排除 audio-convert + requirements + Dockerfile),自包含不 import backend(10 号"共享契约不共享代码")
- [x] docker-compose 加 stirling-worker 服务(复用 backend/.env,depends_on stirling 用镜像自带 healthcheck /api/v1/info/status)+ STIRLING_API_KEY 变量化(stirling 与 worker 共用同一 key)
- [x] .env 模板加 STIRLING_API_KEY/STIRLING_URL;deploy_sync 加 workers/ 同步 + `REBUILD_BACKEND=1` 重建 worker/api
- [x] 修复 convert.py expires_at 23点崩溃(timedelta);file_validator 补 4 工具 INPUT_TYPES(pdf-to-scanned/excel-to-pdf/ppt-to-pdf/ebook-convert)
- [x] 修复 .env SMTP 字段错位(后台支付接入页遗留,致 api crash loop,手动还原+备份 .env.bak_stirlingfix)
- [x] SMTP 接入闭环(#033)+ 邮件真发(新建 utils/email.py 接通)+ 后端类型防线(INT_FIELDS)+ 验证发信功能(`POST /api/admin/config/test-email` 真发测试邮件,前端 SMTP 渠道加收件邮箱输入+按钮)+ openapi 补 config 模块 5 端点(修一致性债,48 路径)。端到端:SMTP 登录连通成功 + 真发测试邮件到 admin@bibilabu.cc 收信链路通
- [x] **新踩坑 4 个**(27 号 #032-#035):httpx files+data(list)破坏 multipart / .env 字段错位 / expires_at replace 越界 / registry↔INPUT_TYPES 一致性
- [x] 端到端验证:Stirling 端点 10/10 + worker E2E(pdf-compress / pdf-to-scanned 2/2,下载验证 %PDF)+ audio-convert 假工具优雅失败 E006 不挂起 + verify_all 119/119
- [x] 产出回归脚本:`scripts/test_stirling_worker_e2e.py`(POST→轮询→mc 验证 MinIO 输出)+ `scripts/test_stirling_unsupported.py`(假工具优雅失败)
- ⚠️ 待办:① pdf-ocr 500(tesseract 缺 chi_sim,需自建 stirling 镜像,worker 已优雅失败 E004);② ~~PaymentConfig.vue 字段错位根因待查(#033)~~ ✅ 已闭环(2026-07-16,见 27 号 #033 闭环段:后端 write_env 加 INT_FIELDS 类型防线 + 新建 utils/email.py 接通 SMTP 真发 + 前端 saveEdit 校验/placeholder 引导);③ Stirling SaaS license 商用前解决(#029);④ ~~P1 补 11 个 stirling 工具前端页~~ ✅ 见下方 P1 块

**stirling-worker P1 前端页落地完成**(2026-07-16,8 工具上线,后端 P0 已活):
- [x] 实测未验证端点(决定可 ship 集合):pdf-decrypt/password ✅200、pdf-watermark/watermarkText+fontSize ✅200、pdf-split 留空=拆单页(zip)✅、pdf-ocr ❌500(tesseract 无语言包)、pdf-merge 单文件✅但需多文件 API(延迟)
- [x] route_map 修正:pdf_split 留空 omit pages(拆全部)、pdf_watermark 仅留已验证字段(watermarkText+fontSize)
- [x] `scripts/add_stirling_pages.py` 为 8 工具补 input_schema/action_label/related_tools/seo(how_to+faq≥3+description_for_ai)→ `npm run gen:tools`(108 工具)
- [x] 创建 8 前端页(按 pdf-to-scanned 模式:ToolLayout+ToolInput+ToolButton+useToolForm+useConvertTask):pdf-compress/pdf-to-image/excel-to-pdf/ppt-to-pdf/ebook-convert/pdf-decrypt/pdf-watermark/pdf-split
- [x] router implementedTools 加 8 id;构建 SSG 8 页 HTML 生成,title 专属
- [x] 部署(deploy_sync REBUILD_BACKEND=1)+ 验证:8 工具页 200 + verify_all 127/127 + E2E 回归 2/2 + pdf-split 多页 E2E success
- **延迟 3 个(诚实标注)**:pdf-merge(需多文件上传 API,convert.py 单文件→独立任务)、pdf-ocr(tesseract 缺 chi_sim+eng,需自建 stirling 镜像)、audio-convert(假工具,需 ffmpeg-worker)

> - **下一步**:stirling P2 新增 image-format-convert(ImageMagick,SEO ⭐⭐⭐⭐⭐)+ word-to-pdf(Stirling 白送端点);另:pdf-ocr 自建镜像装 tesseract chi_sim、pdf-merge 多文件 API
> - **(原)下一步**:P1 在线支付(微信Native + Checkout/订单页 + 退款↔卡密联动,11人日)

**stirling-worker P2 白送端点新工具落地完成**(2026-07-17,5 工具上线,无需新 worker):
- [x] 实测确认 P2 端点字段:word-to-pdf(/convert/file/pdf 传 docx)✅、html-to-pdf ✅、pdf-to-text(必传 outputFormat=txt)✅、markdown-to-pdf ✅、pdf-to-html(多页 zip)✅
- [x] route_map 加 5 映射(pdf-to-text builder 必传 outputFormat,其余 None);file_validator INPUT_TYPES 加 5(word-to-pdf=.docx/.doc,html-to-pdf=.html/.htm,pdf-to-text/.pdf,markdown-to-pdf=.md,pdf-to-html=.pdf)
- [x] `scripts/add_stirling_p2.py` 新增 5 完整工具条目到 registry SSOT → `npm run gen:tools`(113 工具)
- [x] 创建 5 前端页(同 P1 模式);router implementedTools +5;构建 SSG 5 页 HTML 生成
- [x] 部署 + 验证:5 工具页 200 + 后端识别新 tool_id(api restart/rebuild 重载 registry 缓存)+ word-to-pdf 真 docx E2E success + pdf-to-text E2E success + verify_all 132/132
- **部署关键**:tool_registry.py import 时缓存(L16-17)→ 加新工具需 restart/rebuild api 重载;file_validator 改动需 --build api(restart 不够,代码 baked 进镜像)。WSL backend 已含用户 SMTP 代码(email.py),--build api 不丢 SMTP
- **踩坑**:手工 zip 造的极简 docx Stirling 报 "source file could not be loaded"(非链路问题);用 Stirling pdf-to-word 生成真 docx 验证 word-to-pdf success

> - **下一步**:stirling P3 独立引擎新工具(image-format-convert 需 ImageMagick worker,SEO ⭐⭐⭐⭐⭐;audio-convert 真版需 FFmpeg worker);另:pdf-ocr 自建镜像装 tesseract chi_sim、pdf-merge 多文件 API
> - **(原)下一步**:P1 在线支付(微信Native + Checkout/订单页 + 退款↔卡密联动,11人日)

> **P0 前端开发记录**(2026-07-16):
> - userStore 加 isAdmin getter(ADMIN_USERNAMES=['admin'],与后端§7.3一致)
> - client.ts 加 cardKeyAdminApi(6接口,注意 baseURL=/api + 后端自带 /api/card-keys,前端用 /card-keys)
> - CardKeys.vue:生成舞台(顶部结果)+ 统计表 + 列表(分页/过滤/导出CSV/作废)
> - router/index.ts:加 /admin/cards 路由(meta requiresAuth+requiresAdmin)+ setupRouterGuards(SSR isClient 守卫)
> - main.ts:ViteSSG 回调拿 router 调 setupRouterGuards
> - DefaultLayout:admin 用户显示「🔐 后台」导航
> - gen-sitemap.mjs:robots.txt 加 Disallow /admin/(管理后台不收录)
> - **新踩坑**(记 27 号 #020):E407 漏定义致 raise_error 走 fallback E500(非 403),非 admin 调 admin 接口报 500 而非 403
> - **P0 端到端验证通过**:/admin/cards 200 + robots 屏蔽 + admin 调 stats OK + 非 admin 拦 403 E407 + verify_all 119/119

> **P0 后端开发记录**(2026-07-16,按 37 号 v1.1 执行):
> - 新增 modules/card_key/{models,service,api,open_api,auth,schemas}.py,main.py 注册 2 router(自带 prefix,不加 API_PREFIX)
> - auth.py /redeem 迁入 service.redeem(唯一兑换实现),保留旧路径兼容
> - 错误码补 E104-E106 + E501-E507;config 加 ADMIN_USERNAMES;security 加 get_current_admin
> - **新踩坑 3 个**(记入 27 号 #017-#019):
>   #017 HMAC 验签需 secret 明文,存哈希不可验签 → 改 Fernet 对称加密
>   #018 api_secret_hash 列 128 不够装 Fernet 密文 → 改 256
>   #019 .env 缺 POSTGRES_HOST 导致连 localhost 失败(api healthy 是假象,healthcheck 只查进程)→ 加服务名
> - **环境修复**:WSL .env 加 POSTGRES_HOST=postgres/REDIS_HOST=redis/MINIO_ENDPOINT=minio:9000;backend/.env 软链到根 .env(compose env_file 能读)
> - **端到端验证通过**:注册→登录→admin 生成卡密→兑换→余额;对外 claim→redeem-notify→幂等→跨入口互斥→错签拦截

> **37 号计划关键修订**(对齐代码现状,非改设计):
> ① 迁移重编号 0002→0003 起(0002 已被 guest_token 占用,36 号未察觉)
> ② P0 必含对外 Open API(用户核心需求:卡密后台生成 + 外部系统对接)
> ③ openapi.yaml 补全列为 P0 第一卡点(律一 Schema 优先,现状仅 8 基础接口严重滞后)
> ④ modules 渐进迁移(现有扁平 api/ 不破坏,新功能入 modules)
> 详见 [37-development-plan.md](./37-development-plan.md)

**待执行**(后续):
- [ ] 第3批:亲戚计算器扩关系/其他小工具按规范优化
- [ ] P2:开发者工具降级 + 合并
- [ ] P1:成语/诗词继续扩到 500+
- [ ] P2:开发者工具降级 + 合并
- [ ] P1:成语/诗词继续扩到 500+(当前 218/353,后续补充)

**待用户准备外部资源**(我无法代劳):
- [x] 域名注册(bibilabu.cc,粤ICP备2026096432号-1)
- [ ] SSL 证书(Cloudflare 泛证书免费)
- [ ] 备案(国内服务器必须,20 工作日)
- [ ] Linux VPS(2C4G,阿里云轻量)
- [ ] Windows Worker 机器 + WPS 商业版 License
- [ ] 虎皮椒支付账号
- [ ] 闲鱼/拼多多店铺

**用户拿到外部资源后**:
1. 填写 `.env.prod`(从 .env.prod.example 复制)
2. 放 SSL 证书到 `nginx/certs/`
3. 改 `nginx.conf` 的 server_name 为真实域名
4. 执行 `bash scripts/deploy_prod.sh`
5. 按 29-go-live-checklist-exec.md 验证

**接手须知**:
- 通用组件在 `frontend/src/components/`(含 QueryResult 查询结果组件)
- composable 在 `frontend/src/composables/useToolForm.ts`
- 加新工具参考已实现的 71 个页面
- 数据库类工具:backend=db-query,后端走 `/api/query/{tool_id}` 通用查询(见 backend/app/api/query.py)
- 代理类工具:backend=proxy,后端走 `/api/proxy/{action}`(见 backend/app/api/proxy.py)
- 付费工具:is_premium=true,前端检查余额(userStore.balance),不足跳转兑换页
- 批量改 registry 用 `scripts/add_batch*.py`
- 调试: `python scripts/diagnose.py --latest`(查前端报错)
- 全量验证: `python scripts/verify_all.py`(96 项)
- 工具验证: `python scripts/verify_tools.py --functional`(db/proxy 自动验证,frontend 标记需浏览器)
- 验证用例: `schemas/tests/{tool_id}.json`(77 个,含 31 个功能验证)
- 错题集: `27-lessons-learned.md`(必读)

---

## 版本历史

**基础设施增强**(v0.5.3,2026-07-03):
- [x] 通用组件 ToolInput/ToolButton/ToolResult(驱动 input_schema 自动渲染)
- [x] useToolForm composable(表单状态管理)
- [x] vite manualChunks 精细分割(zxcvbn/qrcode/crypto-js/element-plus 各自独立 chunk)
- [x] dataLoader.ts(数据文件懒加载)
- [x] update_registry.py / add_batch1_remaining.py(批量更新脚本)
- [x] tool.schema.json 扩展 input_schema(accept/rows/checkboxLabel/description/required)

**已实现工具**(30 个,v0.5.3,达成第 1 批目标):

PDF/付费(1):
- [x] pdf-to-word(WPS Worker,高质量 OCR)

电脑网络/加密(11):
- [x] random-password / uuid-gen / qrcode-generator
- [x] md5-hash / sha-hash / hmac-generator / aes-encrypt / file-hash
- [x] base64-encode / url-encode / html-escape

文本工具(6):
- [x] json-format / text-count / case-convert / text-dedup / text-diff / regex-test

计算工具(7):
- [x] timestamp-convert / base-convert / unit-convert / storage-unit / bmi-calculator
- [x] color-convert / morse-code

时间工具(3):
- [x] online-alarm / countdown-timer / world-clock

符号收集(2):
- [x] special-symbols / emoji-collection

**deprecated**(避免重复,2):
- timestamp(被 timestamp-convert 替代)
- base64(被 base64-encode 替代)

**验证结果**:
- ✅ registry 55 个工具(53 活跃 + 2 deprecated),一致性校验通过
- ✅ 前端构建成功(4.82s),30 个工具页各自独立 chunk
- ✅ Nginx 部署后 30 个路由全部 HTTP 200
- ✅ bundle 精细分割(zxcvbn 818KB 独立,各依赖按需加载)

**待执行**(对齐 22 号文档 12 周计划):
- [x] 第 1 批:30 个纯前端工具 ✅ 完成
- [ ] 第 2 批:30 个纯前端+静态数据(Week 3-4)— 万年历/节气/生肖/元素周期表等
- [ ] 第 3 批:30 个数据库工具(Week 5-6)
- [ ] 第 4 批:30 个后端 API+补齐(Week 7-8)
- [ ] 付费工具:PDF/证件照/压缩/简历(Week 9-10)

**接手须知**:
- 通用组件在 `frontend/src/components/`(ToolInput/ToolButton/ToolResult)
- composable 在 `frontend/src/composables/useToolForm.ts`
- 加新工具参考已实现的 30 个页面(用通用组件,约 30-80 行/个)
- 批量改 registry 用 `scripts/update_registry.py` 或 `add_batch1_remaining.py`
- 合入方案见 `24-tool-integration-plan.md`,操作见 `25-tool-operations-sop.md`

**接手须知**:
- 通用组件在 `frontend/src/components/`(ToolInput/ToolButton/ToolResult)
- composable 在 `frontend/src/composables/useToolForm.ts`
- 加新工具参考已实现的 21 个页面(用通用组件,约 30-50 行/个)
- 批量改 registry 用 `scripts/update_registry.py`
- 合入方案见 `24-tool-integration-plan.md`,操作见 `25-tool-operations-sop.md`
- 合入方案见 `24-tool-integration-plan.md`,操作见 `25-tool-operations-sop.md`
- 加新工具:改 registry → 创建页面 → 加 implementedTools → gen:tools → CI 校验
- 现有 33 个工具(30 原有 + 3 示例),已验证合入流程可行

---

## 版本历史

### v1.2.1 (2026-07-09) - SEO T1/T2 内容增强 + 域名 SSOT

**对应文档**: [33-seo-optimization.md](./33-seo-optimization.md) §4(T1)/§5(T2)/§十一(决策+T3)

**T1 内容 SEO**:
- [x] 重复 id 清理 — `dedupe_registry.py` 删除 5 个较不完整重复条目,110→105(103 活跃),validate 重复警告 0
- [x] 同质化 title 重算 — `t1_seo_content.py` 改进公式(挑与 name 不重叠的关键词),33 个 title 重算,残留 0
- [x] 8 个高搜索量工具精选内容 — tax/mortgage/old-almanac/retirement/simplified-traditional/pinyin/interest/relatives
  每个补 how_to + description_for_ai + 3-4 条 FAQ;FAQ 自动喂 FAQPage JSON-LD(实测 tax-calculator 含 4 Question)
- [x] JSON-LD 验证 — 高搜索量工具 dist HTML 含 FAQPage 富结果

**T2 GEO**:
- [x] llms.txt 用 description_for_ai — 8 个工具有精选 AI 描述,llms.txt 不再纯 description 兜底
- [x] llms.txt/sitemap/robots 自动生成链路稳定(仅已实现工具,按 id 去重)

**域名占位 SSOT(为 T3 铺路)**:
- [x] 新增 `frontend/src/config/site.json`(domain/name SSOT)
- [x] canonical/sitemap/llms/robots 全部读 site.json — 换域名从"改 4 处"降到"改 1 处(site.json)+ nginx"
- [x] ToolLayout/Home import site.json;gen-sitemap 生成 robots.txt(域名单一来源)
- [x] `DOMAIN_PLACEHOLDER` 标记 + 一键复核 `grep -rn DOMAIN_PLACEHOLDER frontend/ nginx/`
- [x] nginx server_name 加标记(部署期配置,无法读 JSON)

**回滚基线**:git commit `67a4ffb`(T0 基线)。T1/T2 在其后。撤销 T1/T2:`git reset --hard 67a4ffb`。
**已决断(见 33 号 §11.1)**:重复 id 保留更完整条目、同质化 title 公式、路由统一子目录、域名集中化 site.json。

**验证**:build exit=0,EP 0,105 页面,dist 三件套齐全,canonical 来自 site.json,validate 通过(0 警告)。

**T3 待执行(域名确定后,见 33 号 §11.4)**:
- [ ] T3-1..10:域名注册/Cloudflare/备案 → 改 site.json+nginx 重部署 → 三大站长验证+提交 sitemap → 百度主动推送 → 统计 → 监控收录
- [ ] 政策类文案(个税/退休)法务复核(可选)

### v1.2.0 (2026-07-09) - SEO T0 技术基础落地

**对应文档**: [33-seo-optimization.md](./33-seo-optimization.md)

**致命缺口修复**:vite-ssg 早已在依赖但未接入 → 现已接入,站点从纯 CSR(SPA)改为预渲染静态站(SSG)。
爬虫抓到的是真实 SSR 内容(工具名/about/70 互链),不再是空 `<div id="app">`。这是全站 SEO 的前提。

**已落地(T0)**:
- [x] vite-ssg 接入 — main.ts 用 ViteSSG 入口,router 导出 routes,vite.config 加 ssgOptions(排除 /redeem /user-center)
- [x] SSR 守卫 — user store / errorTracker / user-agent 三处浏览器 API 加 `typeof window` / `onMounted` 守卫
- [x] Element Plus SSR — provide ID/ZINDEX_INJECTION_KEY,消除 hydration 警告(build EP 错误 0)
- [x] 每页 head(useHead @unhead/vue) — title/description/keywords/canonical/OG/robots,108 工具条目+首页独立
- [x] JSON-LD 自动生成(恒输出) — WebApplication + FAQPage(有FAQ时) + BreadcrumbList,走 useHead script
  (注:模板内 `<script>` 会被 Vue 编译器移除,旧 `<script v-html>` 实际失效,已修正为 useHead)
- [x] 占位薄页 noindex — 16 个占位工具(_placeholder) robots=noindex,follow,不进 sitemap
- [x] sitemap.xml — gen-sitemap.mjs 从 registry 生成,87 已实现工具+首页(按 id 去重,无重复)
- [x] llms.txt 自动生成 — 从 registry 按分类,仅已实现工具;修正旧 llms 路径错(/tools/x→/x)+ 列未实现工具问题
- [x] robots.txt/llms.txt 移至 frontend/public/(vite publicDir),确保进 dist(原仅项目根 public/ 不会被打包)
- [x] nginx try_files 加 `$uri.html`(SSG 产物是 ${route}.html,否则会回退首页)+ sitemap.xml location
- [x] **SEO 护栏(CI)** — validate.mjs:非 deprecated 工具必须填 seo.title/description,缺则 exit 1;加重复 id 检测
- [x] seo 字段全量补全 — gen_seo_fields.py 派生 108/108 工具 seo(fallback 公式,T1 人工打磨)
- [x] schema 扩展 — tool.schema.json seo 加 title/description/keywords 定义
- [x] README/CLAUDE 加入 33 号文档导航 + 禁止行为护栏

**发现的数据 bug(待 T1 清理)**:registry.json 有 5 个重复 id 条目
(text-diff/regex-test/url-encode/base-convert/unit-convert 各 2 条)。validate 已加检测告警,
gen-sitemap 按 id 去重(sitemap 干净,88 URL=87 工具+首页,无重复)。registry 重复条目需人工确认保留哪条后删除。

**验证**:build exit=0,EP 错误 0,105 页面预渲染,dist 三件套(robots/llms/sitemap)齐全,
tax-calculator.html 含独立 title/canonical/index,follow + 3 块 JSON-LD,占位页 noindex,validate 通过(5 重复 warning)。

**生成命令(加新工具后必跑)**:
- `npm run build` = gen-sitemap + vite-ssg build(产出 SSG dist,SEO 资源一并生成)
- `node scripts/gen-sitemap.mjs` = 生成 sitemap.xml + llms.txt(仅已实现工具,自动随 registry,按 id 去重)
- `python scripts/gen_seo_fields.py` = 派生补全缺失 seo(幂等)
- `node scripts/validate.mjs` = SEO 护栏 + 重复 id 检测(缺 seo.title/description 即拦截)

**接手须知(SEO)**:
- 加新工具:实现页面 → gen:tools → gen_seo_fields.py(补 seo,或手填更佳) → validate 必过 → build 产出 SSG
- robots/llms/sitemap 以 frontend/public/ 为准(gen-sitemap 自动生成 llms/sitemap;robots 静态)
- 域名 `https://zhiniao.tools` 为占位,域名到位后改 4 处:ToolLayout 的 SITE_DOMAIN、gen-sitemap 的 SITE、robots 的 Sitemap 行、index.html og
- vite-ssg build 时 Math.random() 互链被固化(SSG 构建期),利于 SEO 稳定,可接受

**待办(T1/T2/T3,见 33 号文档)**:
- [ ] T1:seo.title/description 人工打磨(当前 fallback 公式同质化,如"个税计算器 - 个税计算器")
- [ ] T1:清理 registry 5 个重复 id 条目(确认保留哪条)
- [ ] T1:扩 FAQ/how_to/description_for_ai(高搜索量工具优先,SEO+GEO 双用)
- [ ] T2:llms.txt 的 description_for_ai 内容待补(当前用 description 兜底)
- [ ] T3:域名/备案/三大站长验证/sitemap 提交/百度主动推送/统计(需外部资源)

### v0.4.0-dev (2026-07-03) - 平台转型启动

**重大决策**: 从"单品 PDF 转 Word 副业"转向"bmcx 模式便民工具平台"

**转型理由**:
1. PDF 转 Word 是红海市场(SmallPDF/iLovePDF/WPS 已统治)
2. 闲鱼渠道月收入预测仅 30-90 元,投入产出比低
3. 重架构(微服务/5 类 Worker)与副业轻量化矛盾
4. bmcx.com 模式验证 10 年+,流量稳定

**新方案核心**:
- 对标 bmcx.com,8 大类共 120 个便民工具
- 子域名结构(`{拼音}.yourdomain.com`)
- 工具互链(每页 70 个相关链接)
- 极简架构:Vue3 + SQLite + FastAPI(单 VPS)
- 月固定成本 ~80 元(目标净利 720-2220 元/月)

**新增文档**:
- `22-platform-pivot-strategy.md` — 平台转型战略与 120 工具清单
- `23-quick-start-operations.md` — 每日执行清单(可勾选)

**12 周计划**:
- Week 1-2:第 1 批 30 个纯前端工具
- Week 3-4:第 2 批 30 个纯前端+静态数据
- Week 5-6:第 3 批 30 个数据库类工具
- Week 7-8:第 4 批 30 个后端+补齐
- Week 9-10:付费工具上线(PDF/证件照/压缩/简历)
- Week 11-12:优化与监控

**与 v1.0.1 本地部署的关系**: 本地部署验证通过的单品架构作为"付费工具线"保留,平台化后第 9-10 周接入。

### v0.3.0-dev (2026-07-02) - P1 基础设施进行中

**目标**: 后端骨架 + 数据库 + 队列 + 存储 + CI 校验

**P1 验收门禁**:
- [ ] docker compose up 一键启动
- [ ] 健康检查全绿
- [ ] 数据库迁移成功
- [ ] CI 一致性校验通过
- [ ] OpenAPI 契约一致

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
| 域名注册 | 中 | ✅ 已完成 | bibilabu.cc(网站首页 www.bilibabu.cc) |
| 支付通道资质 | 中 | 待定 | 个人/企业 |
| 备案 | 中 | ✅ 已完成 | 粤ICP备2026096432号-1,珠海友米网络科技有限公司,2026-07-17 落地(页脚 ICP + site.json/nginx 域名同步) |

---

## 文档变更记录

| 日期 | 文档 | 变更 |
|------|------|------|
| 2026-07-02 | 全部 | 初始创建 |
| 2026-07-02 | 10/11/12 + README/CLAUDE/01 | Worker 架构调整 + 实施流程总纲 |
| 2026-07-17 | 33/18/01 + site.json/nginx/openapi/tool.schema/.env.prod.example | 备案落地:粤ICP备2026096432号-1(珠海友米网络),域名 zhiniao.tools→bibilabu.cc,页脚展示 ICP 号+公司主体,site.json 加 company/icp 字段,代码区 DOMAIN_PLACEHOLDER 占位标记全清 |
| 2026-07-17 | 27/36/38 + wechat_native.py/payment.py/config.py/compose | 微信支付 V3 真实下单接入:证书入 secrets+compose 挂载,wechat_native 重写(V3 下单 RSA-SHA256+回调验签解密),callback/wechat 路由,后台测试连通=真连下单。踩坑 #036(签名顺序)/#037(微信支付公钥模式)。真连拿真实 code_url,verify_all 132/132。待办:下载微信支付公钥启严格回调验签;公网 DNS+SSL 就绪回调方能触达 |
| 2026-07-20 | 39/36/38/01 + Checkout/4组件/store/composables/modules.payment/nginx/openapi | 在线支付改造 P0-P2(39号SSOT):P0 出二维码(QrCode 组件+舞台模式+套餐对齐+order store)+P1 可靠性(cancel 端点+超时清理+自动兑换+payment_log+checkout 兜底+轮询优化+nginx 限流)+P2 安全(验签 fail-fast+回调走适配器+金额校验+删 TODO_SECRET) |
| 2026-07-20 | 39/27/36 + packages.py/Pricing.vue/registry/growth模块/auth/迁移0008 | 定价方案重新设计:4档诱饵锚定提价(single19/monthly29/yearly199+original_price)+计费文案对齐"消耗1点"+引流增长(注册送3点+IP防刷+限时倒计时+支付后回工具+邀请/签到裂变 modules/growth+迁移0008)。SSOT收敛(价格3副本→1处)。e2e:注册送3/邀请6/自邀不发/签到+429全过,verify_all 136/136。踩坑:_log_action(None)触发 card_key_id NOT NULL(改不落卡密日志) |
| 2026-07-20 | 39 + packages.py/order.py/order_service/payment/promotion模块/迁移0009/Pricing/Checkout/admin/Promotions | 限时优惠双机制+按次套餐:P0 按次套餐(trial_1¥1/pack_5¥3)+P1 优惠生效防绕过(create_order price_tier后端重算amount+Order加列+payment_log拼tier)+P2 后台激活(promotions表+modules/promotion+admin/Promotions.vue)+P3 新用户专享(created_at判定monthly¥19)。优先级活动>新用户>原价。e2e:按次套餐/活动价¥15/新用户¥19/防绕过全过,verify_all 139/139 |
| 2026-07-20 | 40(新增) | 用户系统交互设计文档:诊断后端能力完整+前端缺口(退出登录链路断裂/切换账号缺失/找回密码UI零实现/用户中心能力浪费/头像/邀请码闭环等11项),设计完整交互链路(导航栏下拉菜单+退出调后端吊销+切换账号+找回密码三步页+用户中心6Tab+资料头像+安全强校验),分P0-P5路线。待落地 |
| 2026-07-20 | 40 P0 + user.ts/DefaultLayout.vue/UserCenter.vue/client.ts | 用户系统P0落地:退出登录调后端 /auth/logout 吊销 token(user store logout async+authApi.logout)+导航栏用户下拉菜单(用户中心/切换账号/退出登录)+切换账号(退出+带 redirect)+401拦截提示"登录已过期"+带 redirect(G10/G12)。e2e:logout后 session.revoked_at置位+旧token 401。verify_all 139/140(三处对齐失败为既有registry问题,非P0引入) |
| 2026-07-20 | 40 P1 + ForgotPassword.vue/router/Login.vue/client.ts | 用户系统P1落地:找回密码UI(后端+封装就绪,原零UI)。新建 pages/auth/ForgotPassword.vue 三步流程(邮箱→验证码→新密码,步骤指示器+60s重发倒计时+二次确认);client 补 verifyCode;路由 /forgot-password;Login 加"忘记密码?"入口。e2e:forgot发码→verify校验→reset重置→旧token401(吊销所有会话)+不存在邮箱也回成功(防枚举)。线上 /forgot-password 200 + Login 入口可见 |
| 2026-07-20 | 40 P2 + UserCenter.vue/client.ts | 用户系统P2落地:用户中心改造。顶部固定区接 dashboard(余额点数+配额进度+统计 counts/orders/card_keys/convert_tasks)+加交易记录Tab(订单+卡密二级切换,接 getOrders/getCardKeys,状态徽章+套餐名+脱敏卡密);保留福利Tab;client getDashboard/getOrders/getCardKeys 接通(原封装未用)。e2e:dashboard 余额+counts 返回正确,下单后 counts.orders=1,卡密未兑换=0。线上 /user-center 200 |
| 2026-07-20 | 40 P3 + Login.vue/user.ts/client.ts/modules/user/api.py/openapi | 用户系统P3落地:邀请码闭环+改邮箱。P3-A:Login 注册Tab加邀请码字段(query.invite 预填+自动切注册)+user store/client register 透传 invite_code;P3-B:后端加 POST /api/user/change-email/send+change-email(验证码基建 purpose=change_email,60s冷却+防重复+校验)+前端资料Tab改邮箱 UI+client changeEmailSend/changeEmail。e2e:邀请码注册双方加点(邀请人3+2=5/被邀人3+3=6/自邀不发)+改邮箱发码验证更新 profile+错码E302。openapi 59路径 |
| 2026-07-20 | 40 P4 + router/user.py/auth.py/main.py/user.ts/UserCenter/client/openapi/0011修复 | 用户系统P4落地:安全合规。P4-1:/user-center 加 requiresAuth;P4-2:注销冷静期(deactivate改 status=deactivating+deactivated_at,reactivate 凭邮箱验证码不依赖登录,硬删定时任务 status=deleted,login 对 deactivating 提示撤销);P4-3:isAdmin 去硬编码(后端 UserResponse 返回 role,前端 store 读 role);P4-4:注销强校验(输当前密码)。踩坑:外部已加 0010_convert_credit_logs+0011_user_deactivate,我误建 0010_user_deactivate 重复→删并 stamp 0011,修 0011 用 IF NOT EXISTS 避 updated_at 重复加。e2e:role=user/注销错密码401/正确→deactivating+token401/登录403撤销提示/reactivate恢复/重登200。verify_all 140/140 |
