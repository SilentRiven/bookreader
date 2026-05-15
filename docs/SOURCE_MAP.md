# Legado 源码分层导读

按 `PLAN.md` 阶段安排的阅读顺序。**不要一次性全读**——每个阶段读对应的层，理解后再进下一层。

## 阅读节奏建议

| 阶段（来自 PLAN.md） | 该读的层 | 目的 |
|---|---|---|
| 阶段 0 第一遍 | Layer 1 入口 + Layer 2 数据层 | 知道应用怎么启动、数据怎么存的 |
| 阶段 1 WebDAV 验证前 | Layer 3 配置与同步 | 看懂 `AppWebDav` 调用链，能在 UI 里找到配置入口 |
| 阶段 2 改书架前 | Layer 4 书架与主界面 | 找到要改的 Fragment / Adapter / 布局 |
| 阶段 3 改阅读前（远期） | Layer 5 阅读核心 | 极其谨慎，可能 1-2 周才能读懂 |
| 按需 | Layer 6 书源解析 / Layer 7 服务 | 改到了再看 |

---

## Layer 1：入口与应用初始化

| 文件 | 作用 | 读法 |
|---|---|---|
| `App.kt` | Application 子类，全局单例初始化（CrashHandler、Glide、WebDav、Room、Cronet 等） | 通读，理解启动时初始化了什么 |
| `ui/welcome/WelcomeActivity.kt` | 启动闪屏，处理首次启动 / Intent | 看 onCreate 跳转逻辑 |
| `ui/main/MainActivity.kt` | 主 Activity，Tab 容器（书架 / 发现 / 订阅 / 我的） | 找 ViewPager / BottomNavigationView 用法 |
| `ui/main/MainViewModel.kt` | 主页面 VM，触发版本检查、上次书籍跳转等 | 通读 |
| `ui/main/MainFragmentInterface.kt` | 主 Fragment 共享接口（如 compressExplore） | 短，看一眼 |

**读完应该知道**：app 启动后第一个 Activity 是谁，主界面有几个 Tab，每个 Tab 是哪个 Fragment。

---

## Layer 2：数据层（Room + Entities）

Room 数据库 + KSP，schema 输出在 `app/schemas/`。

| 文件 / 目录 | 关键性 | 备注 |
|---|---|---|
| `data/AppDatabase.kt` | ⭐⭐⭐ | 数据库定义，列出所有 Entity / DAO / Migration |
| `data/DatabaseMigrations.kt` | ⭐⭐⭐ | 所有版本迁移，**改 schema 必加** |
| `data/entities/Book.kt` | ⭐⭐⭐ | 书的核心数据模型 |
| `data/entities/BookSource.kt` | ⭐⭐⭐ | 书源（用户配的抓取规则） |
| `data/entities/BookChapter.kt` | ⭐⭐ | 章节 |
| `data/entities/BookGroup.kt` | ⭐⭐ | **书架分组**——阶段 2 改造点 |
| `data/entities/Bookmark.kt` | ⭐ | 书签 |
| `data/entities/BookProgress.kt` | ⭐ | 阅读进度（用于 WebDAV 同步） |
| `data/entities/ReadRecord.kt` | ⭐ | 阅读统计 |
| `data/entities/ReplaceRule.kt` | ⭐ | 净化替换规则 |
| `data/entities/rule/` | ⭐ | 书源解析规则的子 entity（BookInfoRule / TocRule / ContentRule 等） |
| `data/dao/BookDao.kt` | ⭐⭐⭐ | 书的所有 SQL，**阶段 2 必读** |
| `data/dao/BookGroupDao.kt` | ⭐⭐ | 分组 SQL |
| `data/dao/*Dao.kt` | 其余 | 按需查 |

**读完应该知道**：一本书在数据库里长什么样、分组是怎么存的、SQL 都怎么写、迁移文件的格式。

---

## Layer 3：配置与同步

| 文件 | 作用 | 读法 |
|---|---|---|
| `help/config/AppConfig.kt` | 全局 SharedPreferences 单例 | 通读，知道有哪些全局配置项 |
| `help/config/ReadBookConfig.kt` | 阅读配置（字体、行距、主题、翻页方式） | 通读，阶段 3 改造点 |
| `help/config/ThemeConfig.kt` | 主题（颜色、背景图） | 通读，阶段 3 改造点 |
| `help/config/LocalConfig.kt` | 本地状态（版本号、引导是否完成等） | 浏览 |
| `help/AppWebDav.kt` | ⭐⭐⭐ **WebDAV 同步入口**，阶段 1 必读 | 通读，搞清同步什么数据、什么时候触发 |
| `help/storage/Backup.kt` | 备份导出（含 WebDAV 上传） | 配合 AppWebDav 读 |
| `help/storage/Restore.kt` | 备份恢复 | 配合 Backup 读 |
| `help/storage/BackupConfig.kt` | 备份配置（哪些项备份） | 短，看一眼 |
| `help/storage/ImportOldData.kt` | 老版本数据导入 | 可跳过 |
| `ui/config/WebDavConfigFragment.kt`（路径以实际为准） | WebDAV 设置页 UI | 阶段 1 用户配 WebDAV 的入口 |

**读完应该知道**：在 UI 设置里如何配 WebDAV、点"备份到云端"后调用链是怎样的、同步的是哪些数据。

---

## Layer 4：书架与主界面（阶段 2 主战场）

**关键现实**：legado 的书架有**两套并存的实现**，用户可在设置里切换风格：

- `ui/main/bookshelf/style1/` —— 经典风格（分组在顶部 Tab）
- `ui/main/bookshelf/style2/` —— 现代风格（分组在侧边或网格内）

**改书架时务必两套都改**，否则用户切换风格后看不到你的改造。

| 文件 | 作用 |
|---|---|
| `ui/main/bookshelf/BaseBookshelfFragment.kt` | 两个 style 共享的基类 |
| `ui/main/bookshelf/BookshelfViewModel.kt` | 共享 VM |
| `ui/main/bookshelf/style1/BookshelfFragment1.kt` | 风格 1 主 Fragment |
| `ui/main/bookshelf/style1/books/BooksFragment.kt` | 风格 1 的分组内 Fragment |
| `ui/main/bookshelf/style1/books/BooksAdapterList.kt` | 风格 1 列表 Adapter |
| `ui/main/bookshelf/style1/books/BooksAdapterGrid.kt` | 风格 1 网格 Adapter |
| `ui/main/bookshelf/style2/BookshelfFragment2.kt` | 风格 2 主 Fragment |
| `ui/main/bookshelf/style2/BooksAdapterList.kt` | 风格 2 列表 Adapter |
| `ui/main/bookshelf/style2/BooksAdapterGrid.kt` | 风格 2 网格 Adapter |
| `ui/book/group/GroupManageDialog.kt`（路径以实际为准） | 分组管理 UI |
| `ui/book/info/BookInfoActivity.kt` | 书的详情页 |
| `ui/book/manage/BookshelfManageActivity.kt` | 批量管理界面 |

对应的 layout XML 在 `app/src/main/res/layout/`（如 `fragment_bookshelf*.xml`、`item_bookshelf_*.xml`）。

**改造范围举例（PLAN.md 阶段 2）**：
- 书架卡片样式 → 改 Adapter + `item_bookshelf_*.xml`
- 自定义分类标签 → 扩展 `BookGroup` entity + 加迁移 + 改分组管理 UI
- 批量管理优化 → 改 `BookshelfManageActivity`

---

## Layer 5：阅读核心（阶段 3，**慎入**）

**警告**：这一层涉及大量自绘 View、手势处理、分页算法、动画。改动易引起难以调试的 bug。除非有 4-5 个月 Android 经验，否则**只读不改**。

| 文件 | 作用 |
|---|---|
| `model/ReadBook.kt` | ⭐⭐⭐ 当前书的全局状态机（单例 object）：当前章节、页码、目录、TTS 状态 |
| `ui/book/read/ReadBookActivity.kt` | 阅读 Activity |
| `ui/book/read/ReadBookViewModel.kt` | 阅读 VM |
| `ui/book/read/page/ReadView.kt` | ⭐⭐⭐ 阅读视图（自定义 ViewGroup），手势分发 |
| `ui/book/read/page/PageView.kt` | 单页视图（包含 ContentTextView） |
| `ui/book/read/page/ContentTextView.kt` | ⭐⭐⭐ 文本绘制核心（Canvas 自绘） |
| `ui/book/read/page/delegate/PageDelegate.kt` | 翻页委托基类 |
| `ui/book/read/page/delegate/CoverPageDelegate.kt` | 覆盖翻页 |
| `ui/book/read/page/delegate/SimulationPageDelegate.kt` | **仿真翻页（最复杂，禁区）** |
| `ui/book/read/page/delegate/SlidePageDelegate.kt` | 滑动翻页 |
| `ui/book/read/page/delegate/NoAnimPageDelegate.kt` | 无动画 |
| `ui/book/read/page/delegate/ScrollPageDelegate.kt` | 滚动模式 |
| `ui/book/read/page/provider/ChapterProvider.kt` | ⭐⭐⭐ 章节文本分页算法 |
| `ui/book/read/page/provider/TextChapterLayout.kt`（如有） | 文本布局 |
| `ui/book/read/page/entities/TextChapter.kt`、`TextPage.kt`、`TextLine.kt` | 排版数据结构 |
| `ui/book/read/ReadMenu.kt` | 阅读菜单（点击屏幕中央弹出） |
| `ui/book/read/TextActionMenu.kt` | 长按选词菜单（划线、查词、复制） |
| `ui/book/read/SearchMenu.kt` | 全文搜索菜单 |
| `ui/book/read/config/` | 阅读配置弹层（字体、主题、翻页方式） |

**安全改造点**：
- 主题切换 / 字体管理 → `help/config/ThemeConfig.kt` + 阅读配置弹层
- 设置弹层 UI 重做 → `ui/book/read/config/`

**危险区**：动 ContentTextView 或 PageDelegate 任何一个。

---

## Layer 6：书源与解析

| 路径 | 作用 |
|---|---|
| `model/webBook/WebBook.kt` | 网络书源协调（搜索 / 目录 / 正文 / 探索） |
| `model/analyzeRule/AnalyzeRule.kt` | ⭐ 规则解析器主入口 |
| `model/analyzeRule/AnalyzeByJSoup.kt` 等 | 各种规则后端：JSoup / JsonPath / XPath / 正则 / JS |
| `model/localBook/LocalBook.kt` | 本地书入口 |
| `model/localBook/EpubFile.kt` / `TextFile.kt` / `UmdFile.kt` / `PdfFile.kt` | 各格式解析 |
| `model/Debug.kt` | 书源调试输出 |
| `help/JsExtensions.kt` | 暴露给书源 JS 的 API（HTTP、文件、字符串处理等） |

**关键**：书源 JS 通过 `:modules:rhino` 执行，`JsExtensions` 是给 JS 看的 API 表面，任何重命名/改签名都会让用户的书源失效。

---

## Layer 7：服务

| 路径 | 作用 |
|---|---|
| `service/BaseReadAloudService.kt` 系列 | TTS 朗读后台服务 |
| `service/CacheBookService.kt` | 缓存下载 |
| `service/WebService.kt` | 内置 Web 服务器（端口 1234/1235），见 `api.md` |
| `service/DownloadService.kt` | 一般文件下载 |
| `service/AudioPlayService.kt` | 听书音频 |

---

## 通用阅读技巧

1. **不要试图按文件名顺序读** —— Android 项目类多，按调用链或数据流读才有效率。
2. **从 Activity 入手** —— 找到对应业务的 Activity，看它的 `onCreate` 串到 ViewModel，再串到 Repository / DAO。
3. **善用 Android Studio 的 Find Usages** —— 比 grep 强，能跨 Kotlin / Java / XML。
4. **改前先 git blame** —— 看这块代码最近一次改是为了什么，上下文常常在 commit message 里。
5. **读 layout XML 同等重要** —— UI 改造一半工作量在 XML 里，别只盯着 Kotlin 文件。
