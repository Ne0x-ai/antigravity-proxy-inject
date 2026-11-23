# Antigravity Proxy Inject

[中文版](./README.zh-CN.md)

An **AI-executable skill document** that routes **Antigravity** (Google's AI IDE) through your local proxy (Clash, etc.) on macOS — one document, zero manual setup.

Feed the document to your AI Agent, and it will automatically handle the entire workflow: proxy environment variable injection into the app, code-signing repair, and launch verification — no manual scripts, no repeated setup.

## The Problem

Antigravity's background service (`language_server`) on macOS is a Go process that **only honors `HTTP_PROXY`-style environment variables and ignores macOS system proxy settings**. The usual workaround is a `.command` launch script you have to run manually every time — tedious and easy to forget.

This project uses the official macOS mechanism `LSEnvironment`: it modifies the app's `Info.plist` to embed the proxy environment variables. When launched via GUI (Dock, Spotlight, `open`, or double-click), launchd injects them automatically — **the app goes through your proxy no matter how you open it**.

## Platform

**macOS only** (Apple Silicon / Intel, macOS 13+). Verified on macOS with Antigravity 2.34.0.

Relies on macOS-specific mechanisms: `LSEnvironment`, `.app` bundles, `codesign` signing, and `xattr` quarantine attributes. Not applicable to Windows / Linux.

> The skill document itself is written in Chinese. AI Agents process it fine regardless of your UI language.

## Usage

1. Give the `ProxyInject-技能文档.md` file (or its content) from this repo to your AI Agent;
2. Tell the Agent: "Follow this document and configure local proxy injection for Antigravity";
3. The Agent will execute in order: pre-flight checks → backup → LSEnvironment injection → quarantine cleanup → re-signing → launch & verify → self-inspection against acceptance criteria, fully automated;
4. The document includes complete acceptance criteria and a rollback procedure — one-command restore if anything fails.

## Highlights

- **Works with any launch method**: via `LSEnvironment` — Dock / Spotlight / `open` / double-click all carry the proxy
- **Safety guardrails**: built-in backup, pre-flight checks, and rollback on failure
- **Self-verifying**: 6 acceptance criteria the Agent validates automatically after execution
- **Battle-tested pitfalls**: re-signing, `xattr` cleanup, process restart, proxy reachability probe, re-injection after app updates — all documented from real runs
- **Extensible**: Antigravity is the primary example; other standard `.app` bundle apps can be adapted via the "Generalization" section of the skill document

## Repository Structure

```
antigravity-proxy-inject/
├── README.md                    # This file (English)
├── README.zh-CN.md              # Chinese README
├── LICENSE                      # MIT License
└── ProxyInject-技能文档.md      # AI Agent-executable skill document (core deliverable, in Chinese)
```

## Known Limitations

- **App auto-updates overwrite the config**: Electron app updates rewrite `Info.plist`, which invalidates the injection — re-run the skill document afterwards;
- **No state, no watch mode**: as a pure document, it cannot re-inject unattended after updates, nor manage multiple apps centrally (see the "State Management" section of the skill document);
- **Signature changes**: after injection the app uses an ad-hoc signature, losing its original release-signature verification.

## Disclaimer

This tool works by modifying a third-party app's bundle and signature, which may conflict with the app's terms of service, and app auto-updates will overwrite the configuration. Use at your own risk; the author is not liable for any consequences arising from the use of this tool.

## License

[MIT](./LICENSE)
