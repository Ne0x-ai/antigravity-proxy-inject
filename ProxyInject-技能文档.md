# ProxyInject —— macOS App 代理注入技能文档

> **本文档是给 AI Agent 的执行手册。**
> Agent 收到本文档后，应按照「执行流程」中的步骤，逐条执行命令、核对每步的预期结果，遇到失败按对应「失败处理」执行，最后完成「验收标准」自检。
> 全程不得跳过备份与验证步骤。所有命令均为通用 macOS shell 命令，适配任意 AI Agent 环境。

---

## 一、这个文档解决什么问题

macOS 上部分 AI 编程工具（Antigravity、Claude 等）的后台服务（如 `language_server`）是 Go 进程，**只认 `HTTP_PROXY` 等环境变量，不读系统代理设置**。因此需要通过环境变量注入代理，才能让这些后台服务走本地代理联网。

常规做法是写一个 `.command` 脚本，启动前手动执行。本技能的目标是：**直接修改目标 App 的 `Info.plist`，把代理环境变量写入 `LSEnvironment` 键**，利用 macOS 官方机制——GUI 启动（Dock、调度台、`open` 命令、双击）时由 launchd 自动注入进程环境——实现「任意方式打开 App 都自动走代理」，无需脚本、无需每次手动操作。

**适用对象**：macOS 上需要走本地代理的 AI 编程工具 App（Antigravity、Claude 等 Electron/原生应用）。本文档以 Antigravity 为主示例，其他 App 按「通用化」一节调整即可。

---

## 二、执行前检查（必须先做，缺一不可）

Agent 在动手前，依次执行以下检查，任何一项不满足都要先解决再继续。

### 检查 1：目标 App 是否存在

```bash
ls -la "/Applications/Antigravity.app/Contents/Info.plist"
```

- **预期**：输出文件信息，存在。
- **失败处理**：文件不存在，停止并告知用户：目标 App 未安装或路径不对，请确认 App 名称与安装路径。

### 检查 2：代理端口是否连通

默认代理地址为 `http://127.0.0.1:7892`（用户本地代理软件的端口，如 Clash 等常见工具，具体以实际代理软件为准）。**先探测代理是否可用**，不通则后续注入无意义。

```bash
curl -s -o /dev/null -w "%{http_code}" --max-time 5 -x http://127.0.0.1:7892 https://www.gstatic.com/generate_204
```

- **预期**：输出 `204`（或 `200`），代理可用。
- **失败处理**：输出 `000` 或超时，说明代理未启动或端口不对。停止并告知用户：先启动本地代理软件（如 Clash 等），或确认代理端口后修改本文档中的代理地址，重新探测。

### 检查 3：当前是否有目标 App 在运行

```bash
pgrep -fl "Antigravity"
```

- **预期**：无输出，或仅用户主动保留的进程。
- **失败处理**：若有残留进程，告知用户先完全退出目标 App（或由 Agent 执行 `pkill -f "Antigravity.app/Contents/MacOS/Antigravity"` 后再继续）。**已运行的进程带着旧环境变量，复用它们等于没走代理。**

---

## 三、执行流程（核心步骤，按顺序执行）

以下每一步都必须执行，并核对预期结果。**任何一步失败，按该步「失败处理」执行，不得继续往下走。**

### 步骤 1：备份 Info.plist

**做什么**：修改前先备份，作为回滚依据。

```bash
cp "/Applications/Antigravity.app/Contents/Info.plist" "/Users/$(whoami)/Desktop/antigravity-Info.plist.bak"
```

- **预期**：备份文件生成成功。
- **验证**：`ls -la` 备份文件，确认存在且大小与原文件一致。
- **失败处理**：备份失败（如权限不足），立即停止，不要继续修改。

### 步骤 2：注入代理环境变量到 LSEnvironment

**做什么**：向 `Info.plist` 的 `LSEnvironment` 键追加代理变量。`LSEnvironment` 是苹果官方机制：GUI 启动 App 时，launchd 会自动把这些变量注入进程环境，子进程（如 `language_server`）继承。

```bash
python3 - <<'EOF'
import plistlib

APP = "/Applications/Antigravity.app/Contents/Info.plist"
PROXY = "http://127.0.0.1:7892"

with open(APP, "rb") as f:
    d = plistlib.load(f)

env = d.setdefault("LSEnvironment", {})
env["HTTP_PROXY"] = PROXY
env["HTTPS_PROXY"] = PROXY
env["http_proxy"] = PROXY
env["https_proxy"] = PROXY
env["NO_PROXY"] = "localhost,127.0.0.1,::1,.local"
env["no_proxy"] = "localhost,127.0.0.1,::1,.local"

with open(APP, "wb") as f:
    plistlib.dump(d, f, fmt=plistlib.FMT_XML)

print("注入完成：", env)
EOF
```

- **预期**：输出 `注入完成：{...}`，包含 6 个代理变量。
- **说明**：
  - 大小写都要注入：Go 语言 `language_server` 只认大写 `HTTP_PROXY`/`HTTPS_PROXY`，同时注入小写更保险。
  - `NO_PROXY` 必须包含 `localhost,127.0.0.1,::1,.local`，防止 renderer 与 language_server 之间的本地 gRPC 走代理导致死循环。
- **失败处理**：报错（如 `plistlib` 解析失败、文件被占用），立即停止，从备份恢复（见「回滚流程」）。

### 步骤 3：清理隔离属性

**做什么**：清除 App 的隔离属性（quarantine），避免系统因签名校验问题拒绝启动。

```bash
xattr -cr "/Applications/Antigravity.app"
```

- **预期**：无报错输出，命令执行成功。
- **失败处理**：报权限不足，告知用户手动执行，或确认当前用户对该目录有写权限。

### 步骤 4：重新签名

**做什么**：修改 `Info.plist` 后，原签名失效，launchd 会拒绝启动（错误 `Launch failed` / `Launchd job spawn failed`）。必须重新签名。

```bash
codesign --force --deep --sign - "/Applications/Antigravity.app"
```

- **预期**：输出 `replacing existing signature`，无报错。
- **说明**：
  - `-` 表示 ad-hoc 临时签名，无需开发者证书，功能不受影响。
  - 若用户有开发者证书并希望保留正式签名，可改用用户证书签名：`codesign --force --deep --sign "证书名称" "/Applications/Antigravity.app"`（证书名称需用户提供）。
- **验证**：

```bash
codesign -vv "/Applications/Antigravity.app"
```

- **预期**：输出 `valid on disk` 和 `satisfies its Designated Requirement`。
- **失败处理**：签名失败，立即停止并告知用户，不要继续启动测试。

### 步骤 5：启动并验证环境变量注入

**做什么**：从 GUI 方式启动目标 App，确认代理变量确实注入到进程环境。

```bash
open -a Antigravity
sleep 6
PID=$(pgrep -f "Antigravity.app/Contents/MacOS/Antigravity" | head -1)
ps eww -p "$PID" | tr ' ' '\n' | grep -i proxy
```

- **预期**：输出 6 个代理变量（`HTTP_PROXY`、`HTTPS_PROXY`、`http_proxy`、`https_proxy`、`NO_PROXY`、`no_proxy`），值均为 `http://127.0.0.1:7892`。
- **失败处理**：
  - 若 App 无法启动或报错：说明签名/配置仍有问题，按「回滚流程」恢复，或重新检查步骤 3、4。
  - 若启动成功但无代理变量输出：检查步骤 2 是否写入成功（`plutil -p` 查看 `LSEnvironment`），确认后重新执行步骤 3、4、5。

### 步骤 6：验证语言服务子进程（可选但推荐）

**做什么**：确认后台 `language_server` 进程也继承了代理变量（这是真正需要走代理的进程）。

```bash
LS_PID=$(pgrep -f "Resources/bin/language_server" | head -1)
ps eww -p "$LS_PID" | tr ' ' '\n' | grep -i proxy
```

- **预期**：同样输出 6 个代理变量。
- **说明**：`language_server` 由主程序按需拉起，可能需要打开项目/文件后才出现。若暂未出现，可稍等或触发一次语言服务后再查；子进程默认继承父进程环境，主进程已注入则子进程必然继承。

---

## 四、验收标准（Agent 必须逐条自检）

执行完成后，Agent 需逐条核对，全部通过才算完成：

| # | 验收项 | 验证命令 | 通过标准 |
|---|--------|---------|---------|
| 1 | 签名有效 | `codesign -vv /Applications/Antigravity.app` | 输出 `valid on disk` + `satisfies its Designated Requirement` |
| 2 | 配置已写入 | `plutil -p /Applications/Antigravity.app/Contents/Info.plist \| grep -A 12 LSEnvironment` | 能看到全部 6 个代理变量 |
| 3 | 启动正常 | `open -a Antigravity` | App 正常打开，无报错弹窗 |
| 4 | 主进程注入 | `ps eww -p <主进程PID> \| tr ' ' '\n' \| grep -i proxy` | 6 个代理变量齐全 |
| 5 | 子进程注入 | `ps eww -p <language_server PID> \| tr ' ' '\n' \| grep -i proxy` | 6 个代理变量齐全（若进程已出现） |
| 6 | 代理连通 | `curl -s -o /dev/null -w "%{http_code}" --max-time 5 -x http://127.0.0.1:7892 https://www.gstatic.com/generate_204` | 输出 `204` |

任一条不通过，需定位原因并修复后重测，不得向用户声称「已完成」。

---

## 五、踩坑与注意事项（重要，务必遵守）

以下为实际执行中验证过的坑，Agent 必须了解，避免重复踩：

1. **改 Info.plist 后必须重新签名**：不重新签名，`open` 启动会报 `Launch failed` / `Launchd job spawn failed`（错误码 162），App 打不开。这是最常见的失败原因。
2. **必须清理隔离属性**：重新签名后建议执行 `xattr -cr`，避免系统因隔离属性拦截启动。
3. **已运行的进程不受影响**：`LSEnvironment` 只对**新启动**的进程生效。若目标 App 已运行，必须先完全退出再重新打开，否则带着旧环境变量，等于没走代理。
4. **代理必须连通**：若本地代理未启动或端口不对，App 会正常打开但后台服务无法联网。注入前务必先探测代理（见检查 2）。
5. **代理出口地理位置**：若走代理后仍报 `User location is not supported for the API use`，说明代理出口 IP 所在地区不被服务方支持，需在代理工具中切换到支持的地区的节点（如美国节点），无需重新注入。
6. **App 自动更新会覆盖配置**：Electron 类应用更新会覆盖 `Info.plist`，注入会失效。更新后需重新执行本文档「执行流程」。**本文档模式无法自动监听更新，需人工（或由 Agent 按需）重新执行。**
7. **桌面上的 `.command` 可能是 Alias 快捷方式**：若用户桌面上的启动脚本打不开或内容是乱码二进制，它其实是指向 `/Applications` 下真实脚本的 Finder 快捷方式，真实脚本在 `/Applications` 下。

---

## 六、回滚流程（失败或用户要求还原时执行）

```bash
# 1. 用备份覆盖回原 Info.plist
cp "/Users/$(whoami)/Desktop/antigravity-Info.plist.bak" "/Applications/Antigravity.app/Contents/Info.plist"

# 2. 清理隔离属性
xattr -cr "/Applications/Antigravity.app"

# 3. 重新签名（恢复可启动状态）
codesign --force --deep --sign - "/Applications/Antigravity.app"
```

**执行后验证**：`codesign -vv` 通过、App 可正常启动，即回滚成功。回滚后如需再次注入，重新执行「执行流程」即可。

---

## 七、通用化：适配其他 App 与多代理

### 适配其他 App

本技能不限于 Antigravity。适配其他 macOS App 时，只需替换：

1. **App 路径**：将 `/Applications/Antigravity.app` 替换为目标 App 路径（如 `/Applications/Claude.app`）；
2. **进程匹配**：将 `pgrep` 中的进程特征替换为目标 App 的进程名；
3. **备份文件名**：将 `antigravity` 替换为目标 App 名称。

其他操作（LSEnvironment 注入、重签名、验证、回滚）完全一致。**前提**：目标 App 使用标准 `.app` bundle 结构，且后台服务依赖环境变量走代理。

### 多代理配置

若需为多个 App 配置不同代理，可复制本文档并参数化 `PROXY` 变量，为每个 App 维护一份配置；或在执行时按 App 逐个注入不同代理地址。注意：`LSEnvironment` 是**按 App 独立注入**的，互不影响。

### 状态管理（文档模式的能力边界）

本文档模式**不维护状态、不做自动监听**。以下能力需要工具化才能实现，文档模式无法覆盖：

- App 自动更新后**无人值守自动重做**注入；
- 多 App 的注入状态集中管理与展示；
- 定时检查代理连通性并自动告警。

如需上述能力，应在文档基础上另行开发工具，本文档可作为工具的逻辑依据。

---

## 八、给 Agent 的执行声明

Agent 收到本文档后：

1. **先完整阅读**本文档，理解每一步的目的与预期；
2. **严格按「执行流程」顺序执行**，每步核对预期结果，失败即停并按「失败处理」处理；
3. **完成「验收标准」全部自检**后再向用户报告结果；
4. **不得跳过备份、签名、验证**等关键步骤；
5. **遇到本文档未覆盖的异常**，停止执行，向用户说明现场情况并寻求指示，不得自行猜测操作。

---

*文档版本：v1.0 ｜ 适用平台：macOS（Apple Silicon / Intel）｜ 验证环境：macOS，Antigravity 2.34.0，代理 127.0.0.1:7892*
