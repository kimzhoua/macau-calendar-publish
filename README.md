# 澳门节假日 App · 源码工程说明（v1.8.2）

> 给「下次继续开发」用的完整交接文档。先读这份，再看代码。

## 1. 这是什么

一个只显示**澳门公众假期**的月历 App，纯 Android framework（无 AndroidX / Material / Kotlin），
单 Activity、单 Java 文件，UI 全部代码搭建。产物是一个可直接安装的 debug 签名 APK。

- 包名：`com.example.holidaycalendar`
- 当前版本：versionCode **43** / versionName **1.9.6**
- minSdk **24**，targetSdk **34**（compileSdk 34）
- 桌面显示名称：**澳门节假日**（`res/values/strings.xml` 的 `app_name`）
- 允许竖屏，沉浸状态栏（透明 + 浅色图标）

## 2. 目录结构

```
apkbuild/
├── AndroidManifest.xml                     # 包名/版本/权限/主题，改版本号在这里
├── build.sh                                # 一键构建（详见第 3 节）
├── debug.keystore                          # 固定签名证书，千万别删（删了签名变，用户必须卸载重装）
├── README.md                               # 本文档
├── res/
│   ├── values/strings.xml                  # app_name（桌面名称）
│   ├── raw/holidays.txt                    # 内置离线数据 111 条，格式见 5.3
│   └── mipmap-xxxhdpi/ic_launcher.png      # 192px 图标（蓝底白日历 + 红顶条）
└── src/com/example/holidaycalendar/
    └── MainActivity.java                   # 全部逻辑（约 900 行）
```

## 3. 构建（重要：不能用 Gradle）

本机（云主机）**没有可用的 Android Studio / Gradle / sdkmanager**，走手工链路：

```bash
cd apkbuild
./build.sh          # 输出 build/app-debug.apk
```

build.sh 内部依次做：`aapt2 compile` → `aapt2 link`（生成 R.java）→ `javac -source 8` → `d8` →
zip 塞入 classes.dex → `zipalign` → `apksigner sign`（用根目录 `debug.keystore`）。

脚本里硬编码的路径（换机器需要改）：

| 用途 | 路径 |
|---|---|
| JDK | `/home/user/toolchain/jdk-17.0.10+7` |
| Android SDK | `/home/user/toolchain/android-sdk` |
| android.jar | `.../platforms/android-34/android.jar` |
| build-tools | `.../build-tools/34.0.0` |

**踩过的坑（不要重试）**：
- `sdkmanager` 在这个环境报代理解析错误，永远失败 → 用 curl 下官方 zip 手动解压。
- Gradle + AGP 在本环境解析不了 Google Maven 插件（TLS 被挡）→ 已弃用，不要再试图引入 Gradle 工程。
- 因为不能用 AndroidX，**不要**写 `AppCompatActivity`、`RecyclerView`、`ViewBinding`、Material 控件。

## 4. MainActivity 代码地图

按顺序阅读最省力：

| 区块 | 关键成员 | 说明 |
|---|---|---|
| 常量 | `ICS_URL` / `PAGE_URL` / `PREF*`(theme_id / font_scale / theme_migrated) / `THEME_*` / `KIND_*` / `MAX_BROWSE_YEAR`(=2099) / `FETCH_AHEAD`(=2) | 数据源与主题常量；跳转上限 2099；联网抓取只抓 今年-3 … 今年+FETCH_AHEAD，避免刷一堆 404 页面 |
| 内部类 | `Entry(name, kind, tag)` | 一天的数据：名称、类型(0 假期/1 半天)、标记(""/"补假"/"下午") |
| 状态 | `shown`(当前显示月) / `dataCache` / `themeId` / `lastFetchAt`(静态,10 分钟节流) | |
| `onCreate` | 沉浸状态栏 → 读主题 → `applyPalette()` → `buildUi()` → `loadLocalData()` → `refresh()` → 首次联网 | |
| `applyPalette` | 16 个颜色字段 `colorBg/colorTitle/...` | **所有颜色都从这里取，不要硬编码颜色到视图里** |
| 应用更新 | `checkUpdate(manual)` / `showUpdateDialog` / `downloadApk` / `installApk` / `downloadReceiver` / `DEFAULT_UPDATE_URL` / `PREF_UPDATE_URL` | 读 update.json 比对 versionCode，高则弹窗；用系统 DownloadManager 下到 `Downloads/`，`ACTION_DOWNLOAD_COMPLETE` 后拿 `content://` 交给系统安装器（不需要 FileProvider）。Android 8+ 需 `REQUEST_INSTALL_PACKAGES` 且用户允许未知来源 |
| 字体设置 | `fontScale`(0.8–1.6) / `applyTypography(view)` / `rowHeight()` | `⚙` 面板一个 SeekBar 连续调字号。**字重功能已按用户要求移除**（v1.8.2），不要再加回来 |
| 玻璃工具 | `glassCard(radius, fill)` / `todayGradient()` / `BackdropView` | 半透明卡片 + 今天胶囊 + 背景光斑（自绘 LinearGradient + 3 个 RadialGradient 光斑，无真实高斯模糊） |
| UI 工具 | `dp()` / `statusBarHeight()` / `rounded(color, radius)` / `bottomRoundedGradient()` | 圆角与渐变靠 GradientDrawable |
| `buildUi` | 头部(标题+导航+星期栏) → `grid` → `sectionTitle("本月公休日")` → `scroll`+`monthList` → `statusBar` | 两套主题在同一函数里用 `if (elegant)` 分支，结构一致只改样式 |
| `icon(text)` | 导航用的 `‹ ›`、`⚙` 按钮 | 34dp 触控区（玻璃主题 36dp 胶囊） |
| 翻页 | `pager`(SnapPager, 无惯性) / `pages` / `buildMonthPage(cal)` / `scheduleSettle()` / `snapToNearestPage()` / `commitPage()` / `rotatePages(dir)` | 三页横排（上一月/本月/下一月），各页宽=屏幕宽-左右边距；手指拖动天然跟手；**松手瞬间（ACTION_UP/CANCEL 后 16ms）立即**调用 `snapToNearestPage()`，按拖动方向判定翻页（位移 >8dp 即翻，不要求过半屏），用 200ms ValueAnimator 吸附整页（跨页则 `commitPage` 换月；`SnapPager.fling()` 被置空以去掉惯性，所以松手即定。换月**不重建三页**，而是 `rotatePages` 丢掉一侧、另一侧补一页并把 scrollX 归中，像素级连续所以不闪。全量重建（buildGrid）路径则在 `OnPreDraw` 里兜底定位，防止重建后先画出左页。注意两点坑：① 松手在正中间时 Math.round 可能判成本月，必须仍然执行吸附动画，否则会卡在半页；② `pager.isPressed()` 为真说明手指还没松，要延后再吸附。`pageWidthPx` 在 buildGrid 里按屏幕宽计算，pager 高度固定为 `rowHeight()*6+8dp` |
| `showSettingsDialog` | `⚙` 设置面板：主题三选胶囊（`styleThemeChip`）+ 字号/字重两个 SeekBar + 刷新数据按钮，确定后统一写入 Prefs 并 `applyPalette()+buildUi()+refresh()` | 主题为“待定值”，点确定才生效 |
| `showJumpDialog` | 年月双 NumberPicker（点月份标题触发，带「本月」中性按钮） | 年份下限=数据最早年，上限=max(2099, 数据最新年+1, 今年+2) |
| `loadLocalData` | 缓存文件 → 内置 raw，写底部状态文字 | |
| `parseText` / `serialize` | 自研行格式 `日期\|名称\|类型\|标记` | |
| `parseIcal` | 抓官方 iCal | 兼容 `DTSTART;VALUE=DATE:` 与 `DTSTART:...T063000Z`，取数字前 8 位；SUMMARY 括号后缀判类型 |
| `parseYearPage` | 抓 `year-YYYY/` 页面 | 正则取 3 张表：0=公众假期、1=豁免上班(半天)、2=补假 |
| `fetchOnline` | 后台线程：年度页面(curYear-3 … curYear+FETCH_AHEAD) → 再抓 iCal 覆盖合并 → 写缓存+meta → 回主线程刷新 | 全失败则保留旧数据不清空；未来年份页面 404 时自动跳过 |
| `refresh` | `buildGrid()` + `buildMonthList()` | 任何状态变化都走这里 |
| `buildGrid` | 组装三页并回中；单页由 `buildMonthPage` 生成 6×7 网格，周一开头（`offset = (dow+5)%7`），行高 `rowHeight()` | 颜色优先级：非本月置灰 > 半天橙 > 假期红 > 周末红 > 常规 |
| `buildMonthList` | 本月假期可滚动清单（日期/星期/名称，雅致主题带分隔线） | |

注意：**中间那条“选中日期详情”提示栏已在 v1.6.3 删除**，`detailBar`/`renderDetail` 已不存在，
点日期只用于跨月跳转。不要再加回来，除非用户明确要求。

## 5. 数据

### 5.1 来源
1. 官方 iCal `https://www.gov.mo/zh-hans/public-holidays/ical/`（只有当年+次年）
2. 年度页面 `https://www.gov.mo/zh-hans/public-holidays/year-YYYY/`（历史年份靠它）
3. 内置 `res/raw/holidays.txt`（111 条：2024:28 / 2025:27 / 2026:28 / 2027:28）

### 5.2 运行时文件
- 缓存：`getFilesDir()/holidays_cache.txt`（同 raw 的行格式）
- 元信息：`getFilesDir()/holidays_meta.txt`（显示在底部状态栏，如“官方在线 11-13 08:06 · 2024–2027 共 111 条”）
- 首选项：`getSharedPreferences("prefs")`，键 `theme_id`（0 简约白 / 1 雅致渐变）

### 5.3 行格式
```
2026-12-08|圣母无原罪瞻礼|0|
2027-12-31|除夕|1|下午
```
`日期|名称|类型(0假期/1半天)|标记(""/"补假"/"下午")`，末尾仍保留一个 `|`。

### ### 5.4 待补
2028 年起的澳门假期表政府尚未公布（官方页面 404）。跳转弹窗现在允许翻到 2099，
这些年份会显示“本月无公休数据（官方未公布，可点右上 ⟳ 联网更新）”；
一旦官方发布，点 ⟳ 即自动抓取，无需再改代码或升级安装包。

## 6. 两套主题

| 项 | 主题0 简约白 | 主题1 简约蓝 | 主题2 渐变玻璃（默认） |
|---|---|---|
| 背景 | 纯白 | `#F3F6FB` |
| 头部 | 透明 | TL_BR 渐变 `#1B62C9 → #4AA3F0`，底角 24dp |
| 标题/箭头 | 深色 | 白色 |
| 日历 | 直接铺开 | 白色圆角卡（18dp + elevation 6，边距 12dp） |
| 今天 | `#E3F2FD` 方块 | `#D9E9FF` 圆角胶囊（12dp） |
| 列表 | 纯文字 | 白色圆角卡，行间 `#EDF1F7` 分隔线 |
| 假期/半天/周末 | `#D32F2F` / `#EF6C00` / `#D32F2F` | `#E53935` / `#F57C00` / `#D94A4A` | `#E0455E` / `#E07A2F` / `#D5586B` |

默认主题：v1.7.3 起为**主题2 渐变玻璃**（`theme_migrated` 标记做一次性迁移，把老用户的主题也切过去）。仍然**不做深色模式**。主题0「简约白」保持最初的朴素白样式，改动视觉时优先动主题1/2。
导航现在只有：`‹` 月标题 `⚙` `›`。主题切换与数据刷新已合并进 `⚙` 设置面板（v1.6.8 起不再占用导航栏）。

## 7. 发版流程（每轮改完照做）

1. 改 `MainActivity.java` / 资源
2. `AndroidManifest.xml` 里 `versionCode` +1、`versionName` 升级
3. `./build.sh`
4. 校验：`.../build-tools/34.0.0/aapt2 dump badging build/app-debug.apk | grep -E "package|application-label"`
5. 复制成中文名放项目根目录：`cp build/app-debug.apk ../澳门节假日-vX.Y.Z.apk`
6. 用 `present_files` 交付（本环境**可以**直接交付 .apk 本地文件，不要再声称平台不支持）
7. 首次换签名时必须提醒“先卸载旧版”；自 v1.6.3 起证书已固定为根目录 `debug.keystore`，后续可覆盖安装

签名信息：`debug.keystore`，storepass/keypass = `android`，alias `androiddebugkey`，
SHA-1 `9e30e8270413df35e06e4ba7b2362e763024b80e`。

## 8. 版本历史

| 版本 | 内容 |
|---|---|
| 1.0.0 | 四地区月历（内地/香港/澳门/台湾） |
| 1.1.0 | 只留澳门；周一起始；补全 2026 假期并标红 |
| 1.2.0 | 沉浸状态栏；周末标红；行高收紧；底部本月清单 |
| 1.3.0 | 在线更新（官方 iCal）+ 缓存 + ⟳ 按钮 |
| 1.4.0 | 顶部标题「澳门特别行政区」；星期栏加大加粗；年月 NumberPicker 跳转 |
| 1.5.0 | 年度页面抓取，补齐 2024/2025/2026/2027；内置 111 条 |
| 1.6.0 | 双主题（简约白 / 雅致渐变）+ ◐ 切换 + 10 分钟联网节流 |
| 1.6.1 | 应用名改为「澳门节假日」 |
| 1.6.2 | 误删底部清单（已废弃，勿参考） |
| 1.6.3 | 删除中间“选中日期详情”提示栏，恢复底部本月公休日清单；签名证书固定 |
| 1.6.4 | 跳转年份范围开放到 2030（之后自动顺延） |
| 1.6.5 | 跳转年份上限提高到 2099；抓取范围收敛为 今年-3 … 今年+2，避免无效请求 |
| 1.9.6 | 空月份提示区分「本月无公众假期」（整年有数据）与「官方未公布」（整年无数据）；自动联网补数据只在整年缺数据时触发 |
| 1.9.5 | 农历改用自研 `LunarUtil`（1900–2099 数据表，逐日对账 lunardate 零差异）：修复 `android.icu.ChineseCalendar` 在 2027 年春节算成 2/7、导致 2027-02-11 显示「初五」的问题 |
| 1.9.4 | 「公休日列表显示分隔线」默认改为打开 |
| 1.9.3 | 修复补假日（tag=补假）在日历格和列表里不显示「补假」字样的问题 |
| 1.9.2 | 选中的日期加蓝色描边高亮；点其他日期高亮跟着走，滑动换月后高亮自动取消 |
| 1.9.1 | 点日期改为在底部列表只显示该日详情（非公休日给「标记 AL」按钮，底部可返回整月） |
| 1.9.0 | AL 角标改到右上角，与公休日一致（当天已是公休日则不重复画） |
| 1.8.9 | 点日期可标记「AL」：弹窗选 取消/AL，标记后日期变公休日颜色、右上角加小「休」角标（与公休日一致）（`al_dates` 持久化） |
| 1.8.8 | 图标换成 2048 干净版 3D 日历＋大三巴 |
| 1.9.6 | 空月份提示区分「本月无公众假期」（整年有数据）与「官方未公布」（整年无数据）；自动联网补数据只在整年缺数据时触发 |
| 1.9.5 | 农历改用自研 `LunarUtil`（1900–2099 数据表，逐日对账 lunardate 零差异）：修复 `android.icu.ChineseCalendar` 在 2027 年春节算成 2/7、导致 2027-02-11 显示「初五」的问题 |
| 1.9.4 | 「公休日列表显示分隔线」默认改为打开 |
| 1.9.3 | 修复补假日（tag=补假）在日历格和列表里不显示「补假」字样的问题 |
| 1.9.2 | 选中的日期加蓝色描边高亮；点其他日期高亮跟着走，滑动换月后高亮自动取消 |
| 1.9.1 | 点日期改为在底部列表只显示该日详情（非公休日给「标记 AL」按钮，底部可返回整月） |
| 1.9.0 | AL 角标改到右上角，与公休日一致（当天已是公休日则不重复画） |
| 1.8.9 | 点日期可标记「AL」：弹窗选 取消/AL，标记后日期变公休日颜色、右上角加小「休」角标（与公休日一致）（`al_dates` 持久化） |
| 1.8.8 | 图标换成大三巴版 3D 日历（2048 原图，四套密度） |
| 1.8.7 | 图标换回莲花版 3D 日历（1536 原图无裁切，四套密度） |
| 1.8.6 | 图标换成用户提供的 3D 日历＋大三巴（居中裁方后导出四套密度） |
| 1.8.5 | 换 App 图标：蓝青渐变「日历卡片＋莲花」，提供 hdpi/xhdpi/xxhdpi/xxxhdpi 四套尺寸 |
| 1.8.4 | 日期下方显示农历（`android.icu.util.ChineseCalendar` 计算，节假日优先显示节假日名） |
| 1.8.3 | 「日期显示方框」默认改为关；新增「公休日列表显示分隔线」开关（默认关） |
| 1.8.2 | 设置新增「日期显示方框」开关（默认开） |
| 1.8.1 | OTA 增加安装包 SHA-256 校验（清单填 sha256 时）；发布脚本 publish/publish-to-github.sh 支持一键传 GitHub |
| 1.8.0 | 「日历添加白底显示」开关改名、默认打开 |
| 1.7.9 | 设置新增白底开关 |
| 1.7.8 | 去掉玻璃主题的白底卡片 |
| 1.7.7 | 公休日右上角新增「休」角标 |
| 1.7.6 | 更新检查支持 GitHub raw（含跳转跟随、owner/repo 简写）；随包附 GitHub 托管教程 |
| 1.7.5 | 新增 App 版本在线更新（检查清单 → 下载 → 拉起系统安装）；设置里可长按修改更新源地址 |
| 1.7.4 | 滑动改为松手即翻页（按方向判定，去掉 150ms 等待与惯性）；翻到无数据月份自动联网补一次 |
| 1.7.3 | 移除字重调节；主题更名「简约蓝」「渐变玻璃」；渐变玻璃设为默认（一次性迁移）；修复设置面板底部“数据来源：”重复前缀 |
| 1.7.2 | 修复翻页闪烁：改为三页轮换 + 预绘制兜底定位 |
| 1.7.1 | 修复半页卡死：松手后必吸附到最近整页（200ms 动画），按住期间不吸附 |
| 1.7.0 | 滑动翻月改为跟手三页翻页容器（相邻月份实时滑入）；第三主题更名「类ColorOS17」 |
| 1.6.9 | 日历支持左右滑动翻月（初版为 fling 后淡入，已替换） |
| 1.6.8 | 主题切换与刷新数据移入 `⚙` 设置面板，导航栏精简为 `‹ 月份 ⚙ ›` |
| 1.6.7 | 日历行距收紧（42/46/48dp）；新增 `⚙` 显示设置，字号与字重无极调节并记忆 |
| 1.6.6 | 新增第三主题「ColorOS 玻璃」：渐变光斑背景 + 半透明玻璃卡片 + 蓝紫胶囊今天 + 胶囊导航按钮，主题弹窗改为三选 |

## 8.1 应用更新（OTA）清单格式

`update.json`（UTF-8）：

```json
{
  "versionCode": 23,
  "versionName": "1.7.6",
  "apkUrl": "https://.../app.apk",
  "sha256": "安装包的 SHA-256（可选，填了就会校验）",
  "notes": "更新说明，可用 \n 换行"
}
```

- 内置地址：`DEFAULT_UPDATE_URL`（v1.7.6 为 `https://aka.doubaocdn.com/s/PrR5adtu8g`）
- 支持 `owner/repo` 简写与 GitHub blob 链接，`normalizeUpdateUrl` 会自动转成 raw 地址（详见 `publish/README-GitHub托管教程.md`）
- `httpGet` 已改为手动跟随 301/302/303/307/308（GitHub raw 会跳转）
- 运行时可改：设置面板长按「检查应用更新」，写入 `prefs/update_url`
- 检查频率：启动后每天最多一次自动检查；手动点按钮随时检查
- **维护提醒**：本环境的下载短链每次上传都会变（内容不同 → 新地址），所以每发一版都要把新的 update.json 地址写进 `DEFAULT_UPDATE_URL`；若要老版本也能自动发现新版本，需要固定托管（如 GitHub 仓库 raw 链接），把该固定地址写进 `DEFAULT_UPDATE_URL` 即可

## 9. 常见改法速查

- **改应用名**：`res/values/strings.xml` 的 `app_name`
- **改顶部大标题**：`buildUi` 里 `appTitle.setText(...)`
- **改假期颜色**：`applyPalette` 里 `colorHoliday/colorHalf/colorWeekend`
- **调整行高/间距**：`buildGrid` 中 `dp(elegant ? 50 : 46)` 与 `cellLp.setMargins`
- **改导航按钮**：`buildUi` 中 `icon("...")` 与点击事件
- **新增一个数据年份**：不用改代码，联网后自动抓 `curYear-3 … curYear+1`；若要写进内置数据，往 `holidays.txt` 追加行即可
- **新增第三套主题**：扩展 `THEME_*` 常量 + `applyPalette` 分支 + `buildUi` 的 `elegant` 分支（建议改成 `int style` 判断，避免多层 if）
