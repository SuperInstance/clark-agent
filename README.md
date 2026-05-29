# clark-agent

A small, typed, hookable agent loop — provider-agnostic, sandbox-agnostic, tooling-agnostic.

## What This Gives You

- **Typed agent loop** — `context → LLM → tool batch → results → repeat`, with clean termination semantics
- **Plugin system** — 6 capability traits (`BeforeToolCall`, `AfterToolCall`, `ContextTransform`, `EventObserver`, `SteeringSource`, `FollowUpSource`) for cross-cutting extension
- **Stream abstraction** — `StreamFn` trait: swap LLM providers, inject fixtures, replay scenarios, proxy remotely
- **Tool registry** — `AgentTool` trait + `ToolRegistry`; tools own their schema, validation, and execution
- **Builder API** — `AgentBuilder` composes stream, tools, plugins, and config in one call
- **Cancellation** — `CancellationToken` for graceful shutdown

## Shape

```
context → LLM (StreamFn) → tool batch → results appended → repeat
```

Termination is a tool decision (`ToolResult::terminate = true`, unanimous across the batch). The runtime owns execution and event emission; tools own semantics; plugins own cross-cutting extension.

## Quick Start

```rust
use std::sync::Arc;
use clark_agent::{AgentBuilder, AgentContext, AgentMessage, ToolRegistry, UserContent};
use tokio_util::sync::CancellationToken;

let registry = ToolRegistry::new()
    .with(Arc::new(my_shell_tool()))
    .with(Arc::new(my_file_tool()));

let config = AgentBuilder::new()
    .stream(Arc::new(my_provider()))
    .tools(registry)
    .before_tool_call(my_security_gate())
    .after_tool_call(my_repeat_detector())
    .context_transform(clark_agent::budget::TokenBudget::default())
    .max_iterations(50)
    .build()?;

let outcome = clark_agent::run(
    vec![AgentMessage::User {
        content: UserContent::Text("List files in /tmp".into()),
        timestamp: None,
    }],
    AgentContext::new("You are a helpful assistant."),
    &config,
    CancellationToken::new(),
).await;
```

## Layers

| Layer | Purpose |
|-------|---------|
| `types` | `AgentMessage`, content blocks, `StopReason` |
| `event` | `AgentEvent` enum + `EventSink` trait (ChannelSink, FanOutSink, NoopSink) |
| `tool` | `AgentTool` trait + `ToolRegistry` |
| `stream` | `StreamFn` trait — swappable LLM transport |
| `plugin` | `Plugin` + 6 capability traits |
| `config` | `LoopConfig` + `AgentBuilder` |
| `run` | `run` / `run_continue` — canonical loop |
| `exec` | Parallel + sequential tool dispatch, hook plumbing |
| `budget` | Token-budget context transform |

## Plugin Extension Points

| Trait | When it runs |
|-------|-------------|
| `BeforeToolCall` | After validation, before `tool.execute`. May block. |
| `AfterToolCall` | After `tool.execute`. May override result, vote terminate. |
| `ContextTransform` | Before each LLM call. Window management, redaction. |
| `EventObserver` | On every `AgentEvent`. Logging, telemetry, persistence. |
| `SteeringSource` | Between batches. Inject extra messages mid-run. |
| `FollowUpSource` | After natural stop. Re-start if more is queued. |

## Examples

```bash
# Minimal agent with fixture provider
cargo run --example minimal

# Tool call demo
cargo run --example tool_call
```

## How It Fits

- **[co-captain-git-agent](https://github.com/SuperInstance/co-captain-git-agent)** — Fleet liaison built on clark-agent's loop pattern
- **[cocapn-health-rs](https://github.com/SuperInstance/cocapn-health-rs)** — Health check tools registered in the tool registry
- **[capability-spec-rs](https://github.com/SuperInstance/capability-spec-rs)** — Capability specs define what tools an agent exposes
- **[categorical-agents](https://github.com/SuperInstance/categorical-agents)** — Category-theoretic composition maps to plugin composition

## Testing

26 integration tests covering agent loop, tool dispatch, plugin hooks, graceful turn limits, sandbox isolation, max token recovery, sibling abort, and message persistence round-trips.

```bash
cargo test
```

## Installation

```toml
[dependencies]
clark-agent = { git = "https://github.com/SuperInstance/clark-agent" }
```

```bash
git clone https://github.com/SuperInstance/clark-agent.git
cd clark-agent
cargo build
```

## License

MIT

Part of the [SuperInstance OpenConstruct](https://github.com/SuperInstance) ecosystem.
