# 竞品活动洞察项目 — 构建指引

## 项目目标
抓取头部交易所活动公告页，分类整理、提取洞察，生成静态 HTML 部署到 Vercel，供 Pionex 运营参考。

## 已完成的交易所
| 交易所 | 脚本 | 数据文件 | HTML | 状态 |
|--------|------|---------|------|------|
| Gate.io | `scrape_gate.js` | `gate_articles.json` + `gate_knowledge.json` | `gate_2026.html` | ✅ 已上线 |
| Binance | `scrape_binance.js` | `binance_articles.json` + `binance_knowledge.json` | `binance_2026.html` | ✅ 已上线 |

## 待完成的交易所
| 交易所 | 公告页地址 |
|--------|-----------|
| OKX | https://www.okx.com/zh-hans/help/category/announcements |
| Bybit | https://announcements.bybit.com/zh-MY/ |
| KuCoin | https://www.kucoin.com/zh-hant/news/categories/announcement |
| Bitget | https://www.bitget.com/zh-CN/support/news/announcement |
| MEXC | https://www.mexc.com/zh-CN/support/articles/ |
| HTX | https://www.htx.com/zh-cn/support/articles/ |

## 文件结构
```
deposit_check/
  index.html               # 世界杯活动（已上线，已有）
  gate_2026.html           # Gate.io 活动（自动生成，勿手动改）
  gate_articles.json       # Gate.io 全量文章存储（持久化，关键）
  gate_knowledge.json      # Gate.io 人工洞察/badges/简介（人工维护）
  scrape_gate.js           # Gate.io 抓取脚本
  binance_2026.html        # Binance 活动（自动生成，勿手动改）
  binance_articles.json    # Binance 全量文章存储（持久化，关键）
  binance_knowledge.json   # Binance 人工洞察/badges/简介（人工维护）
  scrape_binance.js        # Binance 抓取脚本
  CLAUDE.md                # 本文件
```

## 每家交易所的标准三件套
1. **`{exchange}_articles.json`** — 持久化文章 store，每次 upsert（新文章追加，旧文章更新阅读量）
2. **`{exchange}_knowledge.json`** — 人工洞察、badges、活动简介，不会被脚本覆盖
3. **`scrape_{exchange}.js`** — 抓取脚本，`node scrape_{exchange}.js` 一键运行

## Gate.io 已验证的 DOM 结构（2026-05）
```
公告列表页：https://www.gate.com/zh/announcements/activity
每张活动卡片结构（<a> 元素）：
  div.mb-2.line-clamp-2          → 标题
  svg[data-icon-id*="circlefilled_progress"] + span → 时间
  svg[data-icon-id*="Show"] + span                  → 阅读量
分页：Mantine <button> 文本匹配 "2"/"3"/"4"
注意：SVG 是 inline path（无 <use> 子元素），data-icon-id 在 <svg> 上
抓取 4 页 = 约 60 条
```

## scrape_gate.js 运行流程
1. 检查 CDP Proxy（localhost:3456）是否运行
2. 加载 `gate_knowledge.json` 和 `gate_articles.json`
3. 用 CDP 抓取 Gate.io 公告页 4 页
4. Upsert 到 `gate_articles.json`（新文章追加，旧文章更新 views/time）
5. 从全量 store 生成 HTML（knowledge 中有的用已有内容，无的留占位）
6. git add + commit + push → Vercel 自动部署

## 分类规则（关键词匹配，按优先级）
```js
{ id: 'routine',  re: /合约积分空投第/ },
{ id: 'trade',    re: /交易大赛|交易竞赛|合约赛|交易赛|机器人.*赛|拉力赛|打卡热力|爆量狂欢/ },
{ id: 'social',   re: /Gate Live|直播嘉年华|广场.*分享|广场.*发帖|Booster|一分钟掌握/ },
{ id: 'vip',      re: /\bVIP\b|护航计划/ },
{ id: 'referral', re: /邀请|邀友|推荐好友|返佣/ },
{ id: 'airdrop',  re: /CandyDrop|BountyDrop|首发上线|新币上线|Launchpool|空投/ },
{ id: 'newuser',  re: /新人专场|闪兑|入金|充值|首兑|首单/ },
{ id: 'event',    re: /疯狂星期三|周年|五一|签启/ },
```

## 热门阈值
- `views >= 5000` → `hot`（橙色）
- `views >= 10000` → `very-hot`（金色发光）

## Vercel 部署
- 仓库：https://github.com/y-cat-van/worldcup2022_crypto_campaigns
- 访问地址：https://worldcup2022-crypto-campaigns.vercel.app/
- 每次 push main 自动触发重新部署
- 各页面路径：`/gate_2026.html`、`/binance_2026.html` 等

## CDP Proxy 说明
- 用 Claude Code 内置的 web-access skill 启动
- Proxy 监听 localhost:3456
- 所有 CDP 操作通过 curl HTTP API（`/new`、`/eval`、`/scroll`、`/click`、`/close`）
- 使用用户本机 Chrome（携带登录态、cookies），比 headless 更可靠

## Binance 已验证的抓取方式（2026-05）
```
公告列表页：https://www.binance.com/zh-CN/support/announcement/list/93
抓取方式：内部 API（不用 DOM 解析，速度快，稳定）
API 端点：/bapi/composite/v1/public/cms/article/list/query
  参数：type=1, pageNo=N, pageSize=20, catalogId=93
  Headers: lang=zh-CN, clienttype=web
  Response: data.catalogs[0].articles → [{id, code, title, releaseDate}]
  total: 约 1950 条
注意：
  - 翻页不可用 DOM click（CDP 环境下 Binance 请求被 block）
  - 必须通过 CDP tab 调用 API（携带 cookies），直接 curl 无法用
  - releaseDate 是毫秒时间戳，无阅读量数据
  - URL 格式：/zh-CN/support/announcement/detail/{code}
  - 每次抓 5 页 × 20 条 = 100 条（已足够）
```

## scrape_binance.js 运行流程
1. 检查 CDP Proxy（localhost:3456）是否运行
2. 加载 `binance_knowledge.json` 和 `binance_articles.json`
3. 打开 Binance 公告页 tab（等待 4s 让 cookie 就绪）
4. 通过 CDP eval 调用内部 API 抓取 5 页（100 条）
5. Upsert 到 `binance_articles.json`
6. 从全量 store 生成 HTML
7. git add + commit + push → Vercel 自动部署

## Binance 分类规则
```js
{ id: 'learn',      re: /学院|答题|完成测试|比特币.*专题|以太坊.*专题/ },
{ id: 'launchpool', re: /Launchpool|Megadrop|HODLer.*空投|新币.*上线|首发上线|上架/ },
{ id: 'trade',      re: /锦标赛|交易赛|体验赛|交易大赛|交易量.*竞赛|排行榜/ },
{ id: 'referral',   re: /邀请|推荐好友|返佣|超级返佣|招募/ },
{ id: 'newuser',    re: /新用户|首次入金|充值.*奖励|买币.*奖励|迎新|入金礼|注册/ },
{ id: 'finance',    re: /借币|活期|申购|赚币|理财|质押|Loans|固定利率/ },
{ id: 'vip',        re: /\bVIP\b|机构用户|做市/ },
{ id: 'event',      re: /周年|五一|节日|大学生|圣诞|春节|世界杯|Score with/ },
```

## 通用 HTML 样式规范
- 背景 `#0f1117`，侧边栏 `#13151f`，卡片 `#1a1d27`
- 强调色 `#818cf8`（紫），热门橙 `#d97706`，爆款金 `#f59e0b`
- 左侧 sticky sidebar 导航，主区域卡片列表
- 每个类别顶部有洞察框（insight-box）+ 可选 sub-pattern
- 卡片含：标题链接、热门徽章、时间+阅读量、badges、活动简介
