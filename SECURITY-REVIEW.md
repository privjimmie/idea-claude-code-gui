# Security Review

**Date:** 2026-04-01
**Reviewed version:** 0.3.2 (commit 9958b78)
**Reviewed by:** Jimmie (with Claude Code)

## Summary

A thorough security audit was conducted on the [idea-claude-code-gui](https://github.com/zhukunpenglinyutong/idea-claude-code-gui) JetBrains plugin codebase. The audit covered dependencies, network communication, credential handling, and code integrity.

**Verdict: No evidence of credential theft, data exfiltration, or malicious behavior.**

---

## Dependencies

### Java (3 production dependencies)

| Library | Version | Publisher | License | Notes |
|---------|---------|-----------|---------|-------|
| gson | 2.10.1 | Google | Apache 2.0 | Industry standard JSON library |
| jlayer | 1.0.1 | javazoom | LGPL | Legacy MP3 decoder for notification sounds, no network code |
| snakeyaml-engine | 3.0.1 | SnakeYAML team | Apache 2.0 | Standard YAML parser |

### npm — Webview (11 production dependencies)

| Library | Publisher | Notes |
|---------|-----------|-------|
| react / react-dom | Meta | Standard UI framework |
| antd | Ant Group | Popular UI component library (93K+ GitHub stars) |
| marked / marked-highlight | marked team | Markdown rendering |
| highlight.js | highlight.js team | Syntax highlighting |
| dompurify | cure53 | HTML sanitization (security library) |
| mermaid | mermaid team | Diagram rendering |
| i18next / react-i18next | i18next team | Internationalization |
| @lobehub/icons | LobeHub | AI provider icon pack, lower usage (~30K weekly) but open source, SVG only |
| vconsole | Tencent | Dev-only debugging console, NOT included in production builds |

### npm — AI Bridge (1 production dependency)

| Library | Publisher | Notes |
|---------|-----------|-------|
| sql.js | sql.js team | SQLite compiled to WebAssembly, for local session storage |

### Repositories

- Java: **mavenCentral()** only (official Maven Central)
- npm: **npmjs.org** only (official npm registry)
- Gradle plugins: **Official JetBrains** and **Gradle** plugins only
- **No Chinese mirrors, private repos, or custom registries**

All dependencies are **open source**. No closed-source or proprietary libraries.

---

## Network Communication

### No unauthorized outbound connections

- Default API endpoint: `https://api.anthropic.com` (official Anthropic API)
- Only localhost HTTP call: Chrome DevTools debugging (`127.0.0.1`)
- **No telemetry, analytics, or tracking code** (no Segment, Mixpanel, Google Analytics, Sentry, etc.)
- **No hidden WebSocket connections**

### Chinese provider presets (opt-in only)

The plugin includes preset configurations for Chinese AI providers (Zhipu, Kimi, DeepSeek, Qwen, Xiaomi, MiniMax) in the Provider Settings UI. These are **never contacted by default** — the user must explicitly select them. They are convenience shortcuts for users of those services.

---

## Credential Handling

### Storage

- API keys are **never stored by the plugin** itself
- Keys are read from `~/.claude/settings.json` (user-managed) or via Claude SDK OAuth flow
- No use of JetBrains PasswordSafe or any plugin-level credential storage
- Keys passed to child Node.js process via environment variables (not CLI arguments)

### Log sanitization

Active log sanitization in `ClaudeLogSanitizer.java` redacts patterns matching `api_key`, `token`, `secret`, `password`, `credentials`, etc. Keys are masked in UI display (first/last 4 chars only).

### Proxy and TLS

- Supports corporate proxy via `HTTP_PROXY`/`HTTPS_PROXY` from `~/.claude/settings.json`
- Custom CA certs via `NODE_EXTRA_CA_CERTS`
- TLS disable (`NODE_TLS_REJECT_UNAUTHORIZED=0`) triggers a security warning
- No plugin-controlled proxies or interceptors

---

## Code Integrity

- **No Base64-encoded URLs or hidden endpoints**
- **No eval() in plugin code** (only in bundled Babel library, which is standard)
- **No native libraries** (.dll/.so/.dylib) — pure Java/JavaScript
- **No hidden files or suspicious binaries** in the repo
- **No obfuscated code**
- Reflection usage is justified and documented (JetBrains internal API access)
- Shell execution uses `ProcessBuilder` with command whitelisting and timeouts
- Path validation prevents directory traversal attacks
- Permission system separates tools into READ_ONLY, AUTO_ALLOW, EDIT, EXECUTION categories

---

## SDK Dependencies (runtime install)

When the plugin prompts to install "Claude Code SDK", it runs `npm install` to fetch from the **official npm registry**:

| Package | Publisher |
|---------|-----------|
| @anthropic-ai/claude-agent-sdk | Anthropic (official) |
| @anthropic-ai/sdk | Anthropic (official) |
| @anthropic-ai/bedrock-sdk | Anthropic (official) |

Installed to `~/.codemoss/dependencies/claude-sdk/node_modules/`.

---

## Conclusion

The plugin is a legitimate, well-engineered open source project with:

- Minimal, mainstream dependencies from official registries only
- No unauthorized network activity or telemetry
- Proper credential handling with log sanitization
- Transparent behavior documented in plugin.xml
- Robust permission and path safety controls

**Risk level: LOW** — safe to use.
