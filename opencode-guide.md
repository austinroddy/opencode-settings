# OpenCode Enterprise / Air-Gapped Setup Guide

This guide explains how to configure OpenCode for a secure enterprise or air-gapped environment where:

- No external AI providers are allowed
- No company code leaves except through your approved internal endpoint
- No automatic downloads should occur
- No plugins or MCP servers should auto-connect

---

## 1. Recommended Architecture

```text
Developer Workstation
        ↓
OpenCode
        ↓
Internal OpenAI-Compatible Gateway
        ↓
Internal Models / Bedrock / vLLM / Ollama
```

Only the internal endpoint should be reachable.

---

## 2. OpenCode Configuration Layers

OpenCode merges configuration from multiple locations.

Priority, conceptually:

1. Global user config
2. Custom config through `OPENCODE_CONFIG`
3. Project config: `opencode.json`
4. Managed system config
5. macOS MDM managed preferences

Recommended use:

- Use **project config** for repo-specific defaults.
- Use **managed system config** for company-wide restrictions.
- Use **firewall/network policy** as the real enforcement boundary.

Official docs:

```text
https://opencode.ai/docs/config/
https://opencode.ai/docs/cli/
https://opencode.ai/docs/enterprise/
```

---

## 3. Project-Level `opencode.json`

Place this file in:

```text
<repo-root>/opencode.json
```

Recommended config:

```json
{
  "$schema": "https://opencode.ai/config.json",

  "autoupdate": false,

  "share": "disabled",

  "plugin": [],

  "mcp": {},

  "enabled_providers": ["internal"],

  "disabled_providers": [
    "zen",
    "opencode",
    "openai",
    "anthropic",
    "google",
    "gemini",
    "openrouter",
    "github-copilot",
    "llmgateway"
  ],

  "permission": {
    "bash": "ask",
    "edit": "allow",
    "webfetch": "deny",
    "websearch": "deny"
  },

  "watcher": {
    "ignore": [
      ".git/**",
      "node_modules/**",
      "dist/**",
      "build/**",
      ".next/**",
      ".turbo/**",
      ".cache/**"
    ]
  },

  "provider": {
    "internal": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Internal Gateway",
      "options": {
        "baseURL": "{env:AI_BASE_URL}",
        "apiKey": "{env:AI_API_KEY}"
      },
      "models": {
        "{env:AI_MODEL}": {}
      }
    }
  },

  "model": "internal/{env:AI_MODEL}",
  "small_model": "internal/{env:AI_MODEL}"
}
```

---

## 4. What These Settings Do

### Disable Auto Updates

```json
"autoupdate": false
```

Prevents OpenCode from automatically downloading updates.

### Disable Sharing

```json
"share": "disabled"
```

Prevents conversation/session sharing.

### Disable Plugins

```json
"plugin": []
```

Prevents plugin loading from config.

### Disable MCP

```json
"mcp": {}
```

Disables MCP servers by default.

### Restrict Providers

```json
"enabled_providers": ["internal"]
```

Only the internal provider should be enabled.

```json
"disabled_providers": [
  "zen",
  "opencode",
  "openai",
  "anthropic",
  "google",
  "gemini",
  "openrouter",
  "github-copilot",
  "llmgateway"
]
```

Explicitly blocks known external providers.

### Disable Web Tools

```json
"webfetch": "deny",
"websearch": "deny"
```

Prevents the agent from fetching webpages or performing web searches.

### Force Small Model to Internal Endpoint

```json
"small_model": "internal/{env:AI_MODEL}"
```

This matters because small/lightweight tasks should also use the internal model.

---

## 5. Required Environment Variables

Set these before launching OpenCode:

```bash
export AI_BASE_URL="https://your-internal-endpoint/v1"
export AI_API_KEY="YOUR_API_KEY"
export AI_MODEL="YOUR_MODEL_NAME"

export OPENCODE_DISABLE_AUTOUPDATE=1
export OPENCODE_DISABLE_DEFAULT_PLUGINS=1
export OPENCODE_DISABLE_MODELS_FETCH=1
export OPENCODE_DISABLE_LSP_DOWNLOAD=1

export OTEL_SDK_DISABLED=true
```

### What They Do

```bash
OPENCODE_DISABLE_AUTOUPDATE=1
```

Disables automatic update checks.

```bash
OPENCODE_DISABLE_DEFAULT_PLUGINS=1
```

Disables bundled/default plugins.

```bash
OPENCODE_DISABLE_MODELS_FETCH=1
```

Disables fetching model/provider metadata from remote sources.

```bash
OPENCODE_DISABLE_LSP_DOWNLOAD=1
```

Disables automatic LSP server downloads.

```bash
OTEL_SDK_DISABLED=true
```

Disables OpenTelemetry SDK behavior.

---

## 6. Machine-Wide Managed Config

Managed configs override user configs.

### Linux

```text
/etc/opencode/opencode.json
```

### macOS

```text
/Library/Application Support/opencode/opencode.json
```

Recommended machine-wide config:

```json
{
  "$schema": "https://opencode.ai/config.json",

  "autoupdate": false,

  "share": "disabled",

  "plugin": [],

  "mcp": {},

  "enabled_providers": ["internal"],

  "disabled_providers": [
    "zen",
    "opencode",
    "openai",
    "anthropic",
    "google",
    "gemini",
    "openrouter",
    "github-copilot",
    "llmgateway"
  ],

  "permission": {
    "bash": "ask",
    "webfetch": "deny",
    "websearch": "deny"
  }
}
```

---

## 7. Machine-Wide Environment Variables

### Linux

Create:

```text
/etc/profile.d/opencode.sh
```

Contents:

```bash
export OPENCODE_DISABLE_AUTOUPDATE=1
export OPENCODE_DISABLE_DEFAULT_PLUGINS=1
export OPENCODE_DISABLE_MODELS_FETCH=1
export OPENCODE_DISABLE_LSP_DOWNLOAD=1
export OTEL_SDK_DISABLED=true
```

### macOS

Deploy through one of:

- MDM
- Jamf
- Kandji
- `/etc/zprofile`
- `/etc/zshrc`
- managed shell profile tooling

---

## 8. Clear Existing Provider/Auth State

Before switching to enterprise configs:

```bash
rm -rf ~/.local/share/opencode
rm -rf ~/.cache/opencode
```

This removes:

- cached provider metadata
- cached auth
- stale model/provider state

This is important because stale providers can survive config changes.

---

## 9. Recommended Firewall / Network Controls

Application config alone is not enough.

Enterprise-grade security should also enforce outbound network controls.

Allow only:

- internal AI endpoint
- internal DNS
- internal package mirrors, if needed

Block:

- public AI providers
- public DNS
- arbitrary outbound HTTPS

---

## 10. Linux `nftables` Example

Replace:

- `10.0.0.53` with your internal DNS
- `10.20.30.40` with your internal AI gateway

```bash
sudo nft flush ruleset

sudo nft add table inet filter

sudo nft add chain inet filter output \
'{ type filter hook output priority 0 ; policy drop ; }'

sudo nft add rule inet filter output ip daddr 127.0.0.1 accept
sudo nft add rule inet filter output ct state established,related accept

sudo nft add rule inet filter output ip daddr 10.0.0.53 udp dport 53 accept
sudo nft add rule inet filter output ip daddr 10.0.0.53 tcp dport 53 accept

sudo nft add rule inet filter output ip daddr 10.20.30.40 tcp dport 443 accept
```

---

## 11. Verification Steps

### Verify Model Picker

OpenCode should only show:

- your internal provider/model

It should not show:

- Zen
- OpenAI
- Anthropic
- Gemini
- OpenRouter
- GitHub Copilot

### Verify Network Traffic

Use:

```bash
tcpdump -nn
```

or:

```bash
ss -tpn
```

Expected traffic:

- internal endpoint
- localhost

Unexpected traffic:

- `opencode.ai`
- `openai.com`
- `anthropic.com`
- `openrouter.ai`
- `googleapis.com`

---

## 12. Recommended Enterprise Defaults

| Setting | Recommended |
|---|---|
| `autoupdate` | `false` |
| `share` | `disabled` |
| `plugin` | `[]` |
| `mcp` | `{}` |
| `webfetch` | `deny` |
| `websearch` | `deny` |
| `bash` | `ask` |
| `edit` | `allow` |
| model fetch | disabled by env var |
| LSP download | disabled by env var |

---

## 13. Final Recommendation

The strongest deployment posture is:

```text
OpenCode JSON restrictions
+
Machine-wide managed config
+
System-wide environment variables
+
Network egress controls
```

That combination gives:

- centralized enforcement
- repeatable setup
- reduced accidental leakage risk
- no external AI providers
- no downloads except what the company explicitly allows
