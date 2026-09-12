# MC 更新器 — 引导器文档

> **技术栈：** Java / Kotlin
>
> 引导器（Launcher Agent）是整个更新系统的入口，通过 `-javaagent` 挂载到 Minecraft JVM 上。它不负责具体的更新逻辑，只做一件事：**扫描本地目录，找到版本号最大的更新器 JAR，然后委托它执行更新。**

---

## 一、目录结构

```
.minecraft/update/
├── launcher-agent.jar              # 引导器（-javaagent 加载的入口）
├── updater-1.0.0.jar               # 更新器 v1.0.0
├── updater-1.0.1.jar               # 更新器 v1.0.1  ← 引导器会选这个
├── updater-1.0.2.jar.new           # 下载中的新版本，尚未就绪
├── config.json                     # 客户端配置
└── .updater/
    ├── local-version.json
    ├── local-manifest.json
    └── local-changelog.json
```

---

## 二、工作原理

### 2.1 JVM 启动参数

```bash
java -javaagent:update/launcher-agent.jar \
     -cp <minecraft-classpath> \
     net.minecraft.client.main.Main
```

启动器只负责加载 `launcher-agent.jar`，不关心更新器的版本。

### 2.2 引导器执行流程

```
JVM 启动
    │
    ├─ [Phase 1] Agent premain() 执行
    │   │
    │   ├─ 1. 扫描 launcher-agent.jar 同目录下的 updater-*.jar
    │   │      排除 *.jar.new（下载中/未就绪的文件）
    │   │
    │   ├─ 2. 按语义版本号排序，选取版本最大的 updater JAR
    │   │      例: updater-1.0.0.jar < updater-1.0.1.jar < updater-1.0.2.jar
    │   │
    │   ├─ 3. 通过自定义 ClassLoader 加载该 updater JAR
    │   │      读取其 MANIFEST.MF 中的 Agent-Class
    │   │
    │   └─ 4. 调用更新器的 premain()，将控制权交给更新器
    │          更新器执行：检查更新 → 下载文件 → 应用规则 → 展示日志
    │
    ├─ [Phase 2] 更新器 premain() 返回
    │
    └─ [Phase 3] Game main() 启动
```

### 2.3 版本号命名规则

更新器 JAR 文件名必须遵循以下格式：

```
updater-{major}.{minor}.{patch}.jar
```

| 示例 | 说明 |
|---|---|
| `updater-1.0.0.jar` | 正式版本 |
| `updater-1.0.1.jar` | 正式版本（更高） |
| `updater-1.0.2.jar.new` | 下载中，引导器会忽略 |
| `updater-2.0.0-beta.jar` | 忽略（非标准格式，引导器不识别） |

---

## 三、更新器自身更新流程

这是引导器存在的核心价值——**更新器可以更新自己**：

```
更新器运行中
    │
    ├─ 检测到自身有新版本（服务端 manifest 中有更新的 updater JAR）
    │
    ├─ 下载新版本 → 写为 updater-1.0.2.jar.new
    │
    ├─ 下载完成 → 重命名为 updater-1.0.2.jar
    │   （此时旧版本 updater-1.0.1.jar 仍可正常运行）
    │
    ├─ 标记旧版本待清理
    │
    └─ premain() 返回 → 游戏正常启动

下次启动：
    │
    ├─ 引导器扫描 → 发现 updater-1.0.2.jar 版本最高
    │
    ├─ 加载 updater-1.0.2.jar（新版本）
    │
    ├─ 新版本更新器启动后，清理旧版本 updater-1.0.1.jar
    │
    └─ 继续正常更新流程
```

**关键设计：**
- 新版本先写为 `.jar.new`，下载完成后才重命名为 `.jar`，避免加载到不完整的文件
- 旧版本在新版本成功启动后才清理，保证回滚能力
- 引导器功能极简（扫描、排序、加载），写好后**不需要更新**

---

## 四、引导器配置

引导器本身不需要独立配置文件，所有配置复用客户端的 `config.json`。引导器只负责：

1. 读取同目录下的 `config.json`（传递给更新器）
2. 扫描 `updater-*.jar`（排除 `.new` 后缀）
3. 按版本号排序，加载最新的

---

## 五、错误处理

| 场景 | 处理方式 |
|---|---|
| 目录下没有 `updater-*.jar` | 弹出崩溃窗口："未找到更新器，请重新安装" |
| 所有 updater JAR 都是 `.new` 状态 | 弹出崩溃窗口："更新器下载未完成，请重新启动" |
| updater JAR 加载失败（损坏/版本号解析失败） | 弹出崩溃窗口，带一键复制按钮 |
| `config.json` 不存在 | 弹出崩溃窗口："缺少配置文件 config.json" |

---

## 六、引导器 vs 更新器 职责划分

| 职责 | 引导器（launcher-agent.jar） | 更新器（updater-*.jar） |
|---|---|---|
| 挂载方式 | `-javaagent` 直接加载 | 引导器通过 ClassLoader 加载 |
| 检查更新 | ❌ | ✅ |
| 下载文件 | ❌ | ✅ |
| 应用行为规则 | ❌ | ✅ |
| 版本选择 | ✅（扫描最新 updater） | ❌ |
| 清理旧版本 updater | ❌ | ✅（启动后清理版本号低于自身的） |
| 展示日志/错误 | ✅（仅引导器自身的错误） | ✅（更新过程的日志和错误） |

> 引导器写好后**不需要更新**，更新器的新版本由服务端作为普通文件（`0/updater-x.x.x.jar`）随游戏文件一起下发。
