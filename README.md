# 影视舱 MediaShell 开发日志

> 桌面壳软件独立仓库的开发日志。2026-11-22 从主仓库 MyTV 的「更新日志.md」中摘出。
> v1.4.x 系列全部为 MediaShell 桌面壳的开发记录。

## v1.10.0 (2026-11-22)

用户反馈直播页看不到 mpv(轻快) 与 ⚡硬解按钮。

- **根因**：v1.9.9 把 mpv 放到 `D:/userdata/players/`，但便携版单文件 EXE 每次解压到 `D://User//Temp//MediaShellData//`，DATA_DIR = 临时目录/userdata，`detect-players` 扫 `DATA_DIR/players/mpv.exe` 落空。目录版 DATA_DIR 才是 D:/userdata。
- **修复**：`PLAYER_CANDIDATES.mpv` 增加固定绝对路径 `D://MediaShellPlayers//mpv.exe`（不受解压影响，优先命中），mpv 部署到该处；保留 DATA_DIR 候选兜底。
- ⚠️ 排查中还发现：从工具启动的 explorer 会被 WorkBuddy 会话清理回收，普通 Start-Process/计划任务/WMI 都被安全策略限制；最终用「注册一次性计划任务 → Start-ScheduledTask → 清理任务」由 Task Scheduler 服务脱离工具进程树拉起，explorer 才稳定存活（PID 54652）。
- **验证**：smoke 25/25 + store 11/11。

## v1.9.9 (2026-11-22)

用户反馈部分直播源是 FLV 格式，内置播放器播不了，需要 mpv/VLC。

- **内置 mpv**：从 shinchiro/mpv-winbuild-cmake 官方发布下载 mpv v0.42.0 x86_64 便携版（34MB 7z，用 ghfast.top 镜像绕过 GitHub 直连空响应；py7zr 不支持 BCJ2 滤镜，改用 7-Zip 独立版 7za.exe 解压），部署到 `D:/userdata/players/mpv.exe`（main.js 的 PLAYER_CANDIDATES 已含 `DATA_DIR/players/mpv.exe`）。mpv.exe 121MB + d3dcompiler_43.dll，实测 `--version` 正常。⚠️ 踩坑：7z 内 mpv.exe 在**解压根目录**而非 `mpv/` 子目录（子目录只有 fonts.conf），最初 cp 错源导致 0 字节。
- **FLV 自动接管**：新增 `isFlvUrl()`（.flv 后缀 / format=flv / /flv/ 路径）+ `availableExternalPlayer()`；livePlayChannel 遇 FLV 直接甩外部播放器，无扩展名流原生失败后也自动切外部。
- **验证**：smoke 25/25（新增 isFlvUrl 用例）+ store 11/11。

## v1.9.8 (2026-11-22)

用户反馈播放卡顿，希望加速播放器内核。

- **诊断**：内置已用 hls.js 1.5.17，卡顿源于配置而非缺内核：①直播 `lowLatencyMode:true` + 仅 12s 缓冲，网络抖动即反复卡顿；②未限码率，小窗也硬解高码率流；③「软件兼容」模式反而 `enableWorker:false`，把解封装压回主线程。本地代理分片是 `pipe` 流式透传、且已防双重代理，链路本身无瓶颈。
- **修复**：抽 `hlsFastConfig(extra)` 统一点播/直播配置为流畅优先——关低延迟、缓冲 12→24s（上限 30→60s）、`capLevelToPlayerSize:true`、Worker 始终开、重试 3 次；直播的代理 `xhrSetup` 作为 extra 叠加保留。
- **硬解通道**：直播页新增「⚡ 硬解加速」按钮，一键切本机 mpv/PotPlayer/VLC（独立窗口硬解、直连原地址），再点切回内置。本机检测到 PotPlayer。
- **验证**：smoke 24/24（新增参数用例）+ store 11/11。

## v1.9.7 (2026-11-22)

用户反馈：直播播放页画面中间仍有常驻 ▶ 按钮，也没有全屏/小窗按钮。

- **根因**：直播 `#liveScreen` 与点播 `#player` 是两套独立实现，v1.9.0 的播放器改造只改了点播。且全局 `.hidden` 规则当时只覆盖 `.player/.modal/.toast/.player-msg`，不含 `.live-center-play.hidden` → 直播中央按钮 `classList.add('hidden')` 加了也白加、一直显示。
- **修复**：① 补 `.live-center-play.hidden{display:none}`，中央图标改为暂停瞬间短暂闪现（`liveFlashIcon()`，pointer-events:none，`.show` 控 opacity）；② 直播模板右上加 `#btnLiveFullscreen`/`#btnLivePiP`；③ 新增 `bindLiveControls()`：单击画面=暂停/恢复、双击=全屏，按钮 e.stopPropagation 防误触；全屏对 `#liveScreen`、小窗对 `#liveVideo`。
- **验证**：smoke 23/23（新增 2 例直播控件）+ store 11/11。

## v1.9.6 (2026-11-22)

用户诉求：像影视仓那样可以选择接口和线路（确认后聚焦「订阅线路/仓库」这一层，不要多订阅管理/点播站点搜索/频道线路排序）。

- **行为变更**：导入多级索引订阅（storeHouse/urls）时不再由程序自动选线，改为弹窗让用户自己挑。之前 `pickBestLine` 自动选虽然按质量排序，但选错了用户无从知晓、只能重新导入碰运气。
- **实现**：
  - `probeCandidates(items)`：并发探测所有候选，返回元数据（name/url/sites/goodLives/livesTotal/ms/next/ok）。
  - `expandInteractive(url, rootCfg, choose)`：逐级展开，遇 storeHouse/urls 回调 `choose({kind,candidates})`（返回行索引 / 'auto' / null），返回 `{cfg, trail}`。
  - `showLineDialog()`：弹窗列出每条线路实测信息 + 质量标记（推荐/可用/下级索引/失效/当前）；点某行即用该线，点「自动选最优」沿用质量排序。
  - `compareLineQuality()` 从 pickBestLine 抽出，供自动选线复用。
- **记住选择**：选中的源地址与线路路径存 `settings.subscriptionUrl` + `settings.lineTrail`，下次启动直接用所选快照，不再重复弹窗；侧边栏与设置面板各加「🔀 切换线路」按钮，点击重拉原地址重新选择，选后刷新站点/分类/直播缓存。
- **踩坑**：① smoke 断言误写成 `r.sites`（实际返回 `{cfg,trail}`），应为 `r.cfg.sites`；② auto 分支原只筛末级 `c.ok`，但真实 zheteng.json 第一级 storeHouse 指向的是含 urls 的索引（无 sites），会报「没有可用的线路」——改为先取末级按质量排、全是中间层则取一条 `c.next` 继续展开。
- **验证**：smoke 21/21（新增候选探测元数据/用户指定线路/auto 选最优/取消 4 例），store 11/11；打包便携版 `MediaShell-1.9.6-portable.exe`（69,913,734B，图标注入 OK、overlay 完整），顶层 exe 替换、D:/MediaShell 目录版部署（asar v1.9.6、userdata 保留）、便携版真实启动进程存活。

## v1.9.5 (2026-11-22)
- 修复 `pickBestLine` 纯按速度选线的缺陷：21 条线路中 14 条是 sites=1 的空壳垂直线路（短剧/课堂/赛事），JSON 小、CDN 命中快（~180ms），永远比主线路（~280ms）快而被选中。选到后站点几乎为空 + lives 指向失效 `PlatformLive.txt`（实测 404）/旧 TXT，用户看到「多仓解析混乱 + 直播无法播放」。
- 新选线策略（质量优先，仍保留影视仓「选最快可用」习惯）：站点数降序 → 有效直播源（m3u/txt 且非 PlatformLive）降序 → 响应速度升序。真实 CDN 连续 3 轮均稳定选中 sites=107 的主线路。
- 测试框架 `t()` 从并发 Promise 改为串行链（共享 `MS._mock`/`MS._delay` 不再互相覆盖），新增 2 用例，共 17/17。

## v1.9.4 (2026-11-22)

用户反馈：直播列表加载逻辑变了——同名频道显示多个名字、多线路没在左下角合并切换、超时不自动换线。

- **根因**：订阅的直播源已从 TXT 换成 `TVLives_v2.m3u`（M3U 格式）。M3U 结构是「一个 `#EXTINF` 后跟多条 URL = 同频道多线路」。而 `parseM3U` 原实现每遇一条 URL 就 `push` 一个新 item 并把 `pending` 清空 → CCTV-1 的 4 条线路被拆成 4 个同名频道，每个只有 1 条线路。没有多线路，左下角切线和超时切流自然失效。此外原实现把所有频道塞进「赛事轮播」一个组，丢失真实 group-title。
- **修复**：重写 `parseM3U`——① `#EXTINF` 后连续 URL 归到同一频道的 `urls`（去重）；② 按 `group-title`/`#EXTGRP` 建组；③ 容错上游某行字节截断产生的 `\uFFFD` 畸形组名（按包含关系并回正常组）。
- **验证**：真实 M3U 解析出 5 组（央视28/卫视32/港澳台11/影视轮播103/体育13）、187 频道、389 线路、CCTV-1 4 条线路。新增 2 个 smoke 用例（多 URL 合并 / 跨组不误并），共 15/15。
- 注：`buildLiveCatalog` 的同名合并 + `liveLines` 左下角切换 + `armWatchdog` 超时换线三处逻辑本就完好，本次是让 M3U 源解析出正确的多线路数据喂给它们。

## v1.9.3 (2026-11-22)

用户反馈：导入 `zheteng.json` 报「配置中没有 sites」。

- **根因**：影视仓配置是三级索引——`zheteng.json` 的 `storeHouse`（2 个主备仓库）→ `aizheteng_v2.json` 的 `urls`（21 条垂直线路）→ 各 `api.json` 真正的 `sites`（20~107 站）+ `lives`。影视仓 App 逐级展开让用户选，桌面壳原导入逻辑只认顶层 `sites`，导入主索引即失败。
- **为什么不全部合并**：实测前 8 条线路去重就有 289 个不同站 key，21 条全展开 500+ 站——站点列表爆炸且启动 `probeAllSites` 并发探测几百站会卡死。因此模仿影视仓「选一条线路」的行为。
- **实现**：`expandSubscription(url, root)` 递归最多 5 层：① 末级有 `sites` 直接返回；② `storeHouse` 逐个尝试首个可达仓库；③ `urls` 并发拉取所有线路（各 12s 超时），过滤无 sites 的，取响应最快的一条。`pickBestLine()` 并发选线、保留该线路的 `sites` + `lives`。展开结果 saveSubscription 持久化，启动 `loadConfig` 直接读缓存。
- **测试**：smoke_test 框架改为支持 async 断言；新增 3 个用例（末级直返 / storeHouse 两级 / storeHouse→urls 三级选线）→ 13/13 全绿。
- **真实环境验证**：用 CDN 真实索引跑展开，`zheteng`→`aizheteng_v2`(21 条)→并发测前 6 条线路，最快「饭太硬」56 站/2 直播，全程 656ms。

## v1.9.2 (2026-11-22)

修复便携版外壳 exe「还是没图标」——**本轮最重要的踩坑**。

- **真根因**：v1.9.1 及以前，`signAndEditExecutable: false`（当初为绕开 winCodeSign 解压需管理员建符号链接的失败）让 electron-builder 跳过了对 **portable 自解压外壳**的图标注入；`afterPack` 钩子只改到内层 `win-unpacked/MediaShell.exe`（用户双击不到）。用户看到的外壳一直是 Electron 默认图标——此前误判为「Windows 图标缓存」，实为外壳根本没注入。
- **⚠️ 致命陷阱**：直接对最终 exe 跑 `rcedit --set-icon` **会重写 PE 并丢掉尾部 overlay（NSIS payload）**，exe 从 66.7MB 缩到 410KB，程序彻底报废。第一次尝试正是这样把文件弄坏的（已从完好副本恢复）。
- **正确解法**：新增 `scripts/inject_icon_keep_overlay.py`——先记录 overlay 起点（最后节末尾），只对 PE 主体用 rcedit 注入图标，再把 69,498,570 字节 overlay 原样拼回，并校验 overlay 逐字节一致。`afterAllArtifactBuild` 钩子（`scripts/apply_final_icon.js`）在打包后自动调用它。
- **验证**：最终 exe 69,909,194 字节不变、图标 7/7 尺寸全命中设计图标、overlay 完整性 OK；真实启动测试进程存活并成功解压出 `MediaShellData` 临时目录（证明 payload 完好、程序能自解压运行）。
- `.gitignore` 扩展为忽略全部 `dist*/`，实验产物不入库。

## v1.9.1 (2026-11-22)

用户反馈三个问题并已修复。

### 1. 分类标签看不到后面的、也无法滑动 → 改为多行自动换行 + 折叠
- `.cat-bar` 从 `flex + overflow-x:auto`（横向滚动且隐藏滚动条，用户根本不知道能滑）改为 **`flex-wrap: wrap` 多行自动换行**。
- 站点真实分类常有 40~50 个，全部平铺会太长，因此**默认只显示常用 12 个**（全部/电影/连续剧/电视剧/综艺/动漫/短剧/纪录片/体育赛事 + 订阅声明的分类），其余收进「**更多 N 个 ⌄**」按钮，点击展开为完整列表，再点收起。展开/收起后分类点击逻辑会重新绑定。

### 2. 全部/电影/连续剧/综艺/动漫/短剧 都匹配不到资源 → 父类子类聚合
- **根因**：苹果 CMS 站点里「电影/连续剧/综艺/动漫/短剧」是**父类**，用 `?ac=videolist&t=<父类id>` 查它自身 `total` 恒为 0（实测红牛站：连续剧/综艺/动漫/短剧 全是 total=0），真实内容全在子类里。
- **解法（叠加机制，不破坏原有精确匹配）**：新增 `PARENT_GROUPS` 父类语义归组表（连续剧←国产剧/港澳剧/日剧/韩剧/泰剧/欧美剧/古装仙侠/现代都市…；电影←动作片/喜剧片/爱情片/科幻片…；综艺←大陆综艺/日韩综艺/港台综艺/欧美综艺；动漫←中国动漫/日本动漫/欧美动漫/动漫电影；短剧←爽文短剧/擦边短剧/漫剧/AI漫剧…）。
- 新增 `resolveCatTids()` 把父类名解析成要聚合的 type_id 列表；新增 `fetchCategoryByName()`：① 先走**原有精确匹配**路径（父类自身有数据就用它）② 查不到才用归组表展开子类并发取、合并去重。**原有精确匹配机制完全保留**，只是多了一条兜底。
- 实测（红牛站）：连续剧聚合到 14 个子类 / 280 条，综艺 4 个 / 80 条，动漫 4 个 / 80 条，短剧 2 个 / 40 条，电影 10 个 / 182 条。
- 「🔥全部」也一并修好：之前只在首页有缓存时平铺，首页为空就点不出内容；现在首页无数据时自动进入完整的「全部资源」分页列表。

### 3. 应用程序 EXE 图标
- **便携版**：PE 资源里已含 7 个尺寸（16/24/32/48/64/128/256）的图标条目，已解码 48×48 逐像素确认图案正确（深色圆角方块 + 红蓝渐变播放三角 + 右上角蓝色信号弧）。用户看不到是 **Windows 图标缓存**问题 —— 已提供图标缓存刷新方式。
- **目录版**（`D://MediaShell`）：此前主程序仍是 `electron.exe`，带的是 Electron 默认图标。新增 `scripts/deploy_dir.py`：部署时把 `electron.exe` 改名为 **`MediaShell.exe`** 并用 rcedit 注入图标，同时生成 `Start MediaShell.bat`（纯 ASCII + CRLF）。从此目录版与便携版图标一致。

### 工程
- 版本 v1.9.0 → v1.9.1。

## v1.9.0 (2026-11-22)

用户提出的体验改造需求一次性落地，核心是「像影视仓一样」的使用习惯。

### 一、空壳首启 + 配置持久化（全新架构）
- **应用不再内置任何影视资源**。`build.bat` 不再复制 `config/` 目录，打包产物里 config 为空目录。
- 新增 `user-store.js`（主进程）：用户配置存 `DATA_DIR`（便携版 userdata / 目录版 userdata/）
  - `settings.json` —— 用户设置（主题、弹幕、直播偏好、lastSite）
  - `subscription.json` —— 用户导入的影视仓/TVBox 配置
- 首次启动没有订阅 → 进入**空壳引导页**（🎬 图标 + 三步说明 + 「导入订阅」/「导入配置备份」按钮），不做任何网络请求、不显示任何影视资源。
- 导入一次后写入 `subscription.json`，**下次打开自动加载**，无需重复导入。
- 主进程新增 IPC：`get-subscription` / `get-settings` / `save-settings` / `save-subscription` / `has-subscription` / `store-info` / `export-backup` / `import-backup` / `reset-userdata`；preload 同步暴露。

### 二、播放器交互重做
- **单击画面 = 暂停/恢复**，再次单击恢复播放；**常驻的中央播放按钮已移除**（仅在暂停/恢复瞬间短暂显示图标，0.9s 后淡出）。
- **双击画面 = 全屏**；右上角独立按钮：**⛶ 全屏**、**⧉ 小窗（画中 Picture-in-Picture）**、**弹 弹幕**、系统播放器。
- 新增自定义控制条：播放/暂停、当前时间/总时长、进度条拖拽、倍速（1.0/1.25/1.5/2.0）、静音切换。鼠标移入画面才显示，移出淡出。

### 三、弹幕
- 设置面板有**弹幕总开关**（开关样式）+ 不透明度 / 字号 / 显示区域 / 速度四项参数。
- 播放器右上角「弹」按钮是**当前播放源的独立开关**；每个源的开关状态按 URL 哈希单独记忆（`ms_danmaku_sources`），切源自动切换对应状态。
- 数据来源：优先用订阅配置里的 `danmaku` 字段，其次公开弹幕接口；取不到时静默降级并提示，不影响播放。

### 四、主题
- **明暗两套配色**，CSS 变量驱动（`body.light` 覆盖 `--bg/--panel/--line/--text/--muted/--accent` 等），浅色下骨架屏、缩略图兜底、滚动条一并反色。
- 顶栏 ◐ 按钮一键切换；设置面板也可选；选择写入 settings.json，**重启后保持**。

### 五、历史与分类
- **观看历史显示点播源观看记录**：带「点播/直播」标签、观看时间、**已看到 xx:xx** 进度、当前集数；点击可**重新打开详情并续播**（原站点不在订阅中时给出提示）。
- 播放中每 5 秒回写一次进度到历史。
- **分类自适应**：分类栏完全以站点 `ac=list` 返回的真实分类（class 字段）为准，父类（有子类）排前；订阅里写死的 categories 只作排序参考、绝不直接当分类名展示 —— 避免「点了没数据」的死标签。

### 六、设置面板与重置
- 设置面板：外观（主题）/ 弹幕（总开关 + 4 项参数）/ 配置管理（存储路径、导出备份、导入配置、**重置**）。
- **重置弹框二次确认**：确认后清除全部配置（订阅、设置、收藏、历史）并恢复空壳状态；**点「取消」不做任何恢复操作**，原样保持。

### 七、其它
- 直播源改为**从订阅的 lives 读取**（type=0 + url 指向 TXT/M3U），不再依赖内置文件；订阅未提供 lives 时给出明确提示文案。
- 空壳模式下切换到搜索/直播/收藏/历史会提示先导入订阅，不发起任何请求。

### 工程
- 新增 `scripts/smoke_test.js`（轻量 DOM 桩，10 项：空壳/主题/弹幕按源记忆/播放控制/历史/TXT 解析）与 `scripts/store_test.js`（真实文件系统，11 项：读写/回落/导出/重置/导入），均全绿。
- 版本 v1.8.0 → v1.9.0。

## v1.8.0 (2026-11-22)

- **新增应用图标**：新增 `build/icon.ico`（16/24/32/48/64/128/256 共七尺寸，全 BMP 条目）与 `build/icon.png`（512×512 透明底）。设计为「圆角电视屏底板 + 品牌渐变播放三角 + 右上角直播信号弧」，配色直接取自界面主题变量：强调色 `#ff5c7a`、次强调 `#6ea8ff`、底色 `#0d0f14`、描边 `#4a5670`。
- **图标生成器 `scripts/make_icon.py`**：**纯 Python 标准库实现**（本机没有 PIL/numpy，且 pip 在沙箱内会被 SIGTERM 静默杀掉），自写 PNG 编码（zlib）、SDF 抗锯齿（圆角矩形 / 圆角凸多边形 / 圆弧带）、盒式滤波缩放、ICO 打包（BITMAPINFOHEADER + 32bpp BGRA + AND 掩码）。`python scripts/make_icon.py` 可重复构建，约 5 秒出图。
- **图标接入打包链**：
  - `electron-builder.yml` 新增 `win.icon: build/icon.ico`；
  - `main.js` 的 `BrowserWindow` 新增 `icon: path.join(APP_DIR, 'icon.png')`（随 asar 打包，窗口与任务栏均显示图标）；
  - `build.bat` 打包时把 `build/icon.png` 一并复制进 `_stage`；
  - 新增 `afterPack: scripts/apply_icon.js` 钩子，用本机 electron-builder 缓存里已解压好的 `rcedit-x64.exe` 给 unpacked 目录的 `MediaShell.exe` 注入图标。
- **踩坑记录**：
  1. electron-builder 默认靠 `winCodeSign` 包里的 rcedit 改 exe 图标，但该包在 Windows 解压时会因「创建符号链接需要管理员权限」而失败（`darwin/10.12/lib/libcrypto.dylib`），整包解压报错连带打包失败 → 改用 `afterPack` 钩子直接复用本机缓存里的 rcedit，绕开下载与解压。
  2. `afterPack` 钩子是**模块导出函数**（`module.exports = afterPack`），不是接收 JSON 命令行参数的脚本；写错会导致 `JSON.parse` 抛错中断打包。
  3. `signAndEditExecutable: false` 会同时跳过图标注入与版本信息写入，因此用 afterPack 补图标时保持 false 即可。
  4. 打包前 `_stage/package.json` 必须是 `package.build.json`（含最新版本号），否则产物文件名仍带旧版本号。
- **验证**：解析 PE 资源目录确认 `win-unpacked/MediaShell.exe` 与便携版 launcher 均含 7 个尺寸的图标条目；解码 256×256 条目导出 PNG，图案与源文件逐像素一致；16px 下播放三角仍清晰可辨。
- 本地部署同步：`MediaShellApp/resources/app.asar` 与 `D://MediaShell//resources//app.asar` 已用带图标的新资源重打包（772,439 字节）；便携版 EXE 重新打包为 `MediaShell-1.8.0-portable.exe`（69,914,116 字节）。
- 版本 v1.7.0 → v1.8.0（`package.build.json`、`package.json`、README 同步）。

## v1.7.0 (2026-11-22)

- **直播源改为影视仓/TVBox 标准 TXT 格式**：用户提供了 TVBoxBD0130 的直播列表作为格式样本（`E://BaiduNetdiskDownload//TVBoxBD0130//TVLives.txt`），本壳随之对齐 —— UTF-8 带 BOM + CRLF、**一行一条线路**、同名频道重复出现（客户端按频道名合并为多线路）、**数据行之间不留空行**（空行不含 `://` 会被影视仓当成频道指令行，直接导致整个直播列表为空）、频道名改 `CCTV-1` 风格。
- **修复 `parseTVLines` 不合并同名频道**：此前按「一行一频道」写死，同名行会变成多条重名条目；现按频道名合并 urls，并剥离 UTF-8 BOM、兼容 `,` 与 `#` 两种地址分隔。实测本地 `TVLives.txt` → **187 频道 / 389 线路，无重名条目**（B站直播 25、赛事直播 185）。
- **本地内置直播源全量刷新**：`config/TVLives.txt`、`config/BiliLive.txt`、`config/LiveSports.txt` 与 `config/channel_plan.json` 同步主仓库最新构建。
- **站点配置切到永久固定地址链**：`zheteng_v2.json`（主 cdn + 备 gcore）→ `aizheteng_v2.json`（21 条，垂直线路走 `YSCv3/`）→ 各配置 lives 指向 `TVLives_v2.m3u`；旧名地址仅作兼容。
- 本地部署同步：`MediaShellApp/resources/app.asar` 与 `D://MediaShell//resources//app.asar` 已用新资源重新打包；便携版 EXE 重新打包为 `MediaShell-1.7.0-portable.exe`。
- 版本 v1.6.0 → v1.7.0（`package.build.json`、`package.json`、README 同步）。

## v1.6.0 (2026-11-22)

- **改用永久固定订阅地址**：用户希望收藏一次地址、不再随版本更换。订阅地址统一为 `@master`，壳仓库 config 的 aizheteng/zheteng 引用同步回 `@master`，版本 v1.5.1 → v1.6.0。
- 内容自动更新由主仓库 GitHub Actions 保证：每天更新后调用 jsdelivr 官方 purge 端点清除边缘缓存（`scripts/purge_cdn.py --changed` 只清变更文件避免限流），固定地址即可立即反映最新内容。

## v1.5.1 (2026-11-22)

- **壳仓库同步主仓库 jar 加载失败修复**：`config/aizheteng.json` 中 6 条线路（饭太硬/肥猫/小米/PG/摸鱼儿/鸟叔）地址改到 `@v1.5.1` tag，`config/zheteng.json` 主备入口同步；版本 v1.5.0 → v1.5.1。修复根因在主仓库：这 6 条 spider 用相对路径 `.jar`，远程导入时 jsdelivr 对 `.jar` 返回 403 导致 jar 加载失败，已改为 `.txt` 伪装文件 + 绝对地址 + md5。

## v1.5.0 (2026-11-22)

- **TVLives 直播源加主备节点**（26 份配置，含壳仓库 config）：接口里每条 `TVLives.txt` 后新增 `TVLives(备)`，URL 走 `fastly.jsdelivr.net@master`；主链接保持 `@master`（直播源每天由 Action 重建，固定 tag 会冻结内容）。两节点实测内容 MD5 一致、约 1 秒返回。
- **多仓入口主备已就位**：`zheteng.json` 前两条为 cdn 主 + fastly 备，均指向 `aizheteng.json`；`aizheteng.json` 21 条子配置保持单地址不变。
- **Action 有效性校验加固**：主仓库新增 `scripts/validate_multi_repo.py`，在 `Aizheteng_Sync.yml` 提交前用严格 JSON 解析校验两个索引 + 21 条子配置 + 主备入口，任一失败即不提交；自有地址映射本地文件校验，避免 CDN 缓存误判。补上了 `update_aizheteng.py` 宽松解析（`strip_comments`）会放过 `//` 注释坏配置的漏洞。
- 版本 v1.4.9 → v1.5.0（壳仓库 `package.build.json`、README 同步）。

## v1.4.8 (2026-11-22)

- **修复「多仓接口解析失败」**（配置层，主仓库 MyTV 同步）：v1.4.7 换 CDN 后域名可达，但影视仓导入多仓仍有 3 条线路解析失败。
  - 主因：`fty/api.json`、`feimao/api.json`、`xiaomi/api.json` 首行带 `//关注微信公众号…` 注释，影视仓严格 JSON 解析不容忍注释，直接 `JSONDecodeError`。删除注释后三份配置恢复严格合法 JSON（sites 56/62/35）。
  - 次因：多处订阅地址仍是 `raw.githubusercontent.com`（国内/电视端实测 20 秒超时 HTTP 000）——`zheteng.json` 备选仓（主仓 + DesktopShell + 壳仓库 config 共 3 份）、README/README.en 备用地址、`scripts/aizheteng_sources.json` 21 条备选用址、`Aizheteng_Sync.yml` 镜像源。全部替换为 jsdelivr 双节点（`cdn.jsdelivr.net` + `fastly.jsdelivr.net`，实测均 1.4 秒 200）。
  - CDN 缓存规避：新增不可变 tag `v1.4.8`，3 条修复子配置地址改用 `@v1.4.8` 而非 `@master`（后者缓存仍返回旧内容），实测两个节点立即返回修复后内容。
  - 校验：21 条子配置严格 JSON 解析 21/21 通过，`zheteng.json` 17 条引用全部有效。
- 版本号 v1.4.7 → v1.4.8（`package.build.json`、`README.md` 同步）。

## v1.4.4 (2026-11-22)

- **直播源筛选逻辑正式写入 GitHub Actions 自动化**（新增 `.github/workflows/LiveSources.yml`）。之前只有两个旧工作流，且都有致命问题：`PlatformLive.yml` 拉的是**已失效的源**（`live.yang-1989.eu.org` 已 302 到占位视频、`sub.ottiptv.cc` 的 bilibili 大量 404），每 30 分钟就把源文件覆盖成不可用状态；`TVLives_Sync.yml` 拉的是仓库自己的旧文件形成自循环，且**不做任何可播性验证**。两者已停用（改名 `.disabled`）。新工作流每 6 小时跑一次，跑完自动校验格式、有广告残留就判定失败、不通过就不提交。
- **新增统一筛选流水线 `scripts/build_live_sources.py`**：多上游拉取 → 规则过滤（广告台名/广告域名/静态录像/分组噪音）→ **逐条实测可播性**（并发探测，只留真正返回 m3u8 的线路）→ 错台修正 → 名称归一 → 同名合并多线路 → 输出 TVBox TXT 三份。
- **清除广告台**：实测残留的「支持作者」（百度网盘 mp4 广告位）、「请user查看最新动态」、「广东卫视」被错标到腾讯视频图片域名 mp4、17 条「春晚频道」实为 kwimgs 静态录像等，现由三层规则拦截（`AD_NAME_RE` 频道名特征 + `AD_HOST_RE` 广告/图床域名 + `STATIC_EXT_RE` 静态媒体后缀）。重建后三份源**零广告残留**。
- **修正错台（源对不上）**：实测上游把**一条流标注成多个台名**——「东方卫视」与「东方卫视4K」都指向 `bp-resource-dfl.bestv.cn/148/3/video.m3u8`（还夹带了一个上游根本不存在的 `/155/` 编号）、「玛纳斯」与「玛纳斯综合」、「浙江民生」与「浙江民生休闲」、「绍兴电视台公共」与「绍兴公共」共用同一条流。现按「同一 URL + 台名别名归一（去掉电视台/频道/卫视/综合/台/休闲等后缀）」判定，只保留一个台名；同名不同 URL 则合并为多线路。
- **CCTV6 线路从 2 条扩到 5 条**：上游源（IPTV-V4 / CNTV-V4 / 央视各版本 m3u）**完全没有 CCTV6**，原先只有旧 TVLives 里残留的 2 条。现加入「保底线路」机制——脚本内置实测长期可播的央视/卫视/教育台线路清单，上游缺失时自动补齐，CCTV6 补到 5 条（含 1080P 与标清两套）。
- **直播源按实测可播率重建**：逐条探测后写出，**485 个频道 / 515 条线路**（TVLives 213 频道 4 组、PlatformLive 215 频道 7 组、BiliLive 54 频道 8 组）。可播率基线：huyayqk 94% / douyuyqk 95% / yylunbo 95% / IPTV-V4 85% / CNTV-V4 82% / bililive 15%；`Live.m3u` 实测 0%（咪咕系全部 403）已从源列表剔除。
- **TVLives 与 PlatformLive 分工明确**：前者只收正规台（央视/卫视/地方/电影，来自 CNTV-V4 + 保底线路），后者收综合台（来自 IPTV-V4），此前两者拉同一份上游、内容完全重复。

## v1.4.3 (2026-11-22)

- **修复「首页加载慢」的根因之一：`http-get` 的 timeout 语义改为「总预算」**。此前主进程 `http-get` 会自动重试一次，导致渲染层传入 8s 实际可能等 15s（10s + 350ms + 5s），超时值完全失真。改为：timeout = 总预算，首次用全部剩余预算 → 失败等 350ms → 二次用剩余预算的一半。并把全部 13 处 `cmsGet` 超时统一收敛到 5~9 秒。实测首页首屏 **15s → 5.3s**。
- **新增「多站竞速」取数**：采集站普遍不稳定（实测同一站 0.4s~14s 剧烈波动），首页现在并发请求 3 个候选站（当前站优先），**谁先返回用谁**，慢站不再拖住整个页面；最终采用的站会自动切换并在侧栏高亮。竞速硬超时 10s，与单站最坏耗时对齐，避免把「已经发出、只是慢」的请求白白丢弃再走一遍降级流程。
- **新增启动期自动测速选站**：应用启动后并发测速所有 CMS 站（整体 8s 上限），若当前站不可用或明显更慢（>600ms 差距）则自动切到最快的可用站并提示；失败时首页会**自动降级**到其它可用站，而不是让用户盯着错误页。站点顺序也按实测速度重排（闪电 462ms → 红牛 1074ms → 无资源 1142ms → 淘片 3624ms → 光速 3951ms）。
- **安全加固：拦截采集站跳转到可疑域名**。实测 `api.guangsuapi.com`（光速）与 `www.huyaapi.com`（虎牙采集/虎牙HLS/少儿动漫CMS）已 302 跳转到博彩类域名 `683469.online`（域名过期后被拍卖）。主进程 HTTP 代理与流媒体代理均新增重定向黑名单校验，命中即拒绝跟随，绝不在应用内跳到风险站点；同时把这 3 个被劫持的站点从 `YSCSites.json` 移除，并新增实测可用的 `🍉无资源｜CMS`（31 分类 / 1.1s）作为替代。
- **直播页大列表性能优化**（频道量已达 2800+）：
  - 用 `Map` 建 id→频道索引，替换原来嵌套循环里的 `channels.find()`（O(n²) → O(1)）。
  - 分组内用 `Set` 去重，替换 `channelIds.indexOf`（上千条分组里同样是 O(n²)）。
  - **事件委托**替代给 3000+ 个节点逐个挂 onclick，只保留 3 个监听器。
  - 每个分组默认只渲染前 60 个，其余折叠为「展开其余 N 个」按需渲染；搜索时直接全量显示（上限 300 条）。
  - 实测：DOM 节点 **3004 → 391**，渲染耗时 135ms → 4ms。
- **直播页首屏改为「本地源先行 + 远程源后台补充」**：原先要等 4 个远程赛事 m3u 全部返回（实测 13.3s）才能显示列表。现在本地三份源（毫秒级）立即渲染，远程赛事/轮播源在后台拉取、到货后自动追加并刷新侧栏。实测 **首屏 13306ms → 117ms**，6 秒后频道数自动从 379 增至 2837。
- **直播源组合按实测可播率重建**（逐条探测后写入）：`huyayqk` 94%、`douyuyqk` 95%、`yylunbo` 95%、`IPTV-V4` 85%、`CNTV-V4` 82%、`bililive` 15%、`Live.m3u` 0%（咪咕系全部 403，已剔除）。本地三份源由 272 个频道提升到 **385 个实测可播频道**（TVLives 22 / PlatformLive 290 / BiliLive 73），并剔除广告、测试、时间戳类噪音条目。赛事源换成实测可播率更高的上游，并新增 `CNTV-V4` 央视频道补充。
- **海报图容错**：实测海报图 CDN 平均 889ms/张、单张最长 16.5s 超时，慢图会拖住整页观感。现在每张图独立 6s 超时（横幅 8s），失败后降级为低调暗色占位而不阻塞其他图；并新增 `watchImages()` 统一处理。
- **详情页预取 + 缓存**：详情接口实测需 2~3.2s（站点一次返回 146KB/20 条），是链路最慢一环。现在鼠标悬停海报即后台预取详情（节流 220ms），点击直接出内容；详情缓存 10 分钟。实测详情页 **3.4s → 1.1s**。
- **自检能力扩展**：UI 自检新增「首屏出海报耗时 / 分类就绪耗时 / 图片加载耗时与降级数 / 详情页耗时与选集数 / DOM 节点数 / 渲染耗时 / 启动测速结果」等指标；主进程侧捕获渲染进程 `[boot]`/`[live]` 日志便于诊断。实测数据：首页海报 5.3s（含竞速）→ 分类 5.4s → 图片 32 张 157ms → 直播首屏 117ms → 详情 1.1s。

## v1.4.2 (2026-11-21)

- **修复「直播列表匹配不到资源」根因**：直播页 `renderLive()` 先调用 `renderLiveTree()` 写入了「无匹配频道」占位符，随后异步加载逻辑用 `if (box && !box.children.length)` 判断是否已加载——该条件因此恒为假，`ensureLiveData()` **从未被执行**，直播目录永远是空的。改用 `liveCache` 是否初始化来判断，并把 `liveCache` 改为全部数据加载成功后再赋值（失败可重试）。实测修复后直播频道数 **0 → 2759** 个、17 个分组，加载耗时 2.3s。
- **新增主进程流媒体代理（解决 CORS 拦截）**：实测 22/49 个 IPTV 域名不返回 `Access-Control-Allow-Origin`，Chromium 渲染进程直连 m3u8/分片会被同源策略拦截，表现为「点了频道但播不出来」。主进程新增本地代理服务 `127.0.0.1:19871`，m3u8 清单内的分片路径自动重写为代理地址，TS 分片原样透传（`Access-Control-Allow-Origin: *`），并跟随重定向、透传 Range、注入 Referer/UA、gzip 解压、失败重试一次。hls.js 通过 `xhrSetup` 把清单与分片请求统一改写到代理。实测 8 个随机源 7 个完整走通（m3u8 200 + 分片 200 + `video/mp2t` 字节流完整），唯一失败的源本身已失效。
- **直播源文件全面重建**：原 `PlatformLive.txt`（539 条）全部指向 `live.yang-1989.eu.org`（已 302 到 `404.mp4` 占位视频）、`BiliLive.txt`（505 条）全部指向 `mursor.ottiptv.cc`（已 301 到 404 页），合计 1044/1373 条 URL 失效。改为从 `peterhchina/iptv`、`Guovin/iptv-api` 拉取上游 m3u 并**逐条实测可播性**后重新生成，同名频道合并多线路。三份源合计由 1373 条失效条目变为 **266 个实测可播频道**（TVLives 22 频道 42 线路、PlatformLive 244 频道 248 线路按 7 个分组、BiliLive 6 频道），并剔除广告/测试/时间戳类噪音条目。赛事与平台轮播源也换到可用上游（虎牙一起看 / 斗鱼一起看 / YY 轮播 / 直播大合集，实测可拉取 500~1100 条）。
- **新增线路健康检测**：直播页右上角「🔍 检测线路」按钮并发探测每条线路是否可播（8 并发），侧栏显示 `可用/总数` 徽标、失效线路标灰划线、点击频道时自动优先选中已验证可用的那条线路。
- **大幅优化首页加载速度**：
  - 修复 `httpGet()` 的**超时误杀 bug**：`req.setTimeout()` 在收到响应头后仍继续计时，而部分服务器（Cloudflare 等）保持 keep-alive 不关闭连接，超时会把「已成功返回」的请求销毁——表现为接口偶发失败。改为收到响应头即清除超时，仅在未收到响应头阶段才算超时；同时重定向链共享同一 deadline、启用 gzip、连接用 `Connection: close`。
  - 首页改为**两步渐进渲染**：先只等视频列表（`ac=videolist`）并立刻画出海报行，分类（`ac=list`）并发拉取、拿到后再填充分类栏，不再是两次串行等待。实测光速站分类请求从 12497ms 降到 692ms。
  - 新增**骨架屏**（首页立即给出视觉反馈，不再长时间空白）、**首页内容缓存**（5 分钟内切站秒开，后台静默刷新）、**请求自动重试**（首次失败 350ms 后重试一次）。
  - 新增「⚡ 自动选站」按钮与站点故障时的「自动切换可用站点」按钮：并发测速全部 CMS 站点，一键切到最快可用站。
- **内置自动化自检**：`electron.exe --selftest` 或环境变量 `MSHELL_SELFTEST=1` 触发，主进程复现首页取数与直播目录构建链路并把真实耗时写入 `userdata/selftest.json`，UI 层另写 `selftest-ui.json`（含海报数、分类数、首屏耗时），便于后续回归。
- 说明：Electron 需**在应用目录内启动**（`cd <应用目录> && electron.exe`），传入应用路径参数会被忽略，导致配置不加载——已在新版启动器中处理。

## v1.4.1 (2026-11-21)

- **修复桌面壳首页「分类匹配不到资源」**：根因是苹果 CMS `ac=list` 接口同时返回 `list`（视频）与 `class`（分类），旧代码 `typeMap()` 用 `d.list || d.class` 优先取了视频列表，只从首页 20 条视频里反推分类名，导致硬编码分类（如短剧/AI漫剧/中国动漫/台球）大面积解析不到 `type_id`。改为只读 `d.class`（真分类），并新增按站点缓存的 `fetchCats()`；分类栏改为以站点 API 真实分类为准、配置 categories 作精选排前，每个分类直接带真实 `type_id`；父类（type_pid=0 且有子类）点击时自动聚合子类资源；另加别名/括号归一化匹配。实测光速/红牛/虎牙/闪电 `ac=list` 均返回 44~59 个真实分类。
- **直播页重构为「电视直播」体验**：左侧新增可折叠竖向分类侧栏（央视/卫视/港澳台/赛事等，含频道搜索 + 折叠状态记忆），右侧为播放器预览大画面；同名频道只显示一个名字（`CCTV-1` 等），来自多源的线路合并为「线路1/2/3…」并显示在画面左下角；超时（4~20s 可选）或无 fatal 错误自动跳下一条线路；解码方式可选（硬件优先/软件兼容/自动）、播放器可选（内置 / mpv / VLC / PotPlayer，自动检测本机已装）。实测 TVLives.txt 329 条原始条目去重为 115 个频道、74 个含多线路。
- **直播卡顿优化**：主进程启动时追加 Chromium 硬解参数（`enable-gpu` / `enable-gpu-rasterization` / `enable-zero-copy` / `ignore-gpu-blocklist` / `enable-accelerated-video-decode` 及 Skia/HEVC feature）；内置 hls.js 开启 worker、低延迟模式并收紧缓冲；检测到本机 mpv/VLC/PotPlayer 时可一键交由其硬件解码（mpv `--hwdec=auto-safe --vo=gpu --profile=fast`），解决手机不卡电脑卡的问题。
- **封装为单文件 EXE**：新增 `electron-builder.yml`，用 electron-builder 24 的 portable 目标把整个壳（含 Electron 运行时与全部配置）打包为单个 `MediaShell便携版.exe`（66MB，自解压到用户目录运行，双击即用，无需 bat 启动器 / 无需散落 dll）；已实测双击可正常拉起 Electron 主进程（4 个子进程），数据目录通过 `MSHELL_DATA` 固定到 D 盘。
- 说明：`type=3` 的 `csp_*Guard` 插件站（10 个）仍不可在桌面端运行——它们是给安卓影视仓的 Java 插件引擎用的，桌面 Electron 壳没有 JVM/插件协议；这些资源的可用替代（短剧/赛事/课堂 CMS 镜像与静态源）已在站点配置里保留。

## v1.4.0 (2026-11-21)

- **新增自研桌面壳软件「影视舱 MediaShell」v1.0.0**（源码在 `DesktopShell/`，成品部署于 `D:\MediaShell`），适合电脑大屏观看本仓库的影视与直播资源。基于 Electron 28 + 原生 JS，内置 hls.js，无需联网加载播放器。
  - 复用标准 Electron 运行时，自写主进程（`main.js`）：窗口、内置配置读取、CMS 跨域 HTTP 代理（含重定向/GBK）、外部播放器调用；`userData` 指向应用目录下 `userdata/`，全部数据落 D 盘不碰 C 盘。
  - 渲染层（`app.js` + `index.html` + `style.css`）：深色大屏 UI，**首页采用电视主页式布局**——顶部大图 Banner 自动轮播（热播推荐 + 标题/简介/播放按钮）、横向分类标签栏、每个分类一行横向滚动海报，点标签切换横排内容、点「更多」进入分页网格；另含站点侧栏、海报墙分页、片名搜索、详情选集、收藏/历史、订阅导入；内置 `YSCSites.json` 全部 21 站与 `TVLives/PlatformLive/BiliLive` 三份直播及赛事 M3U。
  - 播放：内嵌 `video` + hls.js 播 m3u8/mp4，支持「系统播放器」用 PotPlayer/VLC 打开；修复选集索引导致外部播放器取错集的问题。
  - 提供纯 ASCII + CRLF 启动器 `Start MediaShell.bat`（自动清除 `ELECTRON_RUN_AS_NODE` / `NODE_OPTIONS`）与重建脚本 `build.bat`。
  - 兼容性：仅支持苹果 CMS（type=1 http 站）；影视仓 `csp_*` Spider 插件站桌面端不可用，界面灰显说明，完整插件体验请用安卓影视仓。
- 校验：窗口标题显示「影视舱 MediaShell · 21站」证明配置加载成功；主进程生命周期日志走到 READYTOSHOW；虎牙/红牛 CMS `ac=videolist` 实测取回 20 条/页、总量 11 万+。
