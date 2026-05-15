markdown # OpenCode Enterprise / Air-Gapped Setup Guide  This guide explains how to configure OpenCode for a secure enterprise or air-gapped environment where:  - No external AI providers are allowed - No company code leaves except through your approved internal endpoint - No automatic downloads should occur - No plugins or MCP servers should auto-connect - Developers should have a mostly standardized setup  This setup is based on: - Official OpenCode documentation - Official CLI/config documentation - Runtime validation/testing  ---  # 1. Recommended Architecture  text
Developer Workstation
        ↓
OpenCode
        ↓
Internal OpenAI-Compatible Gateway
        ↓
Internal Models / Bedrock / vLLM / Ollama
 ONLY the internal endpoint should be reachable.  ---  # 2. OpenCode Configuration Layers  OpenCode merges configuration from multiple locations.  Priority (highest to lowest):  1. Remote organizational defaults 2. Global user config 3. Custom config (`OPENCODE_CONFIG`) 4. Project config (`opencode.json`) 5. `.opencode` directories 6. Inline config (`OPENCODE_CONFIG_CONTENT`) 7. Managed config 8. macOS MDM managed preferences  Important: - Managed configs override everything else - Project configs override user configs - Project configs are ideal for repo-standardized behavior  Official docs: https://opencode.ai/docs/config/  ---  # 3. Recommended Enterprise Layout  ## Project Repository text
your-repo/
├── opencode.json
├── README.md
├── scripts/
│   ├── setup.sh
│   └── verify.sh
└── .gitignore
 ---  # 4. Personal / Project-Level opencode.json  Place this file in: text
<repo-root>/opencode.json
 Recommended config: json
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
      ".git/",
      "node_modules/",
      "dist/",
      "build/",
      ".next/",
      ".turbo/",
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
 ---  # 5. Why These Settings Matter  ## Disable Auto Updates json
"autoupdate": false
 Prevents automatic update downloads.  ---  ## Disable Sharing json
"share": "disabled"
 Prevents conversation sharing features.  ---  ## Disable Plugins json
"plugin": []
 Prevents plugin loading.  ---  ## Disable MCP json
"mcp": {}
 Disables MCP servers by default.  ---  ## Restrict Providers json
"enabled_providers": ["internal"]
 Only your internal provider becomes usable.  --- json
"disabled_providers": [...]
 Explicitly disables: - OpenAI - Anthropic - Gemini - OpenRouter - Zen - GitHub Copilot - etc.  This is important because OpenCode supports many providers by default.  ---  ## Disable Web Access json
"webfetch": "deny",
"websearch": "deny"
 Prevents the agent from: - fetching webpages - performing searches  ---  ## small_model Override json
"small_model": "internal/{env:AI_MODEL}"
 VERY IMPORTANT.  Without this, lightweight tasks may try to use another provider.  This ensures: - title generation - lightweight operations - metadata tasks  still use your internal model.  ---  # 6. Required Environment Variables  These are officially documented runtime flags.  Recommended setup: bash
export AI_BASE_URL="https://your-internal-endpoint/v1"
export AI_API_KEY="YOUR_API_KEY"
export AI_MODEL="YOUR_MODEL_NAME"

export OPENCODE_DISABLE_AUTOUPDATE=1
export OPENCODE_DISABLE_DEFAULT_PLUGINS=1
export OPENCODE_DISABLE_MODELS_FETCH=1
export OPENCODE_DISABLE_LSP_DOWNLOAD=1

export OTEL_SDK_DISABLED=true
 ---  # 7. What These Environment Variables Do  ## Disable Update Checks bash
OPENCODE_DISABLE_AUTOUPDATE=1
 Prevents update checks/downloads.  ---  ## Disable Default Plugins bash
OPENCODE_DISABLE_DEFAULT_PLUGINS=1
 Prevents bundled/default plugins from loading.  ---  ## Disable Remote Model Metadata Fetches bash
OPENCODE_DISABLE_MODELS_FETCH=1
 VERY IMPORTANT.  Prevents: - provider metadata downloads - model catalog refreshes - external model/provider discovery  This is likely what prevented Zen/provider refresh behavior during testing.  ---  ## Disable Automatic LSP Downloads bash
OPENCODE_DISABLE_LSP_DOWNLOAD=1
 Prevents automatic language server downloads.  ---  ## Disable OpenTelemetry SDK bash
OTEL_SDK_DISABLED=true
 Disables OpenTelemetry instrumentation globally.  ---  # 8. Machine-Wide Managed Config  Managed configs override user configs.  ## Linux  Place config in: text
/etc/opencode/opencode.json
 ---  ## macOS  Place config in: text
/Library/Application Support/opencode/opencode.json
 Managed configs require admin/root access and cannot easily be overridden by users.  Recommended machine-wide config: json
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
 ---  # 9. Machine-Wide Environment Variables  ## Linux  Create: text
/etc/profile.d/opencode.sh
 Contents: bash
export OPENCODE_DISABLE_AUTOUPDATE=1
export OPENCODE_DISABLE_DEFAULT_PLUGINS=1
export OPENCODE_DISABLE_MODELS_FETCH=1
export OPENCODE_DISABLE_LSP_DOWNLOAD=1

export OTEL_SDK_DISABLED=true
 ---  ## macOS  Can be deployed through: - MDM - Jamf - Kandji - shell profiles - `/etc/zprofile` - `/etc/zshrc`  ---  # 10. Clear Existing Provider/Auth State  IMPORTANT.  OpenCode caches provider state.  Before switching to enterprise configs: bash
rm -rf ~/.local/share/opencode
rm -rf ~/.cache/opencode
 This removes: - cached providers - cached auth - stale model metadata  This matters because stale providers can survive config changes.  ---  # 11. Recommended Firewall / Network Controls  Application config alone is NOT enough.  Enterprise-grade security should also enforce:  ## Allow ONLY:  - Internal AI endpoint - Internal DNS - Internal package mirrors (optional)  ## Block:  - Public AI providers - Public DNS - Arbitrary outbound HTTPS  ---  # 12. Recommended Linux nftables Example bash
sudo nft flush ruleset

sudo nft add table inet filter

sudo nft add chain inet filter output 
'{ type filter hook output priority 0 ; policy drop ; }'

# localhost
sudo nft add rule inet filter output ip daddr 127.0.0.1 accept

# established connections
sudo nft add rule inet filter output ct state established,related accept

# internal DNS
sudo nft add rule inet filter output ip daddr 10.0.0.53 udp dport 53 accept
sudo nft add rule inet filter output ip daddr 10.0.0.53 tcp dport 53 accept

# internal AI gateway
sudo nft add rule inet filter output ip daddr 10.20.30.40 tcp dport 443 accept
 Replace: - `10.0.0.53` with your internal DNS - `10.20.30.40` with your internal AI gateway  ---  # 13. Recommended Verification Steps  ## Verify Model Picker  OpenCode should ONLY show: - your internal provider  No: - Zen - OpenAI - Anthropic - Gemini - etc.  ---  ## Verify Network Traffic bash
tcpdump -nn
 or: bash
ss -tpn
 You should ONLY see: - internal endpoint traffic - localhost traffic  You should NOT see: - opencode.ai - openai.com - anthropic.com - openrouter.ai  ---  # 14. Recommended Enterprise Defaults  | Setting | Recommended | |---|---| | autoupdate | false | | plugins | disabled | | MCP | disabled | | webfetch | deny | | websearch | deny | | bash | ask | | edit | allow | | models fetch | disabled | | LSP download | disabled |  ---  # 15. Final Recommendation  The strongest deployment posture is:  ## OpenCode Restrictions + ## Managed System Config + ## Network Egress Controls  That combination gives: - centralized enforcement - reproducibility - operational safety - reduced accidental leakage risk  while still allowing developers to use your approved internal AI infrastructure.
:::
