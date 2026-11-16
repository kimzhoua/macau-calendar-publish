# 澳门节假日（Android）

澳门公众假期月历 App —— 纯 Java、手工命令行构建（aapt2 → javac → d8 → zipalign → apksigner），固定 debug 签名。

## 功能
- 月历展示，周一起始，左右滑动翻月（松手即翻、实时跟随）
- 澳门公休日 / 半天假标红，公休日右上角小「休」角标
- 日期下方显示农历（自研 `LunarUtil`，1900–2099 数据表，不依赖 ICU）
- 点日期查看当天详情；非公休日可标记「AL」，标记后变红 + 右上角「休」角标
- 三套主题：简约白 / 简约蓝 / 渐变玻璃（默认玻璃）
- 设置：字号无极调节、白底显示、日期方框、列表分隔线、刷新数据、检查更新
- App 内 OTA 更新：读取 `dist/update.json` → 比对 versionCode → 下载 → 校验 sha256 → 拉起安装器

## 目录
```
apkbuild/                 Android 工程（源码 + 资源 + 构建脚本）
  src/com/example/holidaycalendar/MainActivity.java   主界面与全部逻辑
  src/com/example/holidaycalendar/LunarUtil.java      农历换算
  res/raw/holidays.txt     离线假期数据（日期|名称|类型|标记）
  build.sh                 手工构建脚本
  debug.keystore           固定调试签名
  publish/                 发布相关（update.json 模板、GitHub 托管说明、发布脚本）
dist/                      OTA 分发目录（app-latest.apk + update.json）
```

## 构建
```bash
cd apkbuild && ./build.sh
```
产物 `apkbuild/build/app-debug.apk`。注意：`build.sh` 会过滤 javac 的 "bootstrap class path" 提示，编译错误需自己 `grep -c "error:"` 日志确认。

## OTA
`dist/update.json` 字段：`versionCode` / `versionName` / `apkUrl`（jsDelivr CDN）/ `rawApkUrl`（备用直链）/ `sha256` / `notes`。
发布新版本：替换 `dist/app-latest.apk`、更新版本号与 sha256 后提交即可。

- 包名：`com.example.holidaycalendar`
- 当前版本：v1.9.6（versionCode 43）
