# MC 更新器 — 客户端文档

> **技术栈：** Java / Kotlin  
> **平台：** Windows / macOS / Linux / Android（FCL 启动器）
>
> 客户端是更新器（`updater-*.jar`），由引导器（`launcher-agent.jar`）通过 ClassLoader 加载执行。更新器在 Minecraft 启动**之前**完成所有更新操作。
> 
> 引导器文档详见：`../引导器/README.md`

---

## 一、目录结构

```
.minecraft/update/（更新器所在目录）
├── launcher-agent.jar             # 引导器（-javaagent 加载，详见 引导器/引导器.md）
├── updater-1.0.0.jar              # 更新器本体（由引导器扫描选择最新版本）
├── config.json                    # 服务端地址 + 行为配置
└── .updater/                      # 更新器内部数据（隐藏目录）
    ├── local-version.json         # 上次拉取的版本号
    ├── local-manifest.json        # 上次拉取的完整清单
    ├── local-changelog.json       # 上次拉取的更新日志（用于展示）
    └── local-game-profile.json    # 上次拉取的游戏版本/加载器配置（可选）
```

---

## 二、客户端配置文件

### 2.1 `config.json` — 服务端地址配置

```json
{
  "servers": [
    "http://main-server.com:8080",
    "http://backup1.example.com:8080",
    "http://backup2.example.com:8080"
  ],
  "retryCount": 3,
  "downloadTimeout": 30,
  "parallelDownloads": 8,
  "showChangelog": true,
  "changelogDuration": 15,
  "autoCloseChangelog": true,
  "gameMainClass": "net.minecraft.client.main.Main"
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `servers` | string[] | 服务端地址列表（结尾不带 `/`），按顺序优先使用，第一个不可达时自动切换到下一个 |
| `retryCount` | number | 每个地址的下载失败重试次数 |
| `downloadTimeout` | number | 单个文件下载超时（秒） |
| `parallelDownloads` | number | 同时下载文件数 |
| `showChangelog` | boolean | 更新完成后是否弹出更新日志窗口 |
| `changelogDuration` | number | 更新日志窗口自动关闭时间（秒），0 = 不自动关闭 |
| `autoCloseChangelog` | boolean | 游戏窗口获得焦点后是否自动关闭日志窗口 |
| `gameMainClass` | string | Minecraft 主类全限定名，Agent 会调用其 `main()` 启动游戏 |
| `gameDir` | string | 手动指定 `.minecraft` 目录路径（可选），不填则自动向上查找 |

> **多地址回退逻辑：** 请求 `version.json` / `manifest.json` 时，依次尝试 `servers[0]` → `servers[1]` → ...，第一个成功响应的地址作为本次更新的基准地址，后续文件下载也使用该地址。

---

### 2.2 `.updater/local-version.json` — 本地版本记录

```json
{
  "version": "a7f3e1b2-c9d0-4e8f-b1a6-3f7e9c2d5a0b",
  "updatedAt": "2026-09-12T10:30:00Z"
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `version` | string | 上次从服务端拉取的随机版本号 |
| `updatedAt` | string | 上次更新完成的时间（ISO 8601） |

> 客户端启动时，先请求服务端的 `version.json`，与本地 `local-version.json` 中的 `version` 对比：
> - **一致** → 无需更新，直接启动游戏
> - **不一致** → 拉取 `manifest.json` 进行更新

---

### 2.3 `.updater/local-manifest.json` — 本地清单缓存

```json
{
  "files": [
    {
      "path": "1/mods/Almanac-1.21.1-2-neoforge-1.5.2.jar",
      "sha1": "d613dbc38ca5bcc718408db432385c2e67ac763f",
      "fileSize": 25593,
      "action": "check_and_overwrite",
      "downloads": [
        "https://mediafilez.forgecdn.net/files/7489/91/Almanac-1.21.1-2-neoforge-1.5.2.jar"
      ]
    }
  ]
}
```

> 与服务端 `manifest.json` 结构完全一致。客户端拉取新清单后写入此文件，用于：
> - **文件追踪**：知道本地哪些文件是通过更新器管理的
> - **增量判断**：对比新旧清单，只处理有变化的文件
> - **回滚参考**：如果需要，可基于旧清单恢复文件

---

### 2.4 `.updater/local-changelog.json` — 本地更新日志缓存

```json
{
  "currentVersion": "1.0.3",
  "history": [
    {
      "version": "1.0.3",
      "date": "2026-09-12",
      "content": "新增了 XX 模组\n修复了 YY 的崩溃问题"
    },
    {
      "version": "1.0.2",
      "date": "2026-09-01",
      "content": "更新了 ZZ 模组版本"
    }
  ]
}
```

> 从服务端 `changelog.json` 拉取后缓存到本地，用于更新完成后弹窗展示。
> 展示规则：第一条直接展示，历史版本默认折叠点击展开。

---

## 三、启动方式：引导器 + 更新器

### 3.1 架构

```
JVM 启动参数: -javaagent:update/launcher-agent.jar
                        │
                        ▼
              ┌─────────────────┐
              │   引导器 (Agent) │
              │  扫描同目录下    │
              │  updater-*.jar   │
              │  选版本最大的    │
              └────────┬────────┘
                       │ ClassLoader 加载
                       ▼
              ┌─────────────────┐
              │  更新器 (Agent)  │
              │  premain() 执行  │
              │  检查更新/下载   │
              │  展示日志        │
              └────────┬────────┘
                       │ 返回
                       ▼
              ┌─────────────────┐
              │  Minecraft 启动  │
              └─────────────────┘
```

### 3.2 更新器自身如何更新

更新器**不主动检查自身更新**。新版本更新器由服务端作为普通文件下发：

```
服务端 data/0/updater-1.1.0.jar    ← 服务端管理员放入新版本
         │
         ▼ 扫描进 manifest.json
{
  "path": "0/updater-1.1.0.jar",
  "sha1": "...",
  "fileSize": 123456,
  "action": "check_and_overwrite"
}
         │
         ▼ 客户端正常更新流程下载
.minecraft/update/updater-1.1.0.jar    ← 下载完成
.minecraft/update/updater-1.0.0.jar    ← 旧版本仍在
         │
         ▼ 下次启动
引导器扫描 → 发现 1.1.0 > 1.0.0 → 加载 updater-1.1.0.jar
         │
         ▼ 新版本更新器启动后，清理旧版本
updater-1.0.0.jar 被删除
```

**关键设计：**
- 新版本先写为 `updater-x.x.x.jar.new`，下载完成后才重命名为 `.jar`，避免引导器加载到不完整的文件（详见 `../引导器/README.md`）
- 新版本和旧版本共存，不会出现"正在运行的 JAR 被覆盖"的问题
- 引导器始终选版本最大的，天然具备回滚能力（保留旧版本不删除即可）
- 更新器完全不需要关心自身的更新逻辑

### 3.3 旧版本清理

新版本更新器启动后，在 `premain()` 中检查同目录下的 `updater-*.jar`：

```
扫描结果:
  updater-1.0.0.jar    ← 版本 < 当前自身版本 → 删除
  updater-1.0.1.jar    ← 版本 < 当前自身版本 → 删除
  updater-1.1.0.jar    ← 当前自身版本 → 保留
```

> 如果管理员想保留某个旧版本用于回滚，可将其重命名为 `updater-1.0.0.jar.bak`，更新器不会清理 `.bak` 文件。

---

## 四、完整更新流程

```
JVM 启动（-javaagent:launcher-agent.jar）
      │
      ├─ 1. 引导器扫描 updater-*.jar，加载版本最大的
      │
      ├─ 2. 更新器 premain() 开始执行
      │
      ├─ 3. 清理旧版本 updater（版本号 < 当前自身的删除）
      │
      ├─ 4. 读取 config.json 获取服务端地址
      │
      ├─ 5. 依次尝试 servers 列表，锁定基准地址
      │     ├─ 全部不可达
      │     │   ├─ 本地有 local-manifest.json → 使用本地缓存，跳到第 12 步
      │     │   └─ 本地无缓存（首次启动） → 弹出崩溃窗口，阻止启动
      │     └─ 某地址响应成功 → 锁定为基准地址，继续
      │
      ├─ 6. 请求基准地址/version.json，与 local-version.json 对比
      │         ├─ 版本一致 → 跳到第 12 步
      │         └─ 版本不一致 → 继续
      │
      ├─ 7. 请求基准地址/manifest.json
      │     └─ 与 .updater/local-manifest.json 对比
      │         ├─ 文件 SHA1 一致 → 跳过该文件
      │         └─ 文件 SHA1 不一致 或 新增文件 → 继续
      │
      ├─ 8. 下载文件（含可能的新版本更新器 updater-*.jar）
      │     ├─ 有 downloads → 所有 CDN + 服务端同时全速拉取，谁快用谁
      │     └─ 无 downloads → 仅从服务端下载
      │     ├─ 桌面端 → Swing 窗口展示进度条和文件名
      │     └─ Android（FCL）→ 通知栏进度条实时更新
      │
      ├─ 9. 应用行为规则（参照服务端 action.json）
      │     ├─ check_and_overwrite → 校验 SHA1 后覆盖
      │     ├─ add_if_missing → 不存在才下载
      │     ├─ skip_if_modified → 玩家改过就跳过
      │     ├─ force_delete → 强制删除
      │     └─ strict → 清理目录多余文件
      │
      ├─ 10. 写入本地缓存
      │      ├─ local-version.json（更新版本号）
      │      ├─ local-manifest.json（更新清单）
      │      └─ local-changelog.json（从基准地址/changelog.json 拉取）
      │
      ├─ 11. 展示更新日志（showChangelog = true 且有更新时）
      │      ├─ 桌面端（Win/Mac/Linux）→ Swing 窗口展示
      │      ├─ Android（FCL）→ 通知栏展示
      │      ├─ 最新版本号 + 更新内容直接展示
      │      ├─ 历史版本默认折叠，点击展开
      │      └─ changelogDuration 秒后自动关闭，或游戏窗口获焦后关闭
      │
      ├─ 12. 更新器 premain() 返回 → JVM 继续启动游戏
      │
      └─ 13. Minecraft 正常启动
```

---

## 五、日志拉取

| 接口 | 说明 |
|---|---|
| `GET {基准地址}/changelog.json` | 拉取完整更新日志，包含 `currentVersion` 和 `history` 数组 |

- 客户端在更新完成后（或版本一致但本地无日志缓存时）拉取
- 缓存到 `.updater/local-changelog.json`
- 展示时优先读本地缓存，避免每次启动都请求

---

## 六、数字路径映射

客户端收到清单中的 `path` 如 `"1/mods/xxx.jar"`，根据前缀数字映射到实际目录：

| 前缀 | 映射规则 |
|:---:|---|
| `0` | 更新器自身所在目录（即 `launcher-agent.jar` / `updater-*.jar` 所在目录） |
| `1` | 上一级目录
| `2` | 上两级目录 |
| `3/4/5...` | 以此类推 |

**示例：** 假设 `launcher-agent.jar` 位于 `D:\Games\.minecraft\update\`

| path | 实际路径 |
|---|---|
| `0/config/updater.cfg` | `D:\Games\.minecraft\update\config\updater.cfg` |
| `1/mods/xxx.jar` | `D:\Games\.minecraft\mods\xxx.jar` |
| `1/config/forge.cfg` | `D:\Games\.minecraft\config\forge.cfg` |
| `2/launcher.jar` | `D:\Games\launcher.jar` |

---

## 七、下载回退逻辑

```
对于清单中的每个文件：
  │
  ├─ 构建下载源列表：
  │   ├─ downloads 中的所有 CDN 地址（如有）
  │   └─ 服务端地址/{path}
  │
  ├─ 所有源同时发起请求，全速拉取
  │   ├─ 某个源返回完整文件 → 校验 SHA1 → 通过即完成
  │   └─ 丢弃其他未完成的请求
  │
  └─ 全部失败 → 进入错误处理
```

> **设计原则：** 所有下载源同时全速拉取，谁先拿到完整文件用谁。CDN 和服务端带宽叠加利用，不浪费任何一个源的速度。

---

## 八、错误处理与崩溃界面

当关键文件下载失败（所有下载源 + 服务端回退均失败）时，Agent **主动中断游戏启动**，弹出错误窗口：

### 8.1 崩溃窗口功能

```
┌─────────────────────────────────────────┐
│  ❌ MC 更新器 — 更新失败                │
├─────────────────────────────────────────┤
│                                         │
│  更新过程中遇到错误，游戏无法启动：      │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │ [错误详情文本框，可滚动]         │    │
│  │                                 │    │
│  │ 服务器:                         │    │
│  │  [1] main-server.com → 连接失败 │    │
│  │  [2] backup1.com → 已连接       │    │
│  │                                 │    │
│  │ 失败文件: 1/mods/xxx.jar        │    │
│  │ 原因: 所有下载源均不可达        │    │
│  │                                 │    │
│  │ 尝试的地址:                     │    │
│  │  1. https://mediafilez... → 超时│    │
│  │  2. https://edge... → 超时      │    │
│  │  3. https://cdn.modrinth...→ 404│    │
│  │  4. http://backup1/... → 500    │    │
│  └─────────────────────────────────┘    │
│                                         │
│  ┌──────────┐  ┌──────────┐            │
│  │复制错误信息│  │  关 闭   │            │
│  └──────────┘  └──────────┘            │
│                                         │
│  提示：请将错误信息发送给服务器管理员    │
└─────────────────────────────────────────┘
```

### 8.2 「复制错误信息」按钮

点击后将以下内容复制到剪贴板：

```
=== MC 更新器错误报告 ===
时间: 2026-09-12 10:30:00
客户端版本: updater-1.0.0
服务器地址:
  [1] http://main-server.com:8080        → 连接失败
  [2] http://backup1.example.com:8080    → 已连接（基准地址）
  [3] http://backup2.example.com:8080    → 未尝试

失败文件: 1/mods/xxx.jar
文件大小: 25593 bytes
预期SHA1: d613dbc38ca5bcc718408db432385c2e67ac763f

下载尝试记录:
  [1] https://mediafilez.forgecdn.net/files/... → 超时 (30s)
  [2] https://edge.forgecdn.net/files/...       → 超时 (30s)
  [3] https://cdn.modrinth.com/data/...         → HTTP 404
  [4] http://backup1.example.com/1/mods/...     → HTTP 500

本地环境:
  OS: Windows 11
  Java: 17.0.2
  工作目录: D:\Games\.minecraft\update\
```

### 8.3 何时触发崩溃

| 场景 | 处理方式 |
|---|---|
| 版本一致，无需更新 | 正常启动游戏 |
| 更新成功 | 展示日志 → 启动游戏 |
| 非关键文件下载失败（如资源文件） | 跳过该文件，记录警告，继续更新 |
| **关键文件下载失败**（`.jar` 模组、核心配置） | **弹出崩溃窗口，阻止游戏启动** |
| 服务端不可达（version.json 请求失败） | 使用本地缓存启动，记录警告 |
| 本地 manifest 为空（首次启动）且服务端不可达 | **弹出崩溃窗口** |

> **关键文件**的判断标准：文件扩展名为 `.jar`，或 `action` 为 `check_and_overwrite` / `force_overwrite` 的文件。

---

## 九、平台适配与 UI 展示

### 9.1 桌面端（Windows / macOS / Linux）

更新过程中使用 **Swing** 弹出轻量窗口展示进度：

```
┌─────────────────────────────────────────┐
│  🔄 MC 更新器                           │
├─────────────────────────────────────────┤
│                                         │
│  正在更新...                             │
│                                         │
│  ████████████████░░░░░░░░  65%          │
│                                         │
│  当前文件: mods/Create-1.21.1.jar       │
│  速度: 12.5 MB/s  剩余: 约 3 秒         │
│                                         │
│  总进度: 13/20 个文件                    │
└─────────────────────────────────────────┘
```

更新完成后该窗口变为日志展示窗口（展示 changelog），或自动关闭。

### 9.2 Android（FCL 启动器）

FCL（Fold Craft Launcher）环境下，更新器使用 **Android 通知** 展示进度：

**首次启动时：**
1. 检测到 Android 环境（FCL 启动器）
2. 请求通知权限（`POST_NOTIFICATIONS`）
3. 用户授权后，后续更新通过通知栏展示

**通知样式：**

```
┌─────────────────────────────────────────┐
│ 🔄 MC 更新器                     [展开] │
│ ████████████████░░░░░░  65% — 13/20 文件│
│ 当前: mods/Create-1.21.1.jar            │
└─────────────────────────────────────────┘
```

- 使用 `NotificationCompat.Builder` 构建进度通知
- `setProgress(100, 65, false)` 展示进度条
- 每下载完一个文件更新一次通知
- 更新完成或失败后，通知变为"更新完成，正在启动游戏"或错误信息

**Android 权限处理：**

| 情况 | 处理 |
|---|---|
| 用户授权通知权限 | 正常展示通知进度条 |
| 用户拒绝通知权限 | 静默更新，不展示 UI，日志写入 `.updater/update.log` |
| Android < 13（无需权限） | 直接展示通知 |

### 9.3 平台检测逻辑

```java
if (isAndroid()) {
    // FCL 环境：请求通知权限 → 使用 NotificationManager
} else {
    // 桌面环境：使用 Swing 窗口
}
```

检测方式：通过系统属性 `os.name` 或检查是否存在 Android 相关类（`android.os.Build`）。
