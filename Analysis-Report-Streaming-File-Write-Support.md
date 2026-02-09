# Analysis Report: Streaming File Write Support for rust-mcp-filesystem

## Project Overview

rust-mcp-filesystem is a blazingly fast, asynchronous, and lightweight MCP (Model Context Protocol) server designed for efficient handling of various filesystem operations. This project is a pure Rust rewrite of the JavaScript-based `@modelcontextprotocol/server-filesystem`, offering enhanced capabilities, improved performance, and a robust feature set.

## Current Write File Implementation Analysis

### 1. Existing Implementation

The current `write_file` tool implementation is located in the following files:

- **Tool Definition**: `src/tools/write_file.rs`
- **Service Implementation**: `src/fs_service/io/write.rs`

#### Core Code Flow:

```rust
// src/tools/write_file.rs
pub async fn run_tool(
    params: Self,
    context: &FileSystemService,
) -> std::result::Result<CallToolResult, CallToolError> {
    context
        .write_file(Path::new(&params.path), &params.content)
        .await
        .map_err(CallToolError::new)?;

    Ok(CallToolResult::text_content(vec![TextContent::from(
        format!("Successfully wrote to {}", &params.path),
    )]))
}

// src/fs_service/io/write.rs
pub async fn write_file(&self, file_path: &Path, content: &String) -> ServiceResult<()> {
    let allowed_directories = self.allowed_directories().await;
    let valid_path = self.validate_path(file_path, allowed_directories)?;
    tokio::fs::write(valid_path, content).await?;
    Ok(())
}
```

#### Current Implementation Characteristics:

1. **Single Write Operation**: Uses `tokio::fs::write()` to write entire content at once
2. **Single Response**: Returns a simple success message after write completion
3. **No Progress Feedback**: No intermediate progress notifications during write
4. **Synchronous Return**: While using async I/O, returns a complete response to the client

## Feasibility Analysis

### 1. Streaming Output Feasibility

#### Technical Perspective:

**Advantages**:
- ✅ Rust has strong support for streaming I/O (via `tokio::io::AsyncWrite`, etc.)
- ✅ Project already uses tokio async runtime, providing async streaming foundation
- ✅ Streaming processing examples exist in `src/fs_service/io/read.rs`:
  ```rust
  use futures::{StreamExt, stream};
  let results = stream::iter(paths)
      .map(|path| async move { ... })
      .buffer_unordered(max_concurrent)
      .collect::<Vec<_>>()
      .await;
  ```

**Challenges**:
- ❌ **MCP Protocol Limitations**: MCP (Model Context Protocol) is based on JSON-RPC 2.0, with tool calls following request-response pattern
- ❌ **SDK Limitations**: `rust-mcp-sdk` (v0.8.1) `CallToolResult` type only supports single response, not streaming
- ❌ **Protocol Design**: From `src/handler.rs` and `src/server.rs`, all tool calls return a single `CallToolResult`

#### MCP Protocol Architecture Analysis:

```rust
// Tool call handling in src/handler.rs
async fn handle_call_tool_request(
    &self,
    params: CallToolRequestParams,
    _: Arc<dyn McpServer>,
) -> std::result::Result<CallToolResult, CallToolError> {
    // Returns single CallToolResult
}
```

MCP Protocol Communication Patterns:
- **Tools**: Request-response pattern, returns `CallToolResult`
- **Notifications**: One-way messages, but not suitable for tool responses
- **Resources**: Used for reading data, but currently not enabled in project
- **Logging**: Can send messages to stderr, but not a standard tool response channel

### 2. Line-Level Output Feasibility

#### Implementation Approaches:

**Approach A: Include Line-Level Information in Single Response**

Feasibility: ⭐⭐⭐⭐⭐ (Highly Feasible)

Implementation:
```rust
pub async fn write_file_with_line_info(
    &self, 
    file_path: &Path, 
    content: &String
) -> ServiceResult<String> {
    let allowed_directories = self.allowed_directories().await;
    let valid_path = self.validate_path(file_path, allowed_directories)?;
    
    let lines: Vec<&str> = content.lines().collect();
    let total_lines = lines.len();
    let bytes = content.as_bytes().len();
    
    tokio::fs::write(valid_path, content).await?;
    
    Ok(format!(
        "Successfully wrote to {}\n\
         Lines written: {}\n\
         Bytes written: {}\n\
         First line: {}\n\
         Last line: {}",
        file_path.display(),
        total_lines,
        bytes,
        lines.first().unwrap_or(&""),
        lines.last().unwrap_or(&"")
    ))
}
```

**Approach B: Use stderr Messages for Progress Notifications**

Feasibility: ⭐⭐⭐⭐ (Feasible but Limited)

From the code, MCP server supports sending log messages via `runtime.stderr_message()`:

```rust
// Example from src/handler.rs
let _ = runtime.stderr_message("Client does not support MCP Roots...").await;
```

Can send progress messages during write:
```rust
pub async fn write_file_with_progress(
    &self,
    file_path: &Path,
    content: &String,
    runtime: Arc<dyn McpServer>,
) -> ServiceResult<()> {
    let allowed_directories = self.allowed_directories().await;
    let valid_path = self.validate_path(file_path, allowed_directories)?;
    
    let lines: Vec<&str> = content.lines().collect();
    let total_lines = lines.len();
    
    // Send start message
    let _ = runtime.stderr_message(
        format!("Starting to write {} lines to {}", total_lines, file_path.display())
    ).await;
    
    // Write file
    tokio::fs::write(valid_path, content).await?;
    
    // Send completion message
    let _ = runtime.stderr_message(
        format!("Completed writing {} lines to {}", total_lines, file_path.display())
    ).await;
    
    Ok(())
}
```

**Limitations**:
- stderr messages are primarily for debugging and diagnostic information
- Not a standard output channel for tool responses
- Clients may not display or may display these messages differently

**Approach C: Chunked Write with Progress Reporting**

Feasibility: ⭐⭐⭐ (Technically Feasible but Architecture Unsupported)

For large files, can implement chunked writing:
```rust
use tokio::io::AsyncWriteExt;

pub async fn write_file_chunked(
    &self,
    file_path: &Path,
    content: &String,
    runtime: Arc<dyn McpServer>,
) -> ServiceResult<()> {
    let allowed_directories = self.allowed_directories().await;
    let valid_path = self.validate_path(file_path, allowed_directories)?;
    
    let mut file = tokio::fs::File::create(valid_path).await?;
    let lines: Vec<&str> = content.lines().collect();
    let total_lines = lines.len();
    
    for (idx, line) in lines.iter().enumerate() {
        file.write_all(line.as_bytes()).await?;
        file.write_all(b"\n").await?;
        
        // Report progress every 100 lines
        if (idx + 1) % 100 == 0 {
            let _ = runtime.stderr_message(
                format!("Progress: {}/{} lines written", idx + 1, total_lines)
            ).await;
        }
    }
    
    file.flush().await?;
    Ok(())
}
```

## Technical Limitations Summary

### MCP Protocol Limitations

1. **Request-Response Pattern**: MCP protocol based on JSON-RPC 2.0, uses strict request-response pattern
2. **No Native Streaming Support**: Protocol specification doesn't define streaming response mechanism for tool calls
3. **Single Response Object**: Each tool call must return a complete `CallToolResult`

### SDK Architecture Limitations

1. **CallToolResult Structure**: 
   ```rust
   pub struct CallToolResult {
       pub content: Vec<Content>,
       pub is_error: Option<bool>,
       pub meta: Option<HashMap<String, Value>>,
   }
   ```
   Designed for single response, doesn't support multiple returns

2. **Handler Interface**: `handle_call_tool_request` method in `ServerHandler` trait returns single `Result<CallToolResult, CallToolError>`

3. **No Progress Reporting Mechanism**: SDK doesn't provide standard progress reporting API

## Recommended Implementation Approaches

### Approach 1: Enhanced Response Content (Recommended ⭐⭐⭐⭐⭐)

**Implementation**:
After successful file write, return response with detailed information including:
- Total line count
- Total byte count
- Preview of first and last few lines
- Write timestamp
- File path

**Advantages**:
- ✅ Fully compliant with MCP protocol specification
- ✅ No SDK or protocol modifications needed
- ✅ Simple implementation, low risk
- ✅ Good client compatibility

**Example Code**:
```rust
pub async fn write_file_verbose(
    &self, 
    file_path: &Path, 
    content: &String
) -> ServiceResult<String> {
    let start_time = std::time::Instant::now();
    let allowed_directories = self.allowed_directories().await;
    let valid_path = self.validate_path(file_path, allowed_directories)?;
    
    // Analyze content
    let lines: Vec<&str> = content.lines().collect();
    let total_lines = lines.len();
    let total_bytes = content.as_bytes().len();
    
    // Write file
    tokio::fs::write(valid_path, content).await?;
    
    let elapsed = start_time.elapsed();
    
    // Build detailed response
    let mut response = format!(
        "Successfully wrote to {}\n\n\
         Statistics:\n\
         - Total lines: {}\n\
         - Total bytes: {}\n\
         - Time elapsed: {:.2}ms\n",
        file_path.display(),
        total_lines,
        total_bytes,
        elapsed.as_secs_f64() * 1000.0
    );
    
    // Add content preview
    if total_lines > 0 {
        response.push_str("\nContent preview:\n");
        
        // First 5 lines
        let preview_lines = std::cmp::min(5, total_lines);
        response.push_str(&format!("First {} lines:\n", preview_lines));
        for (i, line) in lines.iter().take(preview_lines).enumerate() {
            response.push_str(&format!("  {}: {}\n", i + 1, line));
        }
        
        // If file has more than 10 lines, show last 5
        if total_lines > 10 {
            response.push_str(&format!("\n... ({} lines omitted) ...\n\n", total_lines - 10));
            response.push_str("Last 5 lines:\n");
            for (i, line) in lines.iter().rev().take(5).rev().enumerate() {
                let line_num = total_lines - 4 + i;
                response.push_str(&format!("  {}: {}\n", line_num, line));
            }
        }
    }
    
    Ok(response)
}
```

### Approach 2: Use stderr Progress Notifications (Supplementary ⭐⭐⭐)

**Implementation**:
For large file writes, send progress messages via stderr during write process.

**Use Cases**:
- Writing very large files (> 10MB)
- Need to inform user operation is still in progress
- Debugging and monitoring

**Limitations**:
- stderr messages may not display in some clients
- Not a standard tool response channel
- Primarily for logging and diagnostics

### Approach 3: Add write_file_verbose New Tool (Alternative ⭐⭐⭐⭐)

**Implementation**:
Keep existing `write_file` tool for backward compatibility, add new `write_file_verbose` or `write_file_with_stats` tool providing detailed write information.

**Advantages**:
- ✅ Backward compatible
- ✅ Users can choose simple or verbose version
- ✅ Doesn't affect existing clients

## Reference: Similar Existing Implementation

The project's `edit_file` tool provides a good reference example, returning detailed diff information:

```rust
// src/tools/edit_file.rs
pub async fn run_tool(
    params: Self,
    context: &FileSystemService,
) -> std::result::Result<CallToolResult, CallToolError> {
    let diff = context
        .apply_file_edits(Path::new(&params.path), params.edits, params.dry_run, None)
        .await
        .map_err(CallToolError::new)?;

    Ok(CallToolResult::text_content(vec![TextContent::from(diff)]))
}
```

This tool returns complete diff information after editing, demonstrating how to provide detailed operation results in a single response.

## Conclusion

### Streaming Output

**Conclusion**: ❌ **Current MCP protocol and SDK do not support true streaming output**

MCP protocol is based on JSON-RPC 2.0 request-response pattern and doesn't support returning multiple responses or streaming data in a single tool call. Implementing true streaming output would require:

1. MCP protocol specification extension (support for server push or streaming responses)
2. Major update to rust-mcp-sdk (support for streaming response types)
3. Corresponding client support

These changes exceed the scope of a single project and would need coordination at the MCP ecosystem level.

### Line-Level Output

**Conclusion**: ✅ **Can be implemented, "Enhanced Response Content" approach recommended**

While we cannot output each line in real-time during writing, we can return a response containing detailed line-level information after write completion, including:
- Total line count statistics
- Content preview (first and last few lines)
- Byte count statistics
- Execution time
- Other metadata

This approach:
- ✅ Fully compliant with MCP protocol specification
- ✅ Simple implementation, low risk
- ✅ No protocol or SDK modifications needed
- ✅ Provides valuable detailed information about write operations to users

## Recommendations

1. **Short-term**: Implement "Enhanced Response Content" approach, adding detailed statistics and content preview to `write_file` tool

2. **Medium-term**: Consider adding optional parameters (like `verbose` or `show_preview`), allowing users to choose simple or detailed response

3. **Long-term**: If MCP protocol supports streaming responses or progress reporting in the future, can consider reimplementation to leverage these new features

## References

- MCP Protocol Specification: https://modelcontextprotocol.io/
- rust-mcp-sdk: https://github.com/rust-mcp-stack/rust-mcp-sdk
- JSON-RPC 2.0 Specification: https://www.jsonrpc.org/specification
