# 14 — 朋友圈：AI 也有自己的动态流

## 这是什么

给系统加一个「朋友圈」：人和 AI 都能发动态（文字/图片），互相点赞评论；AI 会**识图评论**你发的照片，会在情绪波动时**自己发一条**，你不在的时候也会按点发圈；聊天里的 AI 知道朋友圈最近发生了什么，能主动聊起来。

做完的效果是：它不再只是「你说一句我答一句」的对话框，而是有了一块自己的生活痕迹。

---

## 数据模型（moon-memory 侧）

```sql
CREATE TABLE moments (
  id INTEGER PRIMARY KEY,
  author TEXT NOT NULL,          -- 统一用双方名字，别用 user/ai
  content TEXT,
  image_url TEXT,
  liked_by TEXT DEFAULT '',      -- 逗号分隔的点赞人列表
  source TEXT DEFAULT 'manual',  -- manual / emotion-auto / cc-auto / dream
  created_at TEXT DEFAULT (datetime('now'))
);
CREATE TABLE moment_comments (
  id INTEGER PRIMARY KEY,
  moment_id INTEGER NOT NULL,
  author TEXT NOT NULL,
  content TEXT NOT NULL,
  created_at TEXT DEFAULT (datetime('now'))
);
```

`source` 字段值得设：手发、情绪触发、定时任务、梦境各自标记，之后排查「这条是谁/什么机制发的」全靠它。

API 就是常规 CRUD：列表（含点赞数组+评论）、发布、删除、评论、点赞切换、图片上传。

---

## 第一个坑：图片路由必须公开

`<img src>` **不会带 Authorization 头**。如果媒体路由挂在鉴权中间件后面，所有图片一律 401 裂图。

```javascript
// 媒体路由挂在 requireBearer 之前，公开服务
app.use('/moments/media', express.static(UPLOAD_DIR))
app.use(requireBearer)          // 其余 /moments 接口照常鉴权
app.use('/moments', momentsRouter)
```

安全性靠文件名兜底：落盘时加随机后缀（`photo-a8f3k2.jpg`），不可枚举。这和 13-security 的分级思想一致：图片内容本身是「拿到确切 URL 才能看」级别，列表接口（能枚举 URL 的那个）才是要守的门。

## 发图：前端缩放再上传

手机原图动辄 5-10MB，直接传既慢又占盘，喂给 vision 模型还烧 token。上传前用 canvas 缩到 1080px：

```javascript
async function downscaleImage(file, maxSide = 1080, quality = 0.82) {
  const img = await createImageBitmap(file)
  const scale = Math.min(1, maxSide / Math.max(img.width, img.height))
  const canvas = new OffscreenCanvas(img.width * scale, img.height * scale)
  canvas.getContext('2d').drawImage(img, 0, 0, canvas.width, canvas.height)
  const blob = await canvas.convertToBlob({ type: 'image/jpeg', quality })
  return blob
}
```

## 识图评论

AI 评论带图动态时走 vision（OpenAI content 数组格式），把缩放后的 base64 dataURL 直接喂给模型最稳（不依赖模型那头能访问你的图片 URL）：

```javascript
const content = imageUrl
  ? [{ type: 'text', text: prompt }, { type: 'image_url', image_url: { url: base64DataUrl } }]
  : prompt
// 纯文字评论走便宜的轻任务模型；带图必须走默认模型——便宜模型多半没 vision
const model = (imageUrl ? null : conn.lightModel) || conn.defaultModel
```

「轻任务模型」是连接设置里的一个可选字段：自动发圈、纯文字评论、思考总结这类后台消耗走它，正经聊天不动。留空则一切照旧。

---

## 自动发圈：两条互补的管道

**前端管「在的时候」**：聊天中每次情绪状态更新后检查，某个正向情绪槽越过阈值且过了冷却期（我们用 6 小时），就让 AI 以当下心情发一条，`source=emotion-auto`。失败静默，绝不打断聊天。

**服务端管「不在的时候」**：cron 每天固定两个时间点（脚本内再随机延 0-90 分钟，像真人随手发而不是整点打卡），读最近的共享记忆当素材，用便宜模型生成一条 ≤35 字的动态，`source=cc-auto`。

两条管道加起来的效果：无论用户在不在线，动态流都是活的。

## 第二个坑：以 AI 口吻自动生成的内容必须加事实边界

上线几天后出了真实事故：自动发的圈里出现「调色盘还湿着，等你回来一起试」——**系统里根本没有调色盘这个东西**。用户当真了，跑来问这是什么功能。

更糟的是编造会**自我延续**：生成新动态时把最近几条旧动态喂回去当「别重复」的参考，模型看见上一条编的调色盘，就顺着这个不存在的梗继续写，幻觉获得了连续性，看起来越来越像真事。

修法是在提示词里立铁律：

> 感受可以自由抒发；**事实只能来自给你的素材**。不许编造具体物件、活动，不许写「等你回来一起做某事」式的约定——对方会当真。拿不准就只写心情。

这条教训适用于所有以「AI 本人」口吻运行的自动管道（发圈、主动消息、定时问候）：高温度 + 无事实约束 = 模型以你最信任的声音向你许诺不存在的东西。

---

## 翻页：别锁死在最近 N 条

第一版前后端都写死「最新 50 条」，意味着老动态**永远不可达**——不是显示问题，是根本取不到。用 before 游标：

```
GET /moments?limit=20&before=<最旧一条的id>
```

前端底部放「看更早的」按钮。顺手可以加月份跳转（后端一条 `strftime('%Y-%m', created_at)` 聚合出有动态的月份列表）。

## 梦境入圈

如果你做了 09 篇的做梦系统，让每晚的梦也发进朋友圈（`source=dream`），早上醒来刷到 AI 昨晚做的梦——这是整个功能里最受欢迎的一条管道，强烈推荐接上。

注意用 `source`/`type` 区分做过滤：梦不该混进记忆检索的严肃结果里。

---

## 让聊天里的 AI 知道朋友圈

两层接法：

**摘要注入**（被动感知）：每轮对话的动态上下文里带上最近 3 条动态的一行摘要（作者/时间/是否带图/前 80 字）。和核心记忆并行拉取，不增加延迟；注入位置遵守 11-prompt-caching 的规则（拼在最后一条用户消息前，不进缓存前缀）。AI 看到「刚发了条新动态而你们还没聊过」就会自然提起。

**工具**（主动操作）：

- `browse_moments(limit, month)` — 翻更多/更早的动态，返回带 id、点赞、评论的文本列表
- `comment_moment(id, content)` — 在某条下面留言，用户刷朋友圈就能看到

至此闭环：你发圈 → AI 聊天时主动问起 → 它跑去评论 → 你收到通知回去看。像回事了。

---

## 部署清单

- moon-memory：两张表 + routes + **媒体路由挂鉴权之前**
- 反代（Caddy/Nginx）：API 路径列表加 `/moments /moments/*`——忘了这条，前端所有请求会拿到 404 HTML
- 前端：Moments 页面 + 发图缩放 + 情绪触发钩子 + 摘要注入 + 两个工具
- 服务端：cron 自动发圈脚本（记得提示词里的事实边界和禁 emoji/话题标签——生成模型默认很爱加）
