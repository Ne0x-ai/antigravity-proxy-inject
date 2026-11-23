# Antigravity Proxy Inject

[English](./README.md)

让 macOS 上的 **Antigravity**（Google AI IDE）走本地代理（Clash 等）的**AI 可执行技能文档**——一份文档，全程自动，无需手动脚本。

把本文档喂给你的 AI Agent，它就能自动完成目标 App 的代理环境变量注入、签名修复、启动验证全流程——无需手动写脚本、无需每次重复操作。

## 解决的问题

macOS 上 Antigravity 的后台服务（`language_server`）是 Go 进程，**只认 `HTTP_PROXY` 等环境变量，不读系统代理设置**。常规做法是写 `.command` 脚本手动启动，繁琐且容易忘。

本方案利用 macOS 官方机制 `LSEnvironment`：直接修改目标 App 的 `Info.plist`，把代理环境变量写入，GUI 启动（Dock、调度台、`open` 命令）时由 launchd 自动注入，**任意方式打开 App 都自动走代理**。

## 适用平台

**仅 macOS**（Apple Silicon / Intel，macOS 13+）。已在 macOS + Antigravity 2.34.0 环境实测通过。

依赖 macOS 专有机制：`LSEnvironment`、`.app` bundle、`codesign` 签名、`xattr` 隔离属性，Windows / Linux 不适用。

## 使用方法

1. 将本仓库的 `ProxyInject-技能文档.md` 内容（或文件）交给你的 AI Agent；
2. 对 Agent 说："按这份文档帮我给 Antigravity 配置本地代理注入"；
3. Agent 会依次执行：前置检查 → 备份 → 注入 LSEnvironment → 清理隔离属性 → 重新签名 → 启动验证 → 自检验收，全程自动完成；
4. 文档内含完整的验收标准与回滚流程，失败可一键还原。

## 核心特性

- **任意启动方式生效**：改 `LSEnvironment`，从 Dock / 调度台 / `open` 命令打开都自动带代理
- **安全护栏**：内置备份、执行前检查、失败回滚，避免误操作
- **可自检**：6 条验收标准，Agent 执行完自动验证
- **踩坑全覆盖**：重签名、`xattr` 清理、退出重开、代理连通探测、更新后重做等实际坑位全部写明
- **可延伸**：以 Antigravity 为主示例，其他标准 `.app` bundle 的 App 可参照技能文档「通用化」一节适配

## 目录结构

```
antigravity-proxy-inject/
├── README.md                    # 英文说明（项目主入口）
├── README.zh-CN.md              # 本文件（中文说明）
├── LICENSE                      # MIT 开源协议
└── ProxyInject-技能文档.md      # AI Agent 可执行的技能文档（核心交付物）
```

## 已知限制

- **App 自动更新会覆盖配置**：Electron 类应用更新会重写 `Info.plist`，注入失效，需重新执行技能文档；
- **不维护状态、不自动监听**：文档模式无法在更新后无人值守重做，也无多 App 集中管理（详见技能文档「状态管理」一节）；
- **签名改变**：注入后 App 使用 ad-hoc 签名，失去原始发布签名校验。

## 免责声明

本工具通过修改第三方 App 的 bundle 与签名实现功能，可能与对应 App 的服务条款存在冲突，且 App 自动更新后配置会被覆盖。请自行评估风险后使用，作者不对因使用本工具造成的任何后果承担责任。

## 开源协议

[MIT](./LICENSE)
