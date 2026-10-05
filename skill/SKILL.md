---
name: xhs-corpus
description: 小红书语料采集工作流——搜索池→正文→图卡转录→知识库沉淀的全套方法（登录态IAB、防风控节奏、断点续采）
version: 1.0.0
license: MIT
origin: 2026-10-05 实战验证于考研英语方法大搜索（57关键词/701池/211帖全文）与「怎么说」博主专项（223篇/27页图卡转录）
---

# xhs-corpus · 小红书语料采集工作流

把"刷小红书"变成可复现、可续跑、可沉淀的知识生产流水线。适用于：方法论采集、博主专项、竞品调研、题库/攻略库、任何"搜索→阅读→结构化落盘"的长任务。

> 本 skill 假设运行环境有一个**可编程的真实浏览器通道**（ZCode IAB / browser-use / CDP 注入均可），
> 且用户已完成登录。核心技巧对任何能发真实输入事件的通道有效。

## 0. 硬纪律（先读）

1. **采集中立**：只做信息收集，不加内容价值判断；评论者原话照录，解读权归用户。
2. **前台可见优先**：只读页面上肉眼可见的内容，不调接口、不注入脚本抓数据、不模拟协议。
3. **像真人**：每次导航/点击间隔 2.5-3.5 秒随机停顿；连续 10+ 次操作后歇 8-15 秒；出现验证码/风控页 → STOPPED_SAFE 立即停，记检查点。
4. **断点续采**：所有状态落盘（JSONL 追加式），任何中断从文件恢复，绝不重采。
5. **不加戏**：历史落盘 ≠ 本轮重新采集；工具可调用 ≠ 执行成功。只声称观察到的事实。

## 1. 通道与登录

- 打开目标站点，检查登录态（侧边栏有无"我/通知/消息"入口）。
- 登录态存于浏览器分区的 Cookie，落盘持久；失效重扫码即可。
- 首次无登录态 → 开登录页 → 停手等用户扫码。

## 2. 搜索池模式（元数据层）

**目的**：把 N 组关键词的搜索结果变成一个去重的候选池。

```
关键词矩阵（广义、多轮：同义词/黑话/品牌词/分数档/情绪词）
  → 每词 goto 搜索页
  → 页内直采卡片（标题+id+token+检索词）
  → JSONL 追加落盘（断点续采：每次启动先读文件 skip 已采）
  → 去重累积为"搜索池"
```

页内直采（比 DOM 快照快且稳）：

```js
// 在浏览器通道的 evaluate 中执行（IIFE 形式，函数字面量不会被调用）
JSON.stringify([...document.querySelectorAll('a[href*="/search_result/"], a[href*="/explore/"]')]
  .map(a => a.getAttribute('href'))
  .filter(h => h.includes('xsec_token='))
  .map(h => {
    const m = h.match(/search_result.([0-9a-f]{24}).*xsec_token=([A-Za-z0-9_%=-]+)/);
    return m ? [m[1], decodeURIComponent(m[2])] : null;
  }).filter(Boolean))
```

**关键词设计心法**（越野越好）：同义词轮换（邪修/野路子/秒杀/大法/白嫖/穷鬼）、分数档（94分/80分/逆袭）、人群（零基础/二战/在职）、品牌与人物名、情绪词（保命/救命/劝退）、组合词（领域+方法/技巧/规划/复盘）。

**已知坑**：
- 品牌词/人名搜索常落在视频/用户结果页，卡片 href 结构不同 → 0 命中。变体：加"笔记/方法"后缀重试。
- 同名不同物：泛词会撞实体店/影视等（如"吉大拉面哥"是面馆），靠标题过滤。
- 搜索结果流每次加载会重排，卡片位置不固定——不要依赖位置，只认 id。

## 3. 正文采集（单帖协议）

```
筛选（分桶限额：按主题把池子切成优先级桶，每桶限量）
  → 逐帖 goto 详情页
  → sleep 2.8s → evaluate 抓 {title, author, desc(前2500字), 评论数}
  → 失败重试一次（sleep 2.5s 后再 evaluate）
  → JSONL 追加落盘
```

```js
const grab = async () => tab.playwright.evaluate(`(() => {
  const title = (document.querySelector('#detail-title, [class*="title"]')?.textContent || '').trim();
  const desc  = (document.querySelector('#detail-desc, [class*="desc"]')?.innerText || '').trim();
  const author = (document.querySelector('[class*="author"] [class*="name"], .username')?.textContent || '').trim();
  const count = (() => { const e = [...document.querySelectorAll('*')].find(x => /共 \\d+ 条评论/.test(x.textContent) && x.childElementCount === 0); return e ? e.textContent.trim() : ''; })();
  return JSON.stringify({ title, author, desc: desc.slice(0, 2500), count });
})()`);
```

**节奏锚点**：每帖约 8-10 秒；单批次循环加 `if (Date.now() - t0 > 92000) break;` 防超时；每 call 之间状态全部来自磁盘文件。

## 4. 点击与滚动（平台技巧，踩坑换来的）

- **locator.click 大概率超时；合成事件（dispatchEvent）被无视**。唯一可靠：
  `evaluate（IIFE）取元素坐标 scrollIntoView + 真实坐标点击（cua.click / CDP input）`。
- 点击前必须 `document.elementFromPoint(cx, cy)` 验证落点（透明遮罩/频道栏会截胡）。
- 页面滚动挂在 `document.documentElement.scrollTop`（window.scrollBy 无效）；帖内评论在 `.note-scroller`。
- evaluate 传参必须是 **IIFE 立即执行表达式**，函数字面量不会被调用。
- 评论展开按钮：`/^展开 \d+ 条回复$/` 或 `展开更多回复`，逐个单步点，每步验证。

## 5. 评论层

- **网页端评论封顶：约 10 个主楼 + 回复展开全量，无分页入口**。标称总数（如2203）不可达全量——这是平台限制，不绕过，如实记录"可达上限"。
- 评论区策略：主楼全录 → 逐条展开回复链（单步+验证）→ 按**赞数**标记精华。
- 视频帖：正文只有一句话，方法在视频内无法文本提取——**用评论区（作者答疑+用户共创）替代**，价值常不亚于正文。

## 6. 图卡帖（内容在图片里）

判定：正文文字 < 50 字 + 存在 `.pagination-item` 圆点/`N/M` 指示器 → 内容是图卡。

流程：轮播翻页 → 每卡截图 → 读图逐字转录（OCR 工具不稳时直接用多模态 Read）→ 与评论区"@官方AI 提取全文"的转录**互校**（官方AI转录常被平台长度截断，只当校验源）。

翻页两套姿势：
1. **详情页轮播**：`.arrow-controller.right/left` 坐标点击；点击偶尔不生效→先点空白处再点、单步慢点；圆点直达 `.pagination-item`。
2. **大图查看器**：点主图打开 → **键盘 ArrowRight/ArrowLeft 翻页（最稳）** → 真实页码读 `.preview-toolbar-pager-num`（⚠️ 页面双层视图时 ARIA 指示器会读到被盖住的旧层，别信）→ 截图前等 2.5s+（翻页动画会出残影帧）。
3. 防膨胀：翻完即删中间帧，只留封面 1 张 + 内容卡。

## 7. 落盘结构（语料库）

```
outputs/
  <栏目>_搜索池.jsonl          元数据层（id/token/kw/title）
  <栏目>_正文采集.jsonl        正文层
  <栏目>_<题>_可见页面原文.txt  原始留档
  <栏目>_<题>_详情截图.png      每帖 1 张封面（防膨胀）
  <栏目>_<题>_图卡转录.md      逐字转录（保错别字/emoji，模糊标[?]）
library/
  index.md                    总索引
  posts/Pxxx.md               帖子元数据+观察
  comments/Pxxx-comments-*.md 评论分片（编号连续）
  候选帖-*.md                 未深挖的候选
  进度检查点-*.md             ACTIVE_OBJECT / NEXT_EXECUTABLE_ACTION / STOPPED_SAFE 原因
```

**验收事件词汇**：CAPTURE_STARTED / NEW_VISIBLE_COMMENTS_WRITTEN / SCREENSHOT_VERIFIED / CHECKPOINT_UPDATED / STOPPED_SAFE。

## 8. 知识库沉淀（Obsidian/任意 vault）

- 三层：`10-Projects/<项目>/`（目标与进度）+ `30-Resources/<项目>/`（交付笔记，每帖 1 张封面图）+ `90-AI/<项目>/`（工作流经验）。
- 交付笔记 = 方法论萃取（从逐字转录中提炼结构），原始件路径引用进去。
- **每次写入盖 `updated` 当天日期**；批量改动后 git commit。
- 用户原话存 verbatim 层（逐字，不改写）。

## 9. 成品聚合

JSONL → Python 聚合 → Markdown（分桶/分类、按正文长度排序、原始件路径引用）。聚合脚本要点：
- 从 jsonl 读记录，按桶分组，桶内按 desc 长度降序
- 视频帖标"（方法在视频内）"
- 页码覆盖说明（图卡帖必须注明哪些页转录了、哪些缺）

## 10. 边界与红线

- 不绕登录、不破解风控、不批量并发请求、不发内容不互动（只读）。
- 平台限制如实记录（评论可达上限、视频不可提取），标注"平台限制不绕过"，不冒充采集失败或成功。
- token 有时效，过期重搜获取；不缓存转发 token 给第三方。
- 收尾：把进度写检查点，把新坑写回本 SKILL 的对应章节。
