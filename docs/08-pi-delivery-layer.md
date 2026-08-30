# Pi 的交付层：TUI 渲染、包分发与配置治理

> 系列第 08 篇。前面七篇讲的都是"Agent 怎么想、怎么做、怎么存"，这一篇讲最后一公里：结果怎么呈现给用户（`pi-tui`）、能力怎么分发给用户（Pi Packages）、配置怎么组织（Settings）、以及会话怎么被远程访问（协议三件套）。这些是"能用"和"好用"之间的差距所在。

## 目录

1. [pi-tui：为什么要自己写一个 TUI 框架](#1-pi-tui为什么要自己写一个-tui-框架)
2. [差量渲染与同步输出](#2-差量渲染与同步输出)
3. [两种渲染器：主屏 vs 备用屏](#3-两种渲染器主屏-vs-备用屏)
4. [Focusable 与 IME：中文输入的真实成本](#4-focusable-与-ime中文输入的真实成本)
5. [Pi Packages：能力的分发单元](#5-pi-packages能力的分发单元)
6. [依赖规则：peerDependencies 为何必须是 "*"](#6-依赖规则peerdependencies-为何必须是-)
7. [配置治理：分层设置与 pi config](#7-配置治理分层设置与-pi-config)
8. [协议三件套：会话的远程化](#8-协议三件套会话的远程化)
9. [设计小结](#9-设计小结)

---

## 1. pi-tui：为什么要自己写一个 TUI 框架

Node 生态有 Ink（React for CLI）、blessed 等成熟方案，Pi 仍然自己写了 `pi-tui`。从它的特性列表能看出原因——需求集中在"流式输出 + 无闪烁 + 终端能力适配"这三个点上：

```
- Interchangeable Renderers    主屏/备用屏两种实现共享 TUI 接口
- Differential Rendering       只更新变化的行或视口行
- Application-owned Scrolling  备用屏视口支持鼠标、触控板、键盘导航
- Synchronized Output          CSI 2026 原子屏幕更新（无闪烁）
- Bracketed Paste Mode         正确处理大段粘贴（>10 行加标记）
- Component-based              简单的 Component 接口
- Theme Support                组件接受 theme 接口
- Inline Images                Kitty / iTerm2 图形协议
- Autocomplete Support         文件路径与斜杠命令
```

一个 Coding Agent 的 UI 有个特殊压力：**LLM 流式输出意味着每秒可能有几十次内容更新，且更新位置在文档中间（正在生成的那条消息）**。通用 TUI 框架的重绘策略在这个负载下容易闪烁或卡顿。`pi-tui` 的整个设计都围绕这一点。

### 1.1 Component 接口的极简契约

```typescript
interface Component {
  render(width: number): string[];
  handleInput?(data: string): void;
  wantsKeyRelease?: boolean;
  invalidate(): void;
}
```

只有一个必需方法 `render(width)` 返回字符串数组（一行一个元素）。约束是硬的：**每行不得超过 `width`，超过 TUI 会直接报错**。

这个"宁可报错也不静默截断"的选择是对的——一行超宽会导致终端自动换行，进而让 TUI 维护的"第 N 行在屏幕第 N 行"的映射错位，后续所有差量更新都会画到错误位置。这种错误如果静默发生，表现是界面逐渐错乱、极难定位。所以框架强制组件作者用 `truncateToWidth()` 或 `wrapTextWithAnsi()` 自己处理。

### 1.2 ANSI 样式不跨行

> The TUI appends a full SGR reset and OSC 8 reset at the end of each rendered line. Styles do not carry across lines.

每行末尾自动加完整的 SGR 重置和 OSC 8（超链接）重置。原因是差量渲染只重画变化的行——如果样式跨行延续，重画第 5 行时第 4 行留下的颜色状态就不存在了，会出现颜色错乱。代价是组件作者需要为每行重新应用样式，或者用 `wrapTextWithAnsi()`（它会为每个折行保留样式）。

**这是差量渲染的必然代价**：要么放弃跨行样式，要么放弃差量渲染。Pi 选了前者，并提供工具函数降低成本。

---

## 2. 差量渲染与同步输出

### 2.1 主屏的三种渲染策略

```
1. First Render                        输出全部行，不清除 scrollback
2. Width Changed 或 变化在视口上方      清屏并完整重绘
3. Normal Update                       光标移到第一个变化行，清到末尾，重绘变化行
```

策略 3 是常态路径，也是流式输出流畅的关键：一条正在生成的 assistant 消息更新时，只有那条消息所在的行会被重绘，上面的历史内容完全不动。

策略 2 的两个触发条件都是"无法局部更新"的情况：终端宽度变了（所有行的折行位置都变）、或变化发生在视口上方（光标无法向上移动到 scrollback 区域）。

### 2.2 同步输出：CSI 2026

```
\x1b[?2026h  ...更新...  \x1b[?2026l
```

两个渲染器都把更新包在同步输出序列里。终端收到 `2026h` 后暂停渲染，直到 `2026l` 才一次性把缓冲的所有变更绘制出来。**没有这个机制，"清到末尾 + 重绘"这两步之间用户会看到短暂的空白**，高频更新时表现为闪烁。

这是一个典型的"用终端的现代能力换体验"——不支持 CSI 2026 的老终端会忽略这两个序列，行为退化为可能闪烁但功能正常。

### 2.3 组件级缓存

框架文档明确建议组件缓存渲染结果：

```typescript
class CachedComponent implements Component {
  private cachedWidth?: number;
  private cachedLines?: string[];

  render(width: number): string[] {
    if (this.cachedLines && this.cachedWidth === width) return this.cachedLines;
    const lines = [truncateToWidth(this.text, width)];
    this.cachedWidth = width;
    this.cachedLines = lines;
    return lines;
  }

  invalidate(): void {
    this.cachedWidth = undefined;
    this.cachedLines = undefined;
  }
}
```

差量渲染要判断"哪些行变了"就必须调用所有组件的 `render()`。一个有几百条消息的会话，每帧都要 render 几百个组件——Markdown 组件的解析和高亮是有成本的。缓存 + `invalidate()`（主题切换时调用）是让这个模型可扩展的必要条件。

这也解释了 `invalidate()` 为什么在 `Component` 接口里是**必需**方法而 `handleInput` 是可选的：框架需要一个统一的方式让所有组件放弃缓存。

---

## 3. 两种渲染器：主屏 vs 备用屏

`TUI` 是共享接口，两个实现对应两种终端使用哲学：

| | `TuiMainScreen`（默认） | `TuiAltScreen`（实验性 fullscreen） |
|---|---|---|
| 缓冲区 | 主终端缓冲区 | 备用缓冲区 |
| Scrollback | 保留终端原生 scrollback | 应用自己拥有视口和滚动 |
| 布局 | 无固定高度布局 | 支持 `VStack`/`HStack`/`ScrollView` 显式布局 |
| 退出后 | 内容留在终端历史里 | 恢复主缓冲区，打印完整最终文档 |

对应设置项 `tuiMode: "regular" | "fullscreen"`，可用 `/settings` 即时切换或 `--tui-mode` 启动时指定。

### 3.1 为什么保留主屏模式作为默认

主屏模式的价值是"Pi 的输出就是终端历史的一部分"——退出后可以用终端自己的搜索/滚动/复制去看之前的对话，可以被 `tee` 到文件，可以和其他命令的输出混在一起阅读。这符合 Unix 工具的行为惯例。

备用屏模式（fullscreen）提供更强的 UI 能力（固定底部编辑器、独立滚动的对话区、`Ctrl+Shift+F` 搜索），代价是接管了滚动和历史。它作为实验性选项而非默认，是一个明确的取舍表态：**默认不打破终端惯例**。

### 3.2 备用屏的布局能力

```typescript
if (isViewportTUI(tui)) {
  tui.setLayoutRoot(new VStack([
    { component: new ScrollView(transcript, { follow: "end", primary: true, overscroll: "chain" }),
      basis: 0, grow: 1, minSize: 1 },
    { component: editorAndFooter, basis: "auto", shrink: 1, minSize: 1 },
  ]));
}
```

`basis`/`grow`/`shrink`/`minSize`/`maxSize` 是 flexbox 语义的子集。`isViewportTUI(tui)` 类型守卫的存在说明这些能力**只在备用屏可用**——文档明确写了 "These semantics are intentionally unavailable on `TuiMainScreen`, where the terminal owns scrollback"。

不是"暂未实现"而是"故意不提供"：主屏模式下终端拥有 scrollback，应用去做固定高度布局会和终端的滚动行为冲突。这是把"两种模式的能力差异"如实暴露在类型系统里，而不是提供一个在某个模式下静默失效的 API。

`follow: "end"` 对应流式输出的关键行为：在底部时自动跟随新内容，手动滚上去后保持位置不被新内容拽走。

### 3.3 图形协议的诚实降级

> `TuiAltScreen` supports inline images and partial viewport cropping in terminals that implement the Kitty graphics protocol, including Kitty and Ghostty. iTerm2's inline-image protocol does not provide operations to delete an existing placement or crop its source while scrolling. To prevent stale images from remaining over repainted content, `TuiAltScreen` renders image components as text placeholders in iTerm2.

Kitty 协议支持删除和裁剪已放置的图片，iTerm2 协议不支持。备用屏需要在滚动时裁剪图片，所以在 iTerm2 下**主动降级为文本占位符**——宁可不显示图片，也不能留下遮盖内容的残影。主屏模式下不需要裁剪，iTerm2 图片正常显示。

这是第 06 篇 `compat` 那种"能力探测而非假设"思路在 UI 层的同一体现：不同终端能力不同，按实际能力决定行为，不支持时明确降级。

---

## 4. Focusable 与 IME：中文输入的真实成本

这一节对中文用户尤其重要，也是一个"细节决定可用性"的典型案例。

Pi 渲染的是**假光标**（用反显字符模拟），真实硬件光标默认隐藏——这样光标位置完全由应用控制，不受终端光标行为干扰。但 IME（中文/日文/韩文输入法）的候选词窗口是**跟着硬件光标位置**弹出的。如果硬件光标不在正确位置，输入中文时候选窗会出现在屏幕的错误位置。

解法是 `Focusable` 接口：

```typescript
class MyInput implements Component, Focusable {
  focused: boolean = false;   // TUI 在焦点变化时设置

  render(width: number): string[] {
    const marker = this.focused ? CURSOR_MARKER : "";
    return [`> ${beforeCursor}${marker}\x1b[7m${atCursor}\x1b[27m${afterCursor}`];
  }
}
```

TUI 的处理流程：

1. 设置组件的 `focused = true`
2. 扫描渲染输出里的 `CURSOR_MARKER`（一个零宽 APC 转义序列）
3. 把硬件光标定位到那个位置
4. 只在 `showHardwareCursor` 开启时显示硬件光标

**默认硬件光标仍然隐藏**——假光标继续渲染，硬件光标只是被移到正确位置。对于那些"用隐藏光标位置追踪 IME 候选窗"的终端，这样就够了。少数终端要求光标可见才定位 IME，用 `showHardwareCursor` / `setShowHardwareCursor(true)` / `PI_HARDWARE_CURSOR=1` 开启。

### 4.1 容器必须传播焦点

```typescript
class SearchDialog extends Container implements Focusable {
  private searchInput: Input;
  private _focused = false;
  get focused(): boolean { return this._focused; }
  set focused(value: boolean) {
    this._focused = value;
    this.searchInput.focused = value;   // 必须传播给子组件
  }
}
```

文档的警告很直接："Without this propagation, typing with an IME (Chinese, Japanese, Korean, etc.) will show the candidate window in the wrong position."

**任何包含 `Input`/`Editor` 子组件的容器（对话框、选择器）都必须实现 `Focusable` 并把焦点状态传下去**。内置的 `Editor` 和 `Input` 已经实现了这个接口，写自定义组件（第 02 篇提到的 `ctx.ui.custom()`）时需要自己注意。

这是一个很好的例子说明"支持 CJK 输入"不是加个字体那么简单——它牵扯到假光标 vs 硬件光标、焦点传播、终端能力差异三层机制。

---

## 5. Pi Packages：能力的分发单元

第 02 篇讲了扩展/技能/提示词模板三种扩展机制，Pi Packages 是把它们打包分发的载体。

### 5.1 三种源

```bash
pi install npm:@foo/bar@1.0.0
pi install git:github.com/user/repo@v1
pi install https://github.com/user/repo
pi install /absolute/path/to/package
pi install ./relative/path/to/package
```

| 源 | 安装位置（全局 / 项目） | 版本策略 |
|---|---|---|
| npm | `~/.pi/agent/npm/` / `.pi/npm/` | 带版本的 spec 被 pin 住，`pi update --extensions` 跳过 |
| git | `~/.pi/agent/git/<host>/<path>` / `.pi/git/...` | ref 是 pin 住的 tag/commit，更新只做"对齐到配置的 ref"，不会自动升到更新的 ref |
| 本地路径 | 不复制，直接写入 settings | 无 |

git 源的设计值得注意：`pi update --extensions` **不会把 pin 的 ref 移到更新的版本**，只会把本地 clone 对齐到配置里写的那个 ref。要升级必须显式 `pi install git:host/user/repo@new-ref`。这是"依赖变更必须是显式决定"的安全取向——和项目 AGENTS.md 里"npm 依赖与 lockfile 变更视为需要审查的代码"是同一立场。

CI 场景的两个实用变量：`GIT_TERMINAL_PROMPT=0` 禁用凭证提示、`GIT_SSH_COMMAND="ssh -o BatchMode=yes -o ConnectTimeout=5"` 快速失败——避免无人值守时卡在凭证输入上。

### 5.2 临时试用

```bash
pi -e npm:@foo/bar        # 安装到临时目录，仅本次运行有效
pi -e git:github.com/user/repo
```

装到临时目录、只对本次运行生效。这个入口的存在降低了"试一个第三方包"的心理成本——不需要担心污染自己的配置。

### 5.3 声明方式：manifest 或约定目录

```json
{
  "name": "my-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "themes": ["./themes"]
  }
}
```

没有 `pi` manifest 时按约定目录自动发现：`extensions/`（`.ts`/`.js`）、`skills/`（递归找 `SKILL.md` 目录 + 顶层 `.md`）、`prompts/`（`.md`）、`themes/`（`.json`）。

manifest 的路径数组支持 glob 和 `!排除`，但有两条限制值得注意：

> Positive manifest globs discover visible paths in lexical order. List dot-prefixed paths directly. If a glob would need to continue through a symlink, list the symlinked resource root directly.

**glob 不穿透 dot 目录和符号链接**——需要的话必须直接列出路径。这是防止 glob 意外递归进 `.git`、`node_modules` 或通过 symlink 逃出包目录的保守设计。

### 5.4 安全警示的位置

文档在"Install and Manage"章节开头就放了警示：

> **Security:** Pi packages run with full system access. Extensions execute arbitrary code, and skills can instruct the model to perform any action including running executables. Review source code before installing third-party packages.

这与第 01 篇讲的安全模型一致：Pi 不提供沙箱，装一个包等于运行任意代码。**警示放在"怎么安装"的正上方而不是文档末尾的安全章节**——在用户即将执行 `pi install` 的那一刻提醒，而不是指望他们先读完安全文档。

### 5.5 过滤与去重

```json
{
  "packages": [
    "npm:simple-pkg",
    {
      "source": "npm:my-package",
      "extensions": ["extensions/*.ts", "!extensions/legacy.ts"],
      "skills": [],
      "prompts": ["prompts/review.md"],
      "themes": ["+themes/legacy.json"]
    }
  ]
}
```

规则：省略某个 key = 全部加载；`[]` = 一个都不加载；`!pattern` 排除；`+path`/`-path` 强制包含/排除精确路径。关键约束是"Filters layer on top of the manifest. They narrow down what is already allowed."——**过滤只能收窄，不能扩大**。包作者没在 manifest 里声明的资源，用户无法通过过滤把它加载进来。

同一个包在全局和项目 settings 都出现时的去重规则：项目条目胜出，除非项目条目有 `autoload: false`（此时作为全局条目的增量应用）。身份判定：npm 看包名、git 看不含 ref 的仓库 URL、本地看解析后的绝对路径。

---

## 6. 依赖规则：peerDependencies 为何必须是 "*"

这是包作者最容易踩的坑，规则很具体：

> Pi bundles core packages for extensions and skills. If you import any of these, list them in `peerDependencies` with a `"*"` range and do not bundle them: `@earendil-works/pi-ai`, `@earendil-works/pi-agent-core`, `@earendil-works/pi-coding-agent`, `@earendil-works/pi-tui`, `typebox`.

**为什么必须是 `"*"` 而不是 `"^1.2.0"`**：这五个包由用户安装的 Pi 提供，不是由你的包提供。如果声明了具体版本范围，用户升级 Pi 后版本不匹配，你的包就会安装失败或报警告——而实际上 Pi 加载扩展时用的就是自己那份，你声明的范围毫无约束力，只会制造假的冲突。

**为什么不能 bundle**：如果把 `pi-ai` 打进自己的包，运行时会存在两份 `pi-ai`,类型看起来一样但是不同的模块实例——`instanceof` 检查失败、单例状态不共享，产生极难排查的问题。

**其他 pi 包必须 bundle**（规则正好相反）：

```json
{
  "dependencies": { "shitty-extensions": "^1.0.1" },
  "bundledDependencies": ["shitty-extensions"],
  "pi": {
    "extensions": ["extensions", "node_modules/shitty-extensions/extensions"],
    "skills": ["skills", "node_modules/shitty-extensions/skills"]
  }
}
```

原因是 "Pi loads packages with separate module roots, so separate installs do not collide or share modules"——Pi 用独立 module root 加载每个包，各自的 `node_modules` 互不干扰，所以依赖别人的 pi 包时把它 bundle 进来是安全的，也是唯一可行的方式（因为 Pi 不会去解析你声明的 pi 包依赖并单独安装它）。

规则总结：

| 依赖类型 | 声明位置 | Bundle |
|---|---|---|
| Pi 核心 5 个包 | `peerDependencies: "*"` | 否 |
| 其他 pi 包 | `dependencies` + `bundledDependencies` | 是 |
| 普通第三方运行时依赖 | `dependencies` | 否（Pi 安装时会跑 `npm install`） |

---

## 7. 配置治理：分层设置与 pi config

### 7.1 两层设置

| 位置 | 作用域 |
|---|---|
| `~/.pi/agent/settings.json` | 全局（所有项目） |
| `.pi/settings.json` | 项目（当前目录） |

项目设置覆盖全局设置。少数设置**只允许全局**（`defaultProjectTrust`、`httpProxy`）——这类设置如果允许项目级覆盖，等于让仓库自己决定"我该被信任吗"或"你的流量走哪",是明显的安全漏洞。这个限制不是遗漏，是刻意的。

### 7.2 项目信任在非交互模式下的行为

第 01 篇提过项目信任，这里补充非交互模式的具体规则：

> Non-interactive modes (`-p`, `--mode json`, and `--mode rpc`) do not show a trust prompt. Without an applicable saved trust decision, they use `defaultProjectTrust` from global settings: `ask` (default) and `never` ignore those project resources, while `always` trusts them. Pass `--approve`/`-a` or `--no-approve`/`-na` to override project trust for one run.

关键点：**非交互模式下 `ask`（默认值）的行为是"忽略项目资源"而不是"卡住等输入"**。这是正确的默认——CI 里不该因为一个信任提示挂死，也不该默认信任任意仓库的扩展。要在 CI 里用项目扩展，显式传 `--approve`。

`pi update` 是唯一"从不提示"的命令——更新自己不应该受项目配置影响。

`/trust` 命令写入 `~/.pi/agent/trust.json`，并明确说明"the current session is not reloaded, so restart pi for changes to take effect"——不假装热生效。

### 7.3 值得注意的几个设置

| 设置 | 默认 | 说明 |
|---|---|---|
| `showCacheMissNotices` | `false` | 第 05 篇讲的缓存浪费提示，**默认关闭** |
| `defaultThinkingLevel` | （用 `medium`） | 第 05 篇讲的成本控制项 |
| `thinkingBudgets` | - | 自定义各思考等级的 token 预算 |
| `doubleEscapeAction` | `"tree"` | 双击 Esc 的行为：`tree`/`fork`/`none` |
| `treeFilterMode` | `"default"` | `/tree` 默认过滤：`default`/`no-tools`/`user-only`/`labeled-only`/`all` |
| `enableInstallTelemetry` | `true` | 匿名安装/更新版本 ping |
| `enableAnalytics` | `false` | 分析数据共享，**opt-in** |
| `httpProxy` | - | 设为 `HTTP_PROXY`/`HTTPS_PROXY`，仅全局 |

`showCacheMissNotices` 默认 `false` 值得单独说：第 05 篇分析了这个提示有多精细（1024 噪声地板、双阈值、只陈述可观测归因），但它**默认仍然关闭**。这符合"默认界面尽量安静，成本诊断是主动开启的工具"的取向——不是每个用户都需要持续盯着缓存命中情况。

遥测的两个开关分得很清楚：`enableInstallTelemetry`（默认开，只是版本 ping）和 `enableAnalytics`（默认关，opt-in）。文档还特意说明"Opting out of telemetry does not disable update checks"——退出遥测和禁用更新检查是两件事,后者用 `PI_SKIP_VERSION_CHECK=1`，全部网络操作用 `--offline`/`PI_OFFLINE=1`。**把三个容易被混为一谈的开关分开并说明差异**，避免用户以为关了遥测就不联网了。

### 7.4 pi config：资源的开关面板

```bash
pi config       # 从全局设置开始，Tab 切换全局/项目
pi config -l    # 从项目覆盖开始，继承的全局资源显示为暗色
```

用于启用/禁用已安装包和本地目录里的扩展、技能、模板、主题。`-l` 模式下"继承的全局资源显示为暗色"是一个信息设计上的细节——让用户能区分"这是我在项目里设的"和"这是从全局继承来的"，避免在项目层重复配置。

---

## 8. 协议三件套：会话的远程化

`pi-protocol` / `pi-server` / `pi-client` 支撑"本地 CLI"之外的远程会话形态。

### 8.1 线格式

```
[4 字节无符号大端载荷长度][一个确定长度的 CBOR item]
```

第一条客户端消息永远是 `hello`，携带 `PROTOCOL_VERSION`。之后是相关联的 request/response 信封和服务端事件信封。

选 CBOR 而非 JSON 的理由能从它的严格子集限制看出来——协议要的是**确定性编码**：

```
支持：null/布尔、有限数字（整数限制在 JS 安全范围，非整数编码为 float64）、
      UTF-8 字符串、Uint8Array 字节串、确定长度数组、确定长度 map（对象，键唯一）

拒绝：顶层 undefined、undefined 数组元素、稀疏数组、非有限或不安全数字、
      tag、不确定长度 item、畸形 UTF-8、尾随数据、过深嵌套、超大值
```

默认限制：每个 CBOR 载荷/帧 16 MiB、数组元素或 map 条目 1,000,000、嵌套 64 层。所有 schema 拒绝未知对象属性。

拒绝"不确定长度 item"这一条尤其关键——不确定长度的 CBOR 数组要读到结束标记才知道多长，这让"先校验长度再缓冲"的防御策略失效。文档明确说了"A frame decoder validates the declared length before buffering payload bytes"：**先看声明的长度是否合法，再决定要不要分配缓冲区**，这是防御恶意超大帧的标准做法。

### 8.2 权威状态 vs 瞬时提示

> Session and server snapshots are authoritative. Progress events are transient UI hints and must not be reduced into authoritative state.

这是分布式状态同步的核心约定：**Snapshot 是真相，progress 事件只是 UI 提示，不能被归约进权威状态**。

为什么必须这样：progress 事件可能丢失、重排、重复（网络层不保证），如果客户端把它们累积成自己的状态副本，就会和服务端逐渐分歧且无法自愈。只允许 snapshot 作为状态来源，客户端任何时候都能通过重新获取 snapshot 回到正确状态。

这也和第 01 篇提到的"底层 agentLoop 事件流是观察性的"是同一思路——事件用于观察，状态另有权威来源。

### 8.3 SessionMetadata：不获取运行时也能列会话

> Session lists contain `SessionMetadata`, the normalized durable metadata available without acquiring a session runtime. Only `id` and `createdAt` are required; `updatedAt`, `parentSessionId`, `sessionName`, and `cwd` are included when supported by the backing store. Runtime state such as phase, model, thinking level, attachment, and locking appears only in an acquired `SessionSnapshot`.

`SessionMetadata` 与 `SessionSnapshot` 的分离对应一个实际需求：**列出会话列表不应该需要为每个会话启动一个运行时**。列表只需要持久化的元数据（id、创建时间、名字、cwd），而"当前用哪个模型、是否被锁定、处于什么阶段"这些运行时状态只在真正 acquire 一个会话后才有。

只有 `id` 和 `createdAt` 是必需的——其余字段"当后端存储支持时才包含"，这让不同能力的 session backend（如 `packages/session-backends/sqlite-node`）都能实现这个接口。

### 8.4 传输层与鉴权的分离

> Transports complete authentication before protocol bytes are exchanged.
> All transports are untrusted. Configure matching frame limits and enforce access controls appropriate for the transport before exposing a connection to the protocol. Unix sockets can use filesystem permissions, while network transports can authenticate during connection establishment.

**鉴权在协议字节交换之前完成，协议本身不管鉴权**。这个分层让 Unix socket（用文件系统权限）和网络传输（连接时鉴权）用同一套协议——协议层假设"能连上来的都已经鉴权过了",但同时明确要求"所有传输都是不可信的"，必须配置匹配的帧限制和访问控制。

`pi-protocol` 本身**不捆绑任何传输实现**，使用方提供 `ByteTransport`（保序、能报告流关闭）。自定义传输必须处理任意的帧分片与合并——这是文档反复强调的点，因为"一次 read 拿到一个完整帧"是最常见的错误假设。

### 8.5 实验性状态

> The protocol is experimental and has no compatibility guarantees.

和第 04 篇提到的 `AgentHarness` 一样，这里也如实标注了成熟度。构建生产系统时应该知道这一层随时可能变。

---

## 9. 设计小结

1. **自己写 TUI 是因为流式输出的负载特殊**。差量渲染 + CSI 2026 同步输出 + 组件级缓存，三者共同解决"每秒几十次中间位置更新且不能闪烁"这个通用框架不擅长的问题。代价是"样式不跨行"和"render 不得超宽"两条硬约束。

2. **能力差异如实暴露在类型和行为里**：`isViewportTUI()` 类型守卫区分两种渲染器的布局能力、iTerm2 图片主动降级为占位符、`ctx.ui.custom()` 在 RPC 模式返回 `undefined`。这与第 06 篇的 `compat` 是同一思路——不假装所有环境能力一致。

3. **中文输入的支持成本被认真对待**：假光标 + `CURSOR_MARKER` + 硬件光标定位 + 容器焦点传播四层机制，只为让 IME 候选窗出现在正确位置。这类"不做也能跑，做了才好用"的工作是可用性的真实构成。

4. **分发规则源自加载机制**：`peerDependencies: "*"` + 不 bundle 核心包（避免双实例）、其他 pi 包必须 bundle（因为独立 module root），这两条看似矛盾的规则都是从"Pi 怎么加载包"直接推导出来的，不是任意约定。

5. **配置的层级限制是安全边界**：`defaultProjectTrust` 和 `httpProxy` 只允许全局，因为让仓库自己决定是否被信任、或流量走哪，是明显的漏洞。**什么可以被项目覆盖、什么不可以，是一条安全设计决策而非配置系统的实现细节。**

6. **协议层坚持"snapshot 权威、事件瞬时"**，让客户端状态任何时候都能通过重新获取 snapshot 自愈；`SessionMetadata`/`SessionSnapshot` 的分离让"列出会话"不需要启动运行时。两者都是分布式状态同步的标准做法，在这里被明确写进了协议约定而不是留给实现者自己领悟。
