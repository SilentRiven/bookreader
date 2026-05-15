# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目背景

本仓库的代码起点来自 [gedoor/legado](https://github.com/gedoor/legado)（"阅读 3.0"，开源 Android 小说阅读器），但**作为独立项目维护，不与上游同步**。改造方向见 `PLAN.md`。**始终先读 `PLAN.md`** 再决定改动范围——里面明确列出了"刻意不做"的事情（自建同步后端、重写仿真翻页、重写 TTS/字典/净化/换源 等）。

由于已与上游解耦，本项目可以自由调整 JDK 版本、包名、CI 流水线、依赖版本等。当前已做的偏离：
- JDK 17 → 21（`app/build.gradle` 的 `jvmToolchain` 与 `compileOptions`，以及 `.github/workflows/*.yml` 的 `setup-java`）
- 禁用 Firebase（不向上游 Firebase 项目上报数据）
- `settings.gradle` 启用阿里云 Maven 镜像

## 构建与运行

JDK 21 是当前要求（`app/build.gradle` 配置 `jvmToolchain(21)`，`sourceCompatibility/targetCompatibility = VERSION_21`）。Gradle 通过 `libs.versions.toml` 版本目录管理依赖。

```bash
./gradlew assembleAppDebug              # debug APK，applicationId 后缀 .debug
./gradlew assembleAppRelease            # release APK，需在 gradle.properties 配置签名
./gradlew :app:installAppDebug          # 装到连接的设备
./gradlew clean
```

构建产物：`app/build/outputs/apk/app/<buildType>/legado_app_<version>.apk`。

签名变量（非必填，缺失时 release 走未签名构建）：在 `gradle.properties` 末尾追加 `RELEASE_STORE_FILE` / `RELEASE_KEY_ALIAS` / `RELEASE_STORE_PASSWORD` / `RELEASE_KEY_PASSWORD`。CI 的做法见 `.github/workflows/test.yml`。

Cronet（Chromium 网络栈）jar 与 .so 由 `app/download.gradle` 在首次构建时按 `gradle.properties#CronetVersion` 下载到 `app/cronetlib/` 和 `app/src/main/assets/cronet.json`。**首次构建必须能访问 `storage.googleapis.com`**，国内网络需自备代理或预先准备好这些文件。

`flavorDimensions = ['mode']` 只有一个 flavor `app`；`buildType` 有 `debug` 与 `release`，所以变体名是 `appDebug` / `appRelease`。CI 还会通过 `sed` 临时把 `.release` 改成 `.releaseA` 来产出"共存版"——本地无需关心。

## 测试

```bash
./gradlew :app:testAppDebugUnitTest                                      # 本地 JVM 单测（app/src/test/）
./gradlew :app:connectedAppDebugAndroidTest                              # 需连接设备/模拟器，跑 app/src/androidTest/
./gradlew :app:testAppDebugUnitTest --tests "io.legado.app.JsTest"       # 跑单个测试类
```

测试覆盖率很低：`app/src/test/` 只有 `JsTest` + 模板示例；`app/src/androidTest/` 含 Room `MigrationTest`、`HttpTtsTest`、`HttpTest`、`UpdateTest`、`AndroidJsTest`。**修 Room 实体或迁移时必须跑 `MigrationTest`**——Room schema 输出在 `app/schemas/`，是迁移测试的黄金来源。

## 模块结构

Gradle 多模块，`settings.gradle` 中 `include`：

- `:app` —— Android 应用主体，包名 `io.legado.app`
- `:modules:book` —— EPUB / UMD / Chm 等本地书籍格式解析器（Java），`me.ag2s.*` 包名
- `:modules:rhino` —— 内嵌 Mozilla Rhino JS 引擎，给书源 JS 规则执行用

`modules/web/` 是独立的 Vue 3 + Vite Web 端书架/源编辑器，**与 Android 构建无关**，由 `.github/workflows/web.yml` 单独构建，把产物注入到 `app/src/main/assets/web/`。改 Android 代码时不需要管它。

## Android 代码组织（io.legado.app）

| 包 | 职责 | 备注 |
|---|---|---|
| `api/` | Content Provider + Web 接口（见 `api.md`） | 外部唤起、Web 后台 |
| `base/` | Activity / Fragment / ViewModel 基类 | 改 UI 模式时常碰 |
| `data/` | Room 数据库、Entity、DAO | 改 schema 必走迁移 + `MigrationTest` |
| `help/` | 全局工具与单例配置（`AppWebDav`、`CacheManager`、`TTS`、`config/*`、`storage/*` …） | "杂物间"，但很多核心逻辑住在这里 |
| `model/` | 书源规则解析、本地书籍解析、网络抓取、阅读控制（`ReadBook`、`AudioPlay`、`ReadAloud` …） | 改阅读行为常在这里 |
| `lib/` | 内嵌的第三方/二次封装（如 epub、code-view） | 来自外部库的"vendored"代码，谨慎改 |
| `service/` | 后台服务（下载、TTS、Web 服务器、缓存） | 长期运行的能力 |
| `ui/` | 所有界面，按业务划分（`book/`、`book/read/`、`association/`、`config/` …） | 阅读核心在 `ui/book/read/` |
| `web/` | 内置 nanohttpd Web 服务器（端口 1234/1235） | 配合 `modules/web` 前端 |
| `receiver/` `utils/` `constant/` `exception/` | 见名知意 | |

阅读核心子树（**改之前先看懂**）：

- `ui/book/read/ReadBookActivity.kt` + `ReadBookViewModel.kt`：阅读页 Activity / VM
- `ui/book/read/page/ReadView.kt` + `PageView.kt` + `ContentTextView.kt`：自绘文本与页面
- `ui/book/read/page/delegate/`：翻页动画委托（覆盖 / 仿真 / 滑动 / 滚动）
- `ui/book/read/page/provider/`：分页计算
- `model/ReadBook.kt`：当前书的全局状态机
- `help/config/`：阅读配置 / 主题 / 排版

## 数据层关键点

- Room 数据库 + KSP 编译器；schema 落在 `app/schemas/`，**这些 JSON 文件必须随实体改动一起提交**，否则 `MigrationTest` 会失败
- 全局共享配置走 `help/config/` 下的 `AppConfig` / `ReadBookConfig` / `ThemeConfig` 等单例
- 书源规则系统：`data/entities/BookSource.kt` 定义数据模型，`model/analyzeRule/` 负责解析（JSoup / JsonPath / JsoupXPath / Rhino JS 多种规则混用），`model/webBook/` 负责整本/搜索/章节抓取
- 本地书籍解析：`model/localBook/` 处理 TXT / EPUB / UMD / PDF 等，依赖 `:modules:book`

## 网络栈

- 默认 OkHttp，外加 Chromium Cronet（动态下载，见上文）
- 部分书源逻辑通过 Rhino 执行 JS（`help/JsExtensions.kt` 暴露给 JS 的能力，`modules/rhino`）

## 改动前的检查清单

1. **看 `PLAN.md` 是否已经把这事划为"不做"**——如果是，停手讨论清楚再继续
2. **动 Room 实体？** 加迁移 + 更新 `MigrationTest`，否则线上升级会清库
3. **动 `ui/book/read/page/` 自绘代码？** 极易破坏翻页/分页/选词，建议先在 issue 或对话里说明意图
4. **动书源规则相关（`model/analyzeRule/`、`model/webBook/`）？** 这是用户配置的运行时契约，破坏后所有书源失效，必须确认向后兼容
5. **新增第三方依赖？** 通过 `gradle/libs.versions.toml` 版本目录添加，不要直接在 `app/build.gradle` 写 `implementation "group:name:version"`

## 文档参考

- `README.md` / `English.md`：legado 上游用户向介绍（功能、社区、依赖致谢），内容尚未本地化为本项目身份
- `api.md`：Web 接口（端口 1234 HTTP / 1235 WS）与 Content Provider 接口
- `app/src/main/assets/updateLog.md`：更新日志（用户可见）
- `app/src/main/assets/web/help/md/appHelp.md`：应用内帮助
- `PLAN.md`：**本项目的开发规划（必读）**
- `docs/SOURCE_MAP.md`：起点代码的分层导读
