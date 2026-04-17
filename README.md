# ACP adapter for Codex

[![npm](https://img.shields.io/npm/v/%40yoke233%2Fcodex-acp)](https://www.npmjs.com/package/@yoke233/codex-acp)

> **Fork note.** This fork of [zed-industries/codex-acp](https://github.com/zed-industries/codex-acp) adds a **`session/steer`** extension for mid-turn user-input injection. See the [Mid-turn steer](#mid-turn-steer-extension) section below. Everything else matches upstream.

Use [Codex](https://github.com/openai/codex) from [ACP-compatible](https://agentclientprotocol.com) clients such as [Zed](https://zed.dev)!

This tool implements an ACP adapter around the Codex CLI, supporting:

- Context @-mentions
- Images
- Tool calls (with permission requests)
- Following
- Edit review
- TODO lists
- Slash commands:
  - /review (with optional instructions)
  - /review-branch
  - /review-commit
  - /init
  - /compact
  - /logout
  - Custom Prompts
- Client MCP servers
- **Mid-turn steer** (this fork) — inject additional user input into a running turn without cancelling it
- Auth Methods:
  - ChatGPT subscription (requires paid subscription and doesn't work in remote projects)
  - CODEX_API_KEY
  - OPENAI_API_KEY

Learn more about the [Agent Client Protocol](https://agentclientprotocol.com/).

## How to use

### Zed

The latest version of Zed can already use the upstream adapter out of the box. To use this fork instead, point Zed's external-agent configuration at the binary from a release of this repo (or at `npx @yoke233/codex-acp`).

Read the docs on [External Agent](https://zed.dev/docs/ai/external-agents) support.

### Other clients

Or try it with any of the other [ACP compatible clients](https://agentclientprotocol.com/overview/clients)!

#### Installation

Install a prebuilt binary for your OS/arch from the [latest release](https://github.com/yoke233/codex-acp/releases):

```
OPENAI_API_KEY=sk-... codex-acp
```

Or via npm (automatically pulls the right platform binary as an optional dependency):

```
npm install -g @yoke233/codex-acp
# or without installing
npx @yoke233/codex-acp
```

## Mid-turn steer extension

ACP's `session/prompt` is a blocking request-response: once a prompt is active, its JSON-RPC channel is held open until the `PromptResponse` comes back, so a client can't send another prompt on the same session until the turn finishes. That rules out a common UX pattern — letting the user append a follow-up message *while the agent is still thinking or executing tools*.

Codex already ships the primitive for this internally: `Codex::steer_input`, which appends a `UserInput` to the active turn's `pending_input` queue so it's consumed at the next agent step. This fork exposes that primitive through an ACP extension notification.

### Capability

Advertised in the `initialize` response:

```jsonc
{
  "agentCapabilities": {
    "_meta": { "steer": true }
  }
}
```

Clients that also advertise support for `session/steer` may send the notification below; clients that don't know about it can ignore the capability.

### Wire format

`session/steer` is a **notification** (no response). It is delivered via the ACP extension-notification channel; the Rust ACP SDK encodes extension methods with a leading `_`, so on the wire the method name is `_session/steer`:

```jsonc
// Client → Agent (notification — no id, no response)
{
  "jsonrpc": "2.0",
  "method": "_session/steer",
  "params": {
    "sessionId": "uuid-xxx",
    "prompt": [
      { "type": "text", "text": "actually, use try/catch instead" }
    ]
  }
}
```

### Semantics

- If there is no active turn the notification is **silently ignored** (with a `warn!` log line). No error is returned.
- Injected content is appended to the active turn's `pending_input` queue; the Codex core picks it up at the next thinking step.
- The originating `session/prompt` is **not** affected: it keeps streaming `session/update` notifications and eventually returns its own `PromptResponse` with `stopReason: "end_turn"` (or `"cancelled"`).
- Turns of kind `review` and `compact` are not steerable — the adapter surfaces the underlying `SteerInputError` as a JSON-RPC internal error.
- Parameters accept the same `ContentBlock[]` shape as `session/prompt`.

## License

Apache-2.0
