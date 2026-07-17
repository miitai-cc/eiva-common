# eiva-common

Common code shared across the Eiva project.

## rusty_claw

[rusty_claw](https://docs.rs/rusty_claw) lets you build Claude-powered agents in Rust. It supports
bidirectional JSONL transport over stdio, the Claude Control Protocol (CCP), Model Context Protocol
(MCP) tool integration, a hook system for lifecycle events, and procedural macros (`#[claw_tool]`) for
ergonomic tool definitions.

### Installation

Add the crate to your `Cargo.toml`:

```toml
[dependencies]
rusty_claw = "0.1"
tokio = { version = "1", features = ["full"] }
tokio-stream = "0.1"
```

### One-shot query

The `query()` function is the simplest entry point. It creates the agentic loop and returns a stream
of messages that you consume as Claude works:

```rust
use rusty_claw::prelude::*;
use tokio_stream::StreamExt;

#[tokio::main]
async fn main() -> Result<(), ClawError> {
    let options = ClaudeAgentOptions::builder()
        .allowed_tools(vec!["Read".into(), "Edit".into(), "Bash".into()])
        .permission_mode(PermissionMode::AcceptEdits)
        .build();

    let mut stream = query("Find and fix the bug in auth.py", Some(options)).await?;

    while let Some(message) = stream.next().await {
        match message? {
            Message::Assistant(msg) => {
                for block in msg.message.content {
                    if let ContentBlock::Text { text } = block {
                        println!("{}", text);
                    }
                }
            }
            Message::Result(ResultMessage::Success { result, .. }) => {
                println!("Done: {}", result);
            }
            _ => {}
        }
    }
    Ok(())
}
```

### Configuration (`ClaudeAgentOptions`)

`ClaudeAgentOptions::builder()` exposes the agent configuration. Common fields:

| Field | Description |
|-------|-------------|
| `allowed_tools` | Tools the agent may use (e.g. `Read`, `Edit`, `Glob`, `Bash`, `Grep`). |
| `permission_mode` | Default policy when no allow/deny rule matches. |
| `system_prompt` | Custom system prompt via `SystemPrompt::Custom(...)`. |
| `max_turns` | Limit the number of conversation turns. |
| `model` | Override the model (e.g. `"claude-haiku-4-5"`). |

#### Permission modes

| Mode | Variant | Behavior |
|------|---------|----------|
| `default` | `PermissionMode::Default` | No auto-approvals; relies on handler callback. |
| `acceptEdits` | `PermissionMode::AcceptEdits` | Auto-approve file edit operations. |
| `bypassPermissions` | `PermissionMode::BypassPermissions` | Skip all permission checks. |
| `plan` | `PermissionMode::Plan` | Planning mode; no tool execution. |
| `allow` | `PermissionMode::Allow` | Allow all tools without prompting. |
| `ask` | `PermissionMode::Ask` | Prompt user for each tool use. |
| `deny` | `PermissionMode::Deny` | Deny all tools by default. |
| `custom` | `PermissionMode::Custom` | Use a custom hook-based decision. |

### Defining custom tools with `#[claw_tool]`

The `#[claw_tool]` attribute macro turns a plain `async fn` into an MCP tool. The macro auto-generates
a `ToolHandler` impl and a builder function, and derives JSON Schema from the function signature
(`Option<T>` params become optional; everything else is required).

```rust
use rusty_claw::prelude::*;
use rusty_claw::mcp_server::ToolResult;

#[claw_tool(name = "word_count", description = "Count words in a text string")]
async fn word_count(text: String) -> ToolResult {
    let count = text.split_whitespace().count();
    ToolResult::text(format!("{} words", count))
}

// Optional parameters use Option<T> — Claude can omit them.
#[claw_tool(name = "repeat", description = "Repeat a message N times")]
async fn repeat(message: String, times: Option<i32>) -> ToolResult {
    let n = times.unwrap_or(1) as usize;
    let output = std::iter::repeat_n(message, n).collect::<Vec<_>>().join("\n");
    ToolResult::text(output)
}
```

Register the tools with an MCP server and expose them to the agent:

```rust
use std::sync::Arc;
use rusty_claw::prelude::*;

#[tokio::main]
async fn main() -> Result<(), ClawError> {
    let mut server = SdkMcpServerImpl::new("text_tools", "1.0.0");
    server.register_tool(word_count());
    server.register_tool(repeat());

    let mut registry = SdkMcpServerRegistry::new();
    registry.register(server);

    let options = ClaudeAgentOptions::builder()
        .max_turns(5)
        .permission_mode(PermissionMode::BypassPermissions)
        .sdk_mcp_servers(vec![rusty_claw::options::SdkMcpServer {
            name: "text_tools".to_string(),
            version: "1.0.0".to_string(),
        }])
        .build();

    let mut client = ClaudeClient::new(options)?;
    client.register_mcp_message_handler(Arc::new(registry)).await;
    client.connect().await?;

    let mut stream = client.send_message("Count the words in 'hello world foo bar'").await?;
    while let Some(result) = stream.next().await {
        // handle messages...
    }
    client.close().await?;
    Ok(())
}
```

### Message types

Each item yielded by the stream is a `Message`. Common variants:

| Variant | Contents |
|---------|----------|
| `ContentBlock::Text { text }` | Claude's reasoning and explanations. |
| `ContentBlock::ToolUse { id, name, input }` | A tool invocation request. |
| `ContentBlock::ToolResult { tool_use_id, content, is_error }` | Result from a tool execution. |
| `ContentBlock::Thinking { thinking }` | Extended thinking tokens. |

### Further reading

- [API reference](https://docs.rs/rusty_claw)
- [Examples](https://github.com/citadelgrad/rusty_claw/tree/main/examples)
- [Hooks guide](https://github.com/citadelgrad/rusty_claw/blob/main/docs/HOOKS.md)
- [Permissions guide](https://github.com/citadelgrad/rusty_claw/blob/main/docs/PERMISSIONS.md)
