# dsh-stardew-farm —— 星露谷小农场

左下角的一小块像素农场，跟着 DeepSeek Harness 一起出现：AI 干活时小人自己在 26 块地里种田，
每敲一次键盘就在对应的字母地里种下一株作物（中文输入法上屏也算），同一块地按 3 次成熟，小人跑过去收割换金币。
**AI 真的在向你提问**时，小人才会抬头喊「老乡，有个事要你处理…」；切换任务时会说「老乡，我该做下一步了」。

同一套页面脚本有**两种运行方式**，共用同一个 `farm.json`（v5）格式：

- **DSH 插件**：注入 DeepSeek Harness 页面左下角，跟随会话待办 / AI 忙闲 / 提问。
- **Windows 桌面版**：Electron 透明置顶窗口 + 托盘常驻，`uiohook-napi` 抓全局按键，
  不要求农场窗口有焦点；后台记录前台软件工时，检测到 harness 自动开工。

![农场页](docs/tab-farm-crop.png)
![键盘农场页](docs/tab-keys-crop.png)
![工时页](docs/tab-usage-crop.png)

## 界面：五个页签

```
┌──────────────────────────────────────────┐
│ FARM  <用户名> 的农场              ● 金币 ☰ ▾ │
│ 10-25  干活中  2 待办  12s  桌面           │
│ [农场] [键盘] [任务] [工时] [余额]          │
├──────────────────────────────────────────┤
│ 土地 · 26 块 A–Z              成熟 0 · 收割 1 │
│  A B C D E F G                            │
│  H I J K L M N                            │
│  O P Q R S T U                            │
│  水塘  V W X Y Z  记录员                   │
│ 已工作 12s | 帮手 0/4 | 金币 4             │
└──────────────────────────────────────────┘
```

| 页签 | 内容 |
|---|---|
| 农场 | 7×4 田地，前三行 7 格、第四行 5 格居中；26 块固定地各带字母 A–Z；方形/圆形两块 SunnysideWorld 泥土贴图交替铺地，左右两个空格放水塘和记录员；底部状态栏显示工时 / 帮手 / 金币 |
| 键盘 | QWERTY 三行（10+9+7）对应同一批 26 块地；24px 键帽上是泥土贴图 + 作物 + 成长格 + 按键次数，不会溢出 |
| 任务 | 今日待办（SSE 实时同步 + 启动水合）+ 完成进度条 + 当前项高亮 + 点击选中 |
| 工时 | 今日前台软件使用时长、正在用的软件、软件排行、检测到的 harness |
| 余额 | DeepSeek 余额 |

## 玩法

| 机制 | 说明 |
|---|---|
| ⌨️ 每个字母 = 一块地 | A–Z 固定 26 块地，按哪个字母就种哪块；中文/符号按哈希落到 A–Z，控制键忽略 |
| 🌱 3 击成熟 | 同一块地按 3 次立刻成熟并**马上结算 +4 金币**，保留 0.9 秒成熟动画后清地，不会卡住 |
| ⌨️ 键盘 / 农场同一份状态 | 键盘页和农场页读写同一个 `state.plots`，键盘 B 键长到 2/3，农场 B 地就是 2/3 |
| 💰 劳动也产金币 | AI 忙、正在打字、前台有正经软件、检测到 harness 时，小人会自己在田里干活：走到空地自动种下、给成长中的地浇水、成熟后收割，每分钟约 6 金币 |
| 🐾 帮手 | 每累计工作 5 分钟跳出一只小动物帮手，最多 4 只；超过 900 秒没有工作就清零。帮手有自己的选地 / 走路 / 干活状态机，不是跟着小人走 |
| 📋 待办水合 | 插件启动时先从 `session_projcache` 读一次，再用 `sessionQuery.readSession` 找最后一条 `todo/write`；之后走 SSE，实时更新，不会丢启动前的待办 |
| 🔁 切任务提醒 | 待办从一项切到下一项时，小人说「老乡，我该做下一步了」 |
| 🙋 需要你处理 | **AI 提问时**（`ask_user_question` / 方案确认）小人抬头 + 气泡；**一轮任务结束**也会轻提醒「这轮忙完啦」 |
| 📝 工时 | 桌面版记录前台软件工时；切换软件时冒「工作中」气泡 3 秒；农场底部状态栏和工时页都能看 |
| 🖥️ harness 自动开工 | 检测到 dsh / opencode / claude / codex / aider 等进程时，小人自动劳动，窗口自动显示 |
| 💬 气泡 | 最大 124px，按农场容器边界自动限宽并夹紧位置，不越界；四方向都有待机/走路/干活动画 |
| 🚶 动作状态机 | 走路方向在起步时按整段位移定一次，不在每帧 dx/dy 之间翻面；到位后完整播完 8 帧干活动作，再显示 1 秒开心表情，最后去下一块地 |
| ⚙️ 自定义 | 自动缩放（默认开，随窗口大小自适应 0.75×–2.5×）、四角停靠、`Alt+Shift+F` 收起/展开、过零点自动翻新一天 |
| 🔤 字体 | ASCII 用素材包 8×10 位图字库，中文继续系统字体；标题使用系统用户名保留大小写 |


## 安装（DSH 端）

标准 DSH bundle，用宿主自带的 `plugin_manager` 安装（不要手改 profile 文件）：

```
plugin_manager  action: install_bundle  target: /home/ubuntu/Rapvebot/stardew-plugin
```

装好后**刷新页面**即可。自检地址：`/dsh-stardew/api/health`。

## 桌面版（Windows）

桌面版复用同一个 `assets/widget.js`，本地起一个只监听 `127.0.0.1` 的 HTTP + SSE 服务，
页面里跑的仍然是这五个页签；区别是按键由 `uiohook-napi` 全局采集，窗口透明、无边框、置顶、可点击穿透。

```powershell
cd desktop
npm install
npm start          # 开发运行（会先同步最新的 widget.js）
npm run pack       # 打包 → desktop/dist/星露谷小农场-win32-x64/星露谷小农场.exe
```

| 项目 | 说明 |
|---|---|
| 数据目录 | `%APPDATA%\dsh-stardew-farm`：`farm.json` / `tasks.json` / `usage.json` / `config.json` / `window-state.json`，Electron 的 localStorage 在 `electron-profile/` |
| 本地端口 | 优先 `17890`，被占用时自动往后找 20 个端口；只监听 `127.0.0.1` |
| 全局按键 | 默认开启；托盘菜单可暂停。物理按键在全局钩子和页面 `keydown` 之间按 45ms 跨来源去重，同一来源连按全部计数 |
| 显示/隐藏 | `Alt+Shift+F`，或单击托盘图标 |
| 窗口尺寸 | 托盘「窗口尺寸」或 ☰ 面板：小 320×520 / 中 400×700 / 大 500×860 / 贴合当前内容 / 自定义宽高；顶栏可拖动窗口，右下角把手可拖拽缩放 |
| 拖动 / 缩放 | 透明窗口不依赖 `-webkit-app-region`：页面把可交互矩形报给主进程，主进程按光标位置切换点击穿透，同时用 70ms 轮询驱动拖动和缩放 |
| 工时记录 | 每 4 秒用 `foreground.ps1` 采样前台窗口，写 `usage.json`；空闲 120 秒或锁屏时暂停；切换软件触发 3 秒「工作中」 |
| harness | 只认已知进程名：`dsh` / `opencode` / `claude` / `codex` / `aider` / `gemini` / `cursor-agent` / `continue` / `cline` / `roo` / `windsurf` / `amp` / `goose` / `plandex` / `kodu` / `crush` 及对应 `-cli` 变体 |
| 任务 | 编辑 `tasks.json`，支持数组或 `{ "todos": [...] }`；`status` 为 `pending` / `in_progress` / `completed`。有 `in_progress` 时小人会自己下田 |
| 余额 | 在 `config.json` 填 `deepseekApiKey` 后重启；不填则余额页显示提示 |
| 存档 | 与 DSH 端同为 `farm.json` v5 格式，但两端各存各的；可手动复制迁移 |
| 托盘 | 显示/隐藏、重新载入、全局按键开关、模拟 AI 干活、置顶、窗口尺寸、记录工时、harness 自动显示、开机启动、打开数据目录/任务/工时/配置、退出 |

> 桌面窗口是透明的：鼠标在农场/面板上时正常接收点击，移开后自动穿透到桌面。

### 打包说明

`npm run pack` 使用 `--no-asar`。asar 包里的 `foreground.ps1` 不能被 `child_process` 直接执行，
不打 asar 后脚本位于 `resources/app/foreground.ps1`，工时采样才能正常工作。
`foreground.ps1` 必须保持 **ASCII-only**：Windows PowerShell 5.1 会把无 BOM 的 UTF-8 脚本按 ANSI 读取，
中文注释曾吞掉下一行，导致 harness 正则变量为空、所有进程都被当成 harness。

### 自检

```powershell
$env:DSF_DATA_DIR = "$env:TEMP\dsf-test\data"
$env:DSF_TEST_DIR = "$env:TEMP\dsf-test\out"
.\desktop\node_modules\electron\dist\electron.exe .\desktop
```

自检会真实操作页面并截图，覆盖：26 块地 A–Z、键盘 10+9+7 不溢出、3 击金币、成熟清地、作物 16×20 踩在泥土贴图内、
农场/键盘同一状态、自动劳动推进作物、动作播完才推进、开心表情 1 秒、走路方向不翻面、工作 5 分钟解锁帮手 / 900 秒清零、
帮手独立干活、任务切换气泡、待办当前项高亮/选中、气泡不越界、软件切换「工作中」、harness 自动显示、工时页、
窗口预设/自定义宽高/拖动/缩放/贴合、余额页、接口字段、打包后 PowerShell 路径、页面错误捕获。
开发态与打包态均为 **86 项**，结果写 `results.json`。

## 素材

全部来自 `PixelSRPG-Forge/_FarmLife.zip` 与 `SunnysideWorld` 素材包，**逐帧核对过**：

| 用途 | 来源 |
|---|---|
| 小人 | FarmLife `spr_basecharacter_idle_strip9`（9 帧 49×34）/ `walk_strip8`（8 帧 49×34）/ `doing_strip8`（8 帧 64×34） |
| 记录员 | FarmLife 紫色角色 `spr_idle_strip9` / `spr_doing_strip8`（96×64 帧） |
| 作物 | FarmLife 11 种真实作物（beetroot / cabbage / cauliflower / carrot / kale / potato / pumpkin / radish / parsnip / sunflower / wheat），每种 6 帧 |
| 土地 | SunnysideWorld 方形泥土 `239,44,50,53` 与圆形泥土 `304,44,48,52`，按地块奇偶交替铺 |
| 环境装饰 | SunnysideWorld 水塘（放第四行左侧空格）+ 顶部花/草；第四行右侧空格站记录员 |
| 帮手 | 4 只 side-view 小动物（cow / goat / pig / sheep），32×32×4，左右水平翻转 |
| UI 图标 | FarmLife 9-slice 面板/木牌/选中框 + SunnysideWorld 图标/表情/进度条 |
| 字体 | IconsPropsMonsters `Text (Black/White) (8x10).png`，10×5 字形，ASCII 位图字体；中文系统字体 |

FarmLife 角色包只有正面动画：下 / 左用正面帧，右向用已经水平镜像的精灵（页面不再翻转），上向用“只把头部内部涂成头发色、保留外轮廓/手部”的干净背面。
农场里**没有房子/谷仓/鸡舍**，只保留土地和装饰；当前共 78 条精灵、11 种作物、4 种帮手动物、49 个字体格。

## 结构 / 重新构建

```
index.js              宿主：脚本路由、会话事件采集（SSE 推送）、存档、余额、agents 忙闲兜底
cordis.patch.yml      挂载声明
assets/
  widget.template.js  页面脚本源码（占位符由构建替换）
  widget.css          样式（农场 / 键盘 / 工时 / 面板 / 桌面透明窗口）
  extract-farmlife.mjs 从 FarmLife / SunnysideWorld 素材包裁出「精灵条」→ assets/sprites/
  sprites/            78 条精灵条 PNG + index.json（单帧宽/高/帧数）
  build.mjs           template + css + sprites → widget.js（全部内联）
  smoke.mjs           假 DOM 冒烟测试（57 项）
  widget.js           构建产物（宿主实际发给浏览器的文件）
desktop/
  main.js             Electron 主进程：本地 HTTP/SSE、托盘、全局按键、工时、窗口尺寸、自检
  preload.js          最小 IPC 桥（全局按键 / 命中矩形 / 拖动 / 缩放 / 窗口尺寸）
  foreground.ps1      前台窗口 / 空闲 / harness 采样（ASCII-only）
  usage.js            工时累计、软件切换事件、harness 过滤
  index.html          桌面端入口（标记 DESKTOP + GLOBAL_KEYS）
  sync-widget.mjs     把 assets/widget.js + icon.png 同步到桌面端
  package.json        Electron / uiohook-napi / packager（`--no-asar`）
docs/                 截图
locale/               插件管理器里的中英文标题
```

改了页面脚本或样式后：

```bash
node assets/build.mjs     # 重新生成 widget.js（宿主按 mtime 热读，不用重启）
node assets/smoke.mjs     # 57 项冒烟测试
node assets/extract-farmlife.mjs # 只有换素材包时才需要，重新裁精灵条
```

桌面端改了 `main.js` / `preload.js` / `index.html` 后重新 `npm start`；
改了共享组件后先 `node assets/build.mjs`，再 `npm start`（会自动 sync）。

## 宿主接口

| 路由 | 用途 |
|---|---|
| `GET /dsh-stardew/widget.js` | 自包含页面脚本（CSS + 78 条精灵图已内联成 data URI） |
| `GET /dsh-stardew/api/events` | **SSE 推送**：待办 / AI 忙闲 / 提问 / 软件切换，事件发生即推送 |
| `GET /dsh-stardew/api/state` | 同上的快照（SSE 不可用时兜底轮询）；桌面端额外带 `source` / `harness` / `usage` / `workApp` |
| `GET /dsh-stardew/api/snapshot` | 农场存档（金币 / 26 块地 / 键位成长） |
| `POST /dsh-stardew/api/mutate` | 写存档 → `$DSH_HOME/plugin-data/dsh-stardew-farm/farm.json`（桌面端写 `%APPDATA%`） |
| `GET /dsh-stardew/api/usage` | 今日软件工时（仅桌面端有数据） |
| `GET /dsh-stardew/api/balance` | 余额 |
| `GET /dsh-stardew/api/health` | 自检（版本 / 模式 / 脚本大小 / 是否自包含 / SSE 连接数；桌面端另报全局按键、窗口尺寸、工时文件、harness） |

## 设计取舍

- **不声明 `dsh.client`**：页面脚本走 index.html 注入，不参与 client-modules 组合。
  （声明了却无 `./client` 会让宿主启动失败。）
- **一个脚本路由**：样式与精灵图全部内联，杜绝素材 404。
- **精灵按「条」裁**：每条相等帧宽，页面只做 `background-position: -frame*宽度`，
  不会有二维坐标表算错导致的人物/牲畜错位。
- **待办/提问由宿主采集 + SSE 推送**：页面不猜前端 store，也不靠轮询，延迟接近 0。
- **启动水合**：先读 `session_projcache` 的 todos 投影，再用 `sessionQuery.readSession` 找最后一条 `todo/write`；启动前写过的待办不会丢。
- **v5 存档**：26 块地固定 A–Z；旧 v4 存档只迁移能落到字母的地，金币保留。
- **收割即时结算**：第 3 次按键立刻加金币，成熟动画单独计时清地；不再用 `harvesting` 队列门闩。
- **同位移动立即回调**：`walkFarmerTo` 目标等于当前坐标时直接 `onArrive`，避免小人卡住。
- **帮手独立状态机**：`helperSeconds` 每 5 分钟 +1 只（上限 4），每只自己选地 / 走路 / 播完动作 / 推进作物；不是跟着小人走。`lastWorkAt` 超过 900 秒才清零。
- **11 种真实作物**：每种 6 帧 FarmLife 精灵，3 次成熟映射到帧 0 / 2 / 4 / 5，不再用 CSS filter 假装不同作物。
- **方向只定一次**：走路方向在 `startJob` 时按整段位移确定，移动过程中不再按单帧 dx/dy 反复翻面；到达后完整播完 8 帧动作，再显示 1 秒开心表情。
- **只有提问才打扰**：授权、报错、工具结果都不再让小人抬头。
