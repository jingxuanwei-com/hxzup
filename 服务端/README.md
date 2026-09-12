# MC 更新器 — 服务端文档

> **技术栈：** Go  
> 本文档为服务端应用的运行说明。应用运行位置即为本文件所在目录，所有数据文件存放于 `data/` 目录下。

---

## 一、目录结构

```
服务端/
├── 服务端.md              # 本文档
└── data/
    ├── manifest.json      # 最终生成的更新清单（供客户端拉取）
    ├── version.json       # 随机版本号（供客户端快速判断是否有更新）
    ├── cache-store.json   # 下载地址本地缓存（避免重复查询）
    ├── action.json        # 更新行为规则（全局/目录/文件三级）
    ├── changelog.json     # 版本号 & 更新日志（供玩家查看）
    ├── game-profile.json  # 游戏版本 & 加载器配置（可选）
    │   
    ├── 0/                 # → 客户端当前目录（更新器所在目录）
    ├── 1/                 # → 客户端上级目录
    ├── 2/                 # → 客户端上两级目录
    ├── 3/                 # → 客户端上三级目录
    ├── 4/                 # → ...以此类推
    └── 5/
```

---

## 二、数字文件夹映射规则

| 文件夹 | 映射目标 | 示例 |
|:---:|---|---|
| `0/` | 客户端当前目录 | `0/config/mod.cfg` → `<客户端>/config/mod.cfg` |
| `1/` | 客户端上级目录 | `1/mods/xxx.jar` → `<客户端上级>/mods/xxx.jar` |
| `2/` | 客户端上两级目录 | 同理递推 |
| `3/4/5...` | 以此类推 | — |

---

## 三、核心文件说明

### 3.1 `manifest.json` — 更新清单

服务端根据数字文件夹自动扫描生成，供客户端拉取后对比更新。

```json
{
  "files": [
    {
      "path": "1/mods/Almanac-1.21.1-2-neoforge-1.5.2.jar",
      "sha1": "d613dbc38ca5bcc718408db432385c2e67ac763f",
      "fileSize": 25593,
      "action": "check_and_overwrite",
      "downloads": [
        "https://mediafilez.forgecdn.net/files/7489/91/Almanac-1.21.1-2-neoforge-1.5.2.jar",
        "https://edge.forgecdn.net/files/7489/91/Almanac-1.21.1-2-neoforge-1.5.2.jar",
        "https://cdn.modrinth.com/data/Gi02250Z/versions/cHGan9fQ/Almanac-1.21.1-2-neoforge-1.5.2.jar",
        "https://mod.mcimirror.top/data/Gi02250Z/versions/cHGan9fQ/Almanac-1.21.1-2-neoforge-1.5.2.jar",
        "https://mod.mcimirror.top/files/7489/91/Almanac-1.21.1-2-neoforge-1.5.2.jar"
      ]
    }
  ]
}
```

**字段说明：**

| 字段 | 说明 |
|---|---|
| `path` | 相对路径，前缀数字表示目标目录层级 |
| `sha1` | 文件 SHA-1 校验值 |
| `fileSize` | 文件大小（字节） |
| `action` | 更新行为（可选），可选值见下方行为模式表；为空则按 `action.json` 中的规则匹配 |
| `downloads` | 下载地址列表（可选），多个 CDN 地址用于加速，与服务端同时请求 |

**下载策略：**
- `downloads` 列表中的所有地址（官方 CDN + MCIM 镜像 + 服务端）**同时请求**，全速拉取，谁先返回有效结果用谁
- 收到第一个完整文件后立即校验 SHA-1，通过即完成，丢弃其他正在进行的请求
- 若文件未配置 `downloads`，则仅从服务端下载

---

### 3.2 `version.json` — 随机版本号

```json
{
  "version": "每次变动随机一个版本号"
}
```

- **用途**：客户端通过对比此版本号快速判断是否有更新，无需每次都拉取完整的 `manifest.json`
- **更新方式**：管理面板点击按钮 → 生成随机版本号写入
- ⚠️ 此版本号**仅用于更新判断**，不展示给玩家

---

### 3.3 `cache-store.json` — 下载地址缓存

```json
{
  "cache": {
    "d613dbc38ca5bcc718408db432385c2e67ac763f": {
      "fileSize": 25593,
      "downloads": [
        "https://mediafilez.forgecdn.net/files/7489/91/xxx.jar",
        "https://cdn.modrinth.com/data/.../xxx.jar",
        "https://mod.mcimirror.top/data/.../xxx.jar",
        "https://mod.mcimirror.top/files/7489/91/xxx.jar"
      ]
    }
  }
}
```

**缓存策略：**

1. 服务端生成 `manifest.json` 时，对 `.jar` 文件先查询本地 `cache-store.json`
2. **命中缓存** → 直接使用已有的下载地址
3. **未命中** → 向 Modrinth / CurseForge API 查询下载地址，同时生成官方地址 + MCIM 镜像地址，写入缓存
4. **查询失败** → 同样写入缓存记录（标记为无结果），避免重复无效查询

---

### 3.4 `action.json` — 更新行为规则

统一控制客户端更新时对目录和文件的处理方式，分为 **全局 → 目录 → 文件** 三级，后者覆盖前者。

```json
{
  "rules": {
    "global": {
      "directory": "permissive",
      "file": "check_and_overwrite"
    },
    "directory": {
      "1/mods/": {
        "behavior": "strict",
        "description": "核心模组目录，严禁私自添加无关文件，多余文件将被清理"
      },
      "1/config/": {
        "behavior": "permissive",
        "description": "配置文件目录，允许玩家保留自定义修改或自行添加配置"
      }
    },
    "file": {
      "1/config/options.json": {
        "behavior": "skip_if_modified",
        "description": "游戏设置文件，玩家修改过则保留"
      },
      "1/config/old_mod.cfg": {
        "behavior": "force_delete",
        "description": "已废弃的配置文件，强制删除"
      }
    }
  }
}
```

#### 优先级（从低到高）

```
global.directory / global.file  （全局默认）
        ↓ 覆盖
    directory.{path}/            （目录级规则，路径以 / 结尾）
        ↓ 覆盖
      file.{path}                （文件级规则，精确路径）
```

客户端匹配文件时，从最具体的规则开始匹配，命中即停止。

#### 行为模式 如果遗漏或者不需要请自行规定

目录和文件共用同一套行为值，根据目标类型语义略有不同：

| 行为值 | 名称 | 用于目录时 | 用于文件时 |
|---|---|---|---|
| `check_and_overwrite` | 校验覆盖 | 目录内文件按此策略逐个校验更新，多余文件**保留** | 强制校验 SHA-1，哈希不匹配则覆盖；不存在则下载 |
| `add_if_missing` | 缺失添加 | 只往目录里加新文件，已有的不更新 | 文件不存在才下载，已存在则跳过 |
| `skip_if_modified` | 修改跳过 | 目录内文件：玩家改过的跳过，没改的正常更新 | 若玩家修改过（哈希不一致）则跳过，否则正常更新 |
| `strict` | 严格同步 | 目录必须与清单一致，多余文件**删除**，清单内文件按各自策略更新 | — |
| `permissive` | 宽松保留 | 仅更新清单内的文件，玩家自行添加的**保留** | — |
| `additive` | 纯添加 | 只加新文件，不更新已有，不删除多余 | — |
| `force_delete` | 强制删除 | — | 无论状态如何，强制删除该文件 |

---

### 3.5 `changelog.json` — 版本日志（玩家可见）

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

> **展示规则：** 客户端更新完成后，直接展示第一条（最新版本）的版本号和更新内容；其余历史版本折叠在「历史版本」区域，点击后展开查看。

- `currentVersion`：当前最新版本号（由管理面板维护）
- `history`：更新日志列表，**第一条为最新版本**，直接展示其版本号、日期和更新内容；历史版本默认折叠，点击展开查看
- ⚠️ 此版本号**不作为更新判断依据**，仅用于展示
- **客户端拉取接口**：`GET /changelog.json`，客户端更新完成后拉取并缓存到本地

---

### 3.6 `game-profile.json` — 游戏版本与加载器配置（可选）

> **此文件为可选功能。** 不配置则更新器只做文件同步，不处理游戏版本和加载器。

```json
{
  "gameVersion": "1.21.1",
  "loader": {
    "type": "neoforge",
    "version": "21.1.1"
  }
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `gameVersion` | string | 目标 Minecraft 版本号（如 `1.21.1`、`1.20.1`） |
| `loader.type` | string | 加载器类型，可选值：`forge`、`neoforge`、`fabric`、`quilt`、`optifine` |
| `loader.version` | string | 加载器版本号 |

#### 需求说明

1. **管理面板**可配置游戏版本和加载器（类型+版本），写入 `game-profile.json`
2. **客户端**启动时拉取此文件，与本地已安装的版本/加载器对比
3. 如果版本或加载器有变化，客户端需要：
   - 下载新版本的版本 JSON、客户端 JAR、libraries 等文件
   - 安装新加载器（参考 HMCL 的加载器安装逻辑）
   - 文件放到 `.minecraft/versions/<版本ID>/` 和 `.minecraft/libraries/` 下
4. 加载器安装完成后，**当前启动的 classpath 不变**，更新仅对下次启动生效
5. 客户端不需要实现完整的启动器功能，只需保证文件到位，启动器会自动识别

#### `.minecraft` 目录定位

更新器路径**固定**为：`.minecraft/versions/<游戏名>/updater/`

```
.minecraft/                                ← updater 上 3 级
├── versions/
│   └── <游戏名>/                          ← updater 上 1 级（游戏版本目录）
│       ├── <游戏名>.json
│       ├── <游戏名>.jar
│       └── updater/                       ← 更新器所在目录
│           ├── launcher-agent.jar
│           ├── updater-1.0.0.jar
│           ├── config.json
│           └── .updater/
├── libraries/
└── assets/
```

整合包作者只需将 updater 文件夹放到 `versions/<游戏名>/` 下即可。
`config.json` 中的 `gameDir` 字段可覆盖此默认路径（用于特殊目录结构）。

#### 版本 ID 命名约定

| 加载器 | 版本 ID 格式 | 示例 |
|---|---|---|
| 原版 | `{gameVersion}` | `1.21.1` |
| Forge | `{gameVersion}-forge-{loaderVersion}` | `1.21.1-neoforge-21.1.1` |
| Fabric | `{gameVersion}-fabric-{loaderVersion}` | `1.20.1-fabric-0.15.0` |
| Quilt | `{gameVersion}-quilt-{loaderVersion}` | `1.20.1-quilt-0.22.0` |

#### 加载器安装参考

实现时参考 `参考代码/HMCL-main/` 中的加载器安装逻辑，主要包括：
- Forge/NeoForge：下载 installer JAR，执行 processors 生成版本文件
- Fabric/Quilt：调用 meta API 获取 launcherMeta，生成版本 JSON 和 libraries
- 所有加载器的 libraries 文件放到 `.minecraft/libraries/` 共享目录

#### 游戏文件下载源

游戏版本文件、加载器文件、模组文件均使用**多源并发下载**，确保国内可正常下载：

| 源 | 用途 | 镜像说明 |
|---|---|---|
| Mojang 官方 | `https://piston-data.mojang.com/` 等 | 游戏本体 |
| BMCLAPI | `https://bmclapi2.bangbang93.com/` | Mojang/Forge/Fabric/NeoForge 镜像 |
| Modrinth 官方 | `https://cdn.modrinth.com/` | 模组 CDN |
| Modrinth MCIM 镜像 | `https://mod.mcimirror.top/` | Modrinth API + CDN 加速 |
| CurseForge 官方 | `https://mediafilez.forgecdn.net/` 等 | 模组 CDN |
| CurseForge MCIM 镜像 | `https://mod.mcimirror.top/curseforge/` | CurseForge API + CDN 加速 |

**模组下载地址生成逻辑：**
- 服务端通过 Modrinth / CurseForge API 查询模组文件的下载地址
- 同时生成官方地址和 MCIM 镜像地址，写入 manifest 的 `downloads` 列表
- 客户端并发请求所有地址，谁快用谁

> BMCLAPI 镜像仅支持游戏本体和加载器，不支持模组。模组下载使用 Modrinth / CurseForge + MCIM 镜像。

---

## 四、管理面板功能

| 功能 | 说明 |
|---|---|
| **登录** | 简单的身份认证 |
| **在线文件管理** | 可视化管理 `data/` 下的数字文件夹及文件（上传/删除/移动），列表中直接显示每个文件/目录的当前行为标签；右键菜单可快速**添加/修改/删除**该条目的更新行为规则 |
| **全局行为设置** | 设置目录和文件的全局默认更新行为 |
| **刷新版本号** | 一键生成新的随机版本号写入 `version.json` |
| **更新日志** | 编辑版本号和更新内容，支持新增/修改/删除历史版本记录 |
| **游戏版本管理** | （可选）配置目标游戏版本和加载器（类型+版本），见 `game-profile.json` 说明 |

---

## 五、更新流程概览

```
客户端                                      服务端
  │                                          │
  ├─ 1. 依次尝试 servers 列表，锁定基准地址 ─►│
  │◄─ 某地址响应成功 ────────────────────────┤
  │                                          │
  ├─ 2. 请求基准地址/version.json ───────────►│
  │◄─ 返回随机版本号 ────────────────────────┤
  │                                          │
  ├─ 3. 对比 local-version.json              │
  │   ├─ 版本一致 → 跳过更新，启动游戏       │
  │   └─ 版本不一致 → 继续                   │
  │                                          │
  ├─ 4. 请求基准地址/manifest.json ──────────►│
  │◄─ 返回完整更新清单 ──────────────────────┤
  │                                          │
  ├─ 5. 对比 local-manifest.json             │
  │   ├─ 文件 SHA1 一致 → 跳过               │
  │   └─ 不一致/新增 → 下载文件               │
  │       ├─ 有 downloads → 所有 CDN + 服务端 │
  │       │   同时全速拉取，谁快用谁          │
  │       └─ 无 downloads → 从服务端下载      │
  │       ├─ 桌面端 → Swing 进度条            │
  │       └─ Android → 通知栏进度条           │
  │                                          │
  ├─ 6. 应用行为规则（action.json）           │
  │                                          │
  ├─ 7. 写入本地缓存                         │
  │   ├─ local-version.json                  │
  │   ├─ local-manifest.json                 │
  │   └─ local-changelog.json                │
  │                                          │
  ├─ 8. 请求基准地址/changelog.json ─────────►│
  │◄─ 返回更新日志 ──────────────────────────┤
  │                                          │
  ├─ 9. 展示更新日志（可选）                  │
  │   ├─ 桌面端 → Swing 窗口                  │
  │   ├─ Android → 通知栏                     │
  │   ├─ 最新版本 + 更新内容直接展示          │
  │   └─ 历史版本（默认折叠，点击展开）       │
  │                                          │
  └─ 10. 启动 Minecraft
```