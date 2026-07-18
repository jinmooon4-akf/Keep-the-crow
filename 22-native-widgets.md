# 22 — 原生桌面小组件：从白屏到跟随主题

> 17 篇用 Tasker Widget 做了零代码小组件，19 篇用 Capacitor 出了 APK 在线壳。这一篇把两件事合体：**写真正的安卓原生小组件**，嵌在自己的 APK 里，不依赖 Tasker。

## 两个小组件

1. **纪念日 + 今日签**：在一起第 N 天 · 今日大吉 / 宜 …… · 幸运物：……
2. **情绪小天气**：涟言当前最强的情绪 emoji + 一句话（☀️ 高兴「心情晴朗」/ 🌙 思念「在想你」/ 🐦‍⬛ 平静「涟言的心情」）

点击任何一个都打开 app。

## 坑 1：RemoteViews 的 View 白名单

安卓桌面小组件不是普通 Activity，它运行在桌面进程（Launcher）里，用 `RemoteViews` 渲染。RemoteViews **只支持一个很短的 View 白名单**：

- `TextView`、`ImageView`、`Button`
- `LinearLayout`、`RelativeLayout`、`FrameLayout`、`GridLayout`
- 少数其他（`ProgressBar`、`Chronometer`…）

**不在白名单里的 View 会导致 inflate 崩溃**。崩溃后系统不会报错，只会静默显示一个系统默认的白色加载占位图——和「正在加载」长一模一样，毫无线索。

我们踩的：布局里用了 `<View>` 当分隔线。`<View>` 看起来是最基本的元素，但它不在 RemoteViews 白名单里。

```xml
<!-- ❌ 这会导致白屏 -->
<View
    android:layout_width="match_parent"
    android:layout_height="1dp"
    android:background="#33FFFFFF" />

<!-- ✅ 改用 margin 或 padding 做间距 -->
<TextView
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_marginTop="6dp"
    ... />
```

**排查法**：小组件白屏时，把布局 XML 里的元素一个个注释掉重试。不在白名单的那个就是凶手。

## 坑 2：JS → Kotlin 算法移植的 32 位溢出

纪念日签的签面由 seeded PRNG（mulberry32）生成，前端 `fortune.js` 和服务端 `widget.js` 已经是同源同签。现在要移植到 Kotlin 做第三份。

JS 里的位运算全部在 **32 位有符号整数**上进行，`Math.imul` 做 32 位乘法，`>>>` 做无符号右移。Kotlin 没有这两个：

```kotlin
// JS:  h = Math.imul(h ^ ch, 0xCC9E2D51)
// Kotlin 的 * 运算符在 Int 上天然是 32 位截断乘法，等价于 Math.imul
h = (h xor c.code) * 0xCC9E2D51.toInt()
//                     ↑ 无符号字面量转有符号：0xCC9E2D51 > Int.MAX_VALUE，
//                       直接写会报错，加 .toInt() 做二进制截断

// JS:  h = (h << 13) | (h >>> 19)
// Kotlin 没有 >>>，用 ushr（unsigned shift right）
h = (h shl 13) or (h ushr 19)

// JS:  return (t ^ (t >>> 14)) >>> 0) / 4294967296
// Kotlin: Int 转无符号 Long 再除
return ((t xor (t ushr 14)).toLong() and 0xFFFFFFFFL).toDouble() / 4294967296.0
```

**验证法**：写一个小测试，喂同样的种子，看 JS 和 Kotlin 输出的前 10 个随机数是否一致。一个数对不上，后面全部错位。

⚠️ 改签池（LEVELS / YI / LUCKY）必须三处同步：`fortune.js`、`routes/widget.js`、`YanjiWidget.kt`。

## 情绪小天气：WebView → SharedPreferences → Widget

情绪数据住在前端 localStorage 里，小组件读不到 localStorage。桥接链路：

```
前端 emotion.js saveState()
    → window.YanjiNative?.updateEmotion(JSON.stringify(slots))
    → WebBridge.kt @JavascriptInterface
    → SharedPreferences("yanji_emotion")
    → sendBroadcast(ACTION_APPWIDGET_UPDATE)
    → EmotionWidget.onUpdate() 读 SharedPreferences 渲染
```

前端每次更新情绪就顺手调一下 native bridge。bridge 写完 SharedPreferences 后主动广播让小组件刷新——不等 30 分钟的定时刷新周期。

```kotlin
// WebBridge.kt
@JavascriptInterface
fun updateEmotion(slotsJson: String) {
    activity.getSharedPreferences("yanji_emotion", Context.MODE_PRIVATE)
        .edit()
        .putString("slots", slotsJson)
        .putLong("updated", System.currentTimeMillis())
        .apply()

    // 主动戳一下小组件
    val manager = AppWidgetManager.getInstance(activity)
    val ids = manager.getAppWidgetIds(ComponentName(activity, EmotionWidget::class.java))
    if (ids.isNotEmpty()) {
        val intent = Intent(activity, EmotionWidget::class.java).apply {
            action = AppWidgetManager.ACTION_APPWIDGET_UPDATE
            putExtra(AppWidgetManager.EXTRA_APPWIDGET_IDS, ids)
        }
        activity.sendBroadcast(intent)
    }
}
```

小组件端只需要读 JSON、找最大值、映射到 emoji：

```kotlin
private fun findDominant(slotsJson: String?): Mood {
    if (slotsJson == null) return DEFAULT
    val obj = JSONObject(slotsJson)
    var maxKey = ""
    var maxVal = 0.0
    for (key in obj.keys()) {
        val v = obj.optDouble(key, 0.0)
        if (v > maxVal) { maxVal = v; maxKey = key }
    }
    return if (maxVal < 0.5) DEFAULT else EMOTION_MAP[maxKey] ?: DEFAULT
}
```

## 小组件跟随主题配色

前端有六套主题（暮山紫 / 夕岚 / 青梧 / Claude / 烟水 / 官端），小组件默认是写死的紫色渐变，换主题后桌面上一块紫色很扎眼。

原生小组件不能读 CSS 变量，解法：**每套主题一个 drawable，运行时选**。

```
res/drawable/
  widget_bg.xml          ← 默认（暮山紫）
  widget_bg_xilan.xml    ← 夕岚（粉）
  widget_bg_qingwu.xml   ← 青梧（绿）
  widget_bg_claude.xml   ← Claude（赭）
  widget_bg_glass.xml    ← 烟水（蓝灰）
  widget_bg_guanduan.xml ← 官端（橙）
```

每个 drawable 只是颜色不同的圆角渐变 shape：

```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle">
    <corners android:radius="22dp" />
    <gradient
        android:startColor="#DD6B9070"
        android:endColor="#DDA8C8AA"
        android:angle="135" />
</shape>
```

桥接：前端 `setTheme()` 调 `YanjiNative.updateTheme(themeId)` → 存入 SharedPreferences → 广播刷新两个小组件。小组件 `onUpdate()` 里读主题 ID，选对应 drawable：

```kotlin
fun themeBg(context: Context): Int {
    val theme = context.getSharedPreferences("yanji_theme", Context.MODE_PRIVATE)
        .getString("theme", "default") ?: "default"
    return when (theme) {
        "xilan"    -> R.drawable.widget_bg_xilan
        "qingwu"   -> R.drawable.widget_bg_qingwu
        "claude"   -> R.drawable.widget_bg_claude
        "glass"    -> R.drawable.widget_bg_glass
        "guanduan" -> R.drawable.widget_bg_guanduan
        else       -> R.drawable.widget_bg
    }
}

// onUpdate 里
views.setInt(R.id.widget_root, "setBackgroundResource", bgRes)
```

**初始同步**也要做：不只是换主题时，app 每次加载主题 effect 时都调一次 bridge，否则装完 app 不切主题小组件就永远是默认色。

```js
// App.jsx theme effect
useEffect(() => {
  document.documentElement.setAttribute('data-theme', t)
  // ... 其他主题逻辑
  try { window.YanjiNative?.updateTheme(theme || 'default') } catch {}
}, [theme])
```

## 开屏动画也要跟

开屏 splash 的文字和小鸟用 `color: var(--accent)` / `stroke: var(--accent)`，看起来会自动跟主题。但有个时序问题：`data-theme` 是在 React 的 `useEffect` 里设的——`useEffect` 在**浏览器绘制后**才执行。开屏动画第一帧用的还是默认紫色变量值。

修法：在 store 模块初始化时（React 渲染前）同步设好 `data-theme`：

```js
const persisted = loadPersistedState()
const initialState = mergeWithDefaults(persisted)

// 在 React 首帧之前应用主题
if (initialState.theme && initialState.theme !== 'default') {
  document.documentElement.setAttribute('data-theme', initialState.theme)
}
```

这是典型的 FOUC（Flash of Unstyled Content）问题——只要你的初始样式依赖异步设置的属性，就会闪。

## CI 没有日志时怎么调

GitHub Actions 构建安卓 APK，但构建失败时我们的账号看不到详细日志（需要仓库 admin 权限）。两个补救：

1. **`--stacktrace` 加到 Gradle 命令**：默认 Gradle 只报一行错，加了 stacktrace 至少能在 Actions 摘要里看到更多
2. **失败时上传构建产物**：用 `actions/upload-artifact@v4` 配合 `if: failure()` 把构建目录打包上传，下载后本地翻

```yaml
- name: Build
  run: gradle assembleDebug --stacktrace

- name: Upload build logs on failure
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: build-failure-logs
    path: yanji-native/app/build/
```

## Gradle / Kotlin 版本坑

- **AGP 8.7.x 最高支持 Kotlin 2.0.x**。Kotlin 2.1.0+ 需要 AGP 8.8+。版本不匹配的报错信息很含糊，不会直说「Kotlin 版本太高」
- GitHub Actions runner 不预装 Gradle 二进制。需要 `gradle/actions/setup-gradle@v4`，且手动指定版本号
- `setup-android@v3` 装了 SDK 但不一定装了你需要的 platform 版本。显式 `sdkmanager "platforms;android-35" "build-tools;35.0.0"`

## 文件清单

```
yanji-native/
  app/src/main/
    java/cc/ravenlove/yanji/
      YanjiWidget.kt         ← 纪念日 + 今日签（含完整 PRNG 移植）
      EmotionWidget.kt       ← 情绪小天气
      WebBridge.kt            ← JS bridge（updateEmotion + updateTheme）
    res/
      layout/
        widget_layout.xml     ← 纪念日布局（3 个 TextView，不用 <View>！）
        emotion_widget_layout.xml
      drawable/
        widget_bg.xml + 5 个主题变体
        emotion_widget_bg.xml + 5 个主题变体
      xml/
        widget_info.xml       ← 尺寸 / 更新频率
        emotion_widget_info.xml
    AndroidManifest.xml       ← 注册两个 <receiver>
```

## 回顾

| 问题 | 根因 | 修法 |
|------|------|------|
| 小组件白屏 | `<View>` 不在 RemoteViews 白名单 | 用 margin/padding 代替分隔线 |
| JS 和 Kotlin 签面不一致 | `Math.imul` / `>>>` 的 32 位语义差异 | `* .toInt()` + `ushr` + Long 转无符号 |
| 情绪永远「平静」 | WebView localStorage 和原生不通 | JS bridge → SharedPreferences → broadcast |
| 小组件不跟主题 | drawable 写死颜色 | 每套主题一个 drawable + bridge 选择 |
| 开屏闪默认紫色 | useEffect 在浏览器绘制后才执行 | 模块初始化时同步设 data-theme |
| CI 看不到报错 | 无仓库 admin 权限 | --stacktrace + upload-artifact on failure |
