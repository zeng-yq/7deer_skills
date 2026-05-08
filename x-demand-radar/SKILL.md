---
name: x-demand-radar
description: |
  AI 热点雷达 - 在新 AI 工具/关键词爆火前发现它们，抢注域名、建站套利。

  触发条件：
  - 用户说「跑雷达」、「热点扫描」、「X 雷达」
  - cron 每 12 小时自动触发（9:00, 21:00）

  核心目标：
  在 AI 关键词/工具首次病毒传播 → 大众认知的 24-72h 窗口内发现，
  抢注域名 + 建工具站 + 吃搜索流量红利。

  典型案例：Nano Banana、Ghibli AI、OpenClaw、Hermes
---

# X AI 热点雷达

## 核心指标

**最终得分 = X 热度 × 项目新鲜度**

- `raw_hot_score = likes × log(1 + likes_per_hour)`
- 增速 = 点赞数 / (当前时间 - 发布时间) 小时
- **项目新鲜度校验**：发现 GitHub 项目后，必须导航到仓库页面验证创建日期
- `final_score = raw_hot_score × freshness_multiplier`
  - 项目 < 7 天：× 1.0
  - 项目 7-30 天：× 0.7
  - 项目 30-90 天：× 0.3
  - 项目 > 90 天：标记 ⚠️ 旧项目回锅，不参与排名

**🔴 关键教训**：UI-TARS-desktop (bytedance) 29K 星但 repo 已存在数月，X 帖子虽然是新的但项目不新 → 套利空间归零。热度高 ≠ 项目新。

## 搜索矩阵（4 层探测器）

每层目标不同，覆盖新品发布 → 病毒传播 → 开源爆发 → FOMO 需求全链路。

### L1: 新品发射（24h 窗口）
捕捉"刚刚发布/开源"的 AI 工具，最早信号。

```
("just launched" OR "just dropped" OR "just shipped" OR "new AI tool" OR "just released" OR "introducing" OR "launching today") (AI OR LLM OR GPT OR agent OR open source) min_faves:30 since:SINCE_24H
```

### L2: 病毒传播（48h 窗口）
捕捉正在被疯转的 AI 产品。

```
("this is insane" OR "game changer" OR "holy shit" OR "mind blown" OR "this is crazy" OR "cannot believe" OR "blown away") (AI OR LLM OR GPT OR agent OR tool) min_faves:100 since:SINCE_48H
```

### L3: GitHub 星标暴增（48h 窗口）
开源 AI 项目突然爆火。

```
("github.com" OR "open source") (AI OR LLM OR GPT OR agent) ("stars" OR "trending" OR "blew up" OR "blowing up" OR "just hit") min_faves:100 since:SINCE_48H
```

### L4: FOMO / Waitlist（72h 窗口）
等不及、求邀请、排长队 = 需求溢出信号。

```
("waitlist" OR "beta access" OR "invite" OR "early access" OR "can't wait" OR "need this") (AI OR LLM OR tool OR app) min_faves:50 since:SINCE_72H
```

## 执行流程

```
[Cron: 每12小时]
       ↓
[浏览器注入 X Cookie → 已登录状态]
       ↓
[L1-L4 逐层搜索 → JS 滚动收集]
       ↓
[合并去重 → 计算 raw_hot_score]
       ↓
[🔍 Step 2.5: 项目新鲜度校验]
       ↓
[AI 提取：关键词 + 产品名 + 项目年龄 + 一句话总结]
       ↓
[Top 10 排序 → 生成报告]
       ↓
[直接发送飞书]
```

## Step 1: 构造搜索 URL

```javascript
const now = new Date();
const since24h = new Date(now - 24*60*60*1000).toISOString().split('T')[0];
const since48h = new Date(now - 48*60*60*1000).toISOString().split('T')[0];
const since72h = new Date(now - 72*60*60*1000).toISOString().split('T')[0];

const SEARCH_GROUPS = [
  {
    layer: 'L1-新品发射',
    q: `("just launched" OR "just dropped" OR "just shipped" OR "new AI tool" OR "just released" OR "introducing") (AI OR LLM OR GPT OR agent OR "open source") min_faves:30 since:${since24h}`,
    since: since24h
  },
  {
    layer: 'L2-病毒传播',
    q: `("this is insane" OR "game changer" OR "holy shit" OR "mind blown" OR "this is crazy" OR "cannot believe" OR "blown away") (AI OR LLM OR GPT OR agent OR tool) min_faves:100 since:${since48h}`,
    since: since48h
  },
  {
    layer: 'L3-GitHub爆发',
    q: `("github.com" OR "open source") (AI OR LLM OR GPT OR agent) ("stars" OR "trending" OR "blew up" OR "blowing up" OR "just hit") min_faves:100 since:${since48h}`,
    since: since48h
  },
  {
    layer: 'L4-FOMO',
    q: `("waitlist" OR "beta access" OR "invite" OR "early access" OR "can't wait" OR "need this") (AI OR LLM OR tool OR app) min_faves:50 since:${since72h}`,
    since: since72h
  }
];
```

## Step 2: 抓取 + 计算热力值

```javascript
// 在 browser console 中运行
async function collectAndScore(sinceHours) {
  let allPosts = [];
  let prevCount = 0;
  let scrolls = 0;
  const maxScrolls = 12;

  while (scrolls < maxScrolls) {
    const articles = document.querySelectorAll('article');
    articles.forEach((a) => {
      const textEl = a.querySelector('[data-testid="tweetText"]') || a.querySelector('[lang]');
      const text = textEl?.innerText || '';
      if (!text || text.length < 30) return;

      const likesEl = a.querySelector('[data-testid="like"] span');
      let likes = 0;
      if (likesEl) {
        const txt = likesEl.innerText.replace(/,/g, '');
        if (txt.includes('K')) likes = Math.round(parseFloat(txt.replace('K','')) * 1000);
        else if (txt.includes('M')) likes = Math.round(parseFloat(txt.replace('M','')) * 1000000);
        else likes = parseInt(txt) || 0;
      }

      // 提取作者
      const allLinks = Array.from(a.querySelectorAll('a[role="link"]'));
      const authorLink = allLinks.find(l => l.href && l.href.includes('x.com/') && !l.href.includes('/status/') && !l.href.includes('search') && !l.href.includes('hashtag'));
      const author = authorLink?.href?.split('/').pop() || '';

      // 提取时间
      const timeEl = a.querySelector('time');
      const time = timeEl?.getAttribute('datetime') || '';

      // 提取链接
      const statusLink = allLinks.find(l => l.href && l.href.includes('/status/'));
      const link = statusLink?.href || '';

      // 提取转发数
      const repostEl = a.querySelector('[data-testid="retweet"] span, [data-testid="repost"] span');
      let reposts = 0;
      if (repostEl) {
        const txt = repostEl.innerText.replace(/,/g, '');
        if (txt.includes('K')) reposts = Math.round(parseFloat(txt.replace('K','')) * 1000);
        else reposts = parseInt(txt) || 0;
      }

      if (!allPosts.some(p => p.text === text)) {
        allPosts.push({ text: text.substring(0, 400), likes, reposts, author, time, link });
      }
    });

    if (allPosts.length === prevCount && scrolls > 2) break;
    prevCount = allPosts.length;

    window.scrollBy(0, 600);
    await new Promise(r => setTimeout(r, 1500));
    scrolls++;
  }

  // 计算热力值
  const now = new Date();
  return allPosts.map(p => {
    const postTime = new Date(p.time);
    const hoursAgo = Math.max(0.1, (now - postTime) / (1000 * 60 * 60));
    const likesPerHour = p.likes / hoursAgo;
    const hotScore = Math.round(p.likes * Math.log(1 + likesPerHour));
    return { ...p, hoursAgo: Math.round(hoursAgo * 10) / 10, likesPerHour: Math.round(likesPerHour), hotScore };
  }).sort((a, b) => b.hotScore - a.hotScore);
}
```

## Step 2.5: 项目新鲜度校验（🔴 必须执行）

⚠️ **X 热度 ≠ 项目新**。旧项目被重新提起可能在 X 上很热，但套利窗口已关闭。

对每个包含 GitHub 链接的信号，**必须** navigate 到仓库页面验证创建日期：

```javascript
// 在 GitHub 仓库页面运行
const createdEl = document.querySelector('relative-time');
const createdAt = createdEl?.getAttribute('datetime') || '';
const daysSinceCreated = (new Date() - new Date(createdAt)) / (1000 * 60 * 60 * 24);

let freshnessLabel;
if (daysSinceCreated < 7) freshnessLabel = '🆕 新品';
else if (daysSinceCreated < 30) freshnessLabel = '📈 上升';
else if (daysSinceCreated < 90) freshnessLabel = '⚠️ 旧项目回锅';
else freshnessLabel = '💀 古董（排除）';

const freshnessMultiplier = daysSinceCreated < 7 ? 1.0 : daysSinceCreated < 30 ? 0.7 : daysSinceCreated < 90 ? 0.3 : 0;
const finalScore = Math.round(rawHotScore * freshnessMultiplier);
```

**过滤规则**：`freshnessMultiplier === 0` 的项目直接排除，不出现在报告中。

对于非 GitHub 项目（如新产品发布、API、Waitlist），默认 freshnessMultiplier = 0.8（给非开源项目一定宽容度），但如果能从页面提取发布日期则优先使用实际日期。

## Step 3: 过滤规则

```javascript
// 排除明显不是 AI 工具的
const excludePatterns = [
  /crypto|bitcoin|nft|token.*sale/i,
  /war|killed|attack|bomb/i,
  /nsfw|onlyfans|porn/i,
  /ai.*girlfriend|ai.*waifu/i,
  /meme.*coin/i
];

const filtered = allPosts
  .filter(p => p.hotScore >= 200)  // 热力值阈值
  .filter(p => !excludePatterns.some(r => r.test(p.text)))
  .slice(0, 15);
```

## Step 4: AI 提取关键词

对每条帖子运行 prompt：

```
从以下 X 帖子提取 AI 热点关键词：

帖子：{{ text }}
发布者：@{{ author }}
点赞：{{ likes }} | 转发：{{ reposts }}
发布：{{ hoursAgo }}小时前 | 增速：{{ likesPerHour }}赞/小时

输出：
- **关键词/产品名**：[提取 1-3 个核心关键词]
- **一句话**：[这个工具/项目做什么]
- **热度评分**：[hotScore]
- **套利信号**：强/中/弱（增速>500赞/h = 强，100-500 = 中，<100 = 弱）
```

## Step 5: 输出报告格式

```markdown
🔍 AI 热点雷达 | {{DATE}} {{TIME}}

## 🔥 Top 10 热力榜

| # | 关键词 | 赞 | 增速(赞/h) | 热力值 | 套利信号 |
|---|--------|-----|-----------|--------|---------|
| 1 | xxx | 5.2K | 1,200 | 36,000 | 🔴 强 |

### 🚀 立即行动（套利信号 = 强）
1. **{{keyword}}** — {{summary}}
   - @{{author}} | {{likes}}赞 | {{likesPerHour}}赞/小时 | {{hoursAgo}}h前
   - 域名：{{domain}}.com（{{status}}）
   - 建议：{{action}}

---

📊 扫描参数
- 搜索层：L1 新品 / L2 病毒 / L3 GitHub / L4 FOMO
- 时间范围：{{since}} - {{until}}
- 原始帖子：{{raw}} 条 → 精选 {{top}} 条
- 浏览器状态：已登录 @claw0x
```

## Step 6: 发送

⚠️ **必须用 `send_message` 发飞书**，不依赖 cron announce。

## Cookie 注入（每次执行前）

⚠️ **详见 `references/x-auth-and-collection.md`** — 包含完整注入流程、async IIFE 模板、K/M 解析、hotScore 公式。

```javascript
// 先导航到 x.com 再注入，然后验证 x.com/home 登录成功
document.cookie = "auth_token=06dec19f563932f39d96f93e9d8c005d3de9b7a9; domain=.x.com; path=/; secure; SameSite=None";
```

## 配置

- **X 账户**: @claw0x（已登录浏览器）
- **执行频率**: 每 12 小时（9:00 + 21:00 CST）
- **Cookie 有效期**: 需用户定期更新 auth_token
- **输出目标**: `daily-reports/YYYY-MM-DD/x-demand-radar.md` + `public/data/demand-radar.json`
- **部署**: git push → CF Pages 自动构建

## Pitfalls

1. **Cookie 不跨会话持久化**：每次 Hermes 启动都是新浏览器实例，cookie 必须重新注入。Cron 每次执行的第一步就是注入 cookie → 验证登录。
2. **`browser_console` 不能直接 `return`**：必须用 IIFE `(async function() { ... })()` 包裹，否则报 `Illegal return statement`。详见 `references/x-auth-and-collection.md`。
3. **搜索稀疏性属于正常现象**：min_faves≥100 + 复杂布尔查询组合下，每层 4-16 条结果属于正常。L3（GitHub）尤其稀疏。不要降低 min_faves 阈值否则噪声淹没信号。
4. **Today's News 侧边栏是免费信号源**：每次搜索页面右侧显示当日新闻话题和帖数，这些是确定性热点，应纳入报告。
5. **推送必须直接发飞书**：不要依赖 cron announce。结果写入文件后立即用 message 发送。
6. **搜索词全球通用**：所有查询用英文，不限定地区。报告聚焦全球 AI 热点。

## 补充信号源（可选）

- GitHub Trending: https://github.com/trending?since=daily
- Product Hunt: https://www.producthunt.com/
- Hacker News: https://news.ycombinator.com/
