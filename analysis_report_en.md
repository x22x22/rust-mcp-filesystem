# Analysis Report: Stream and Line-by-Line File Writing Support (English)

## Project Overview

**rust-mcp-filesystem** is a Rust-based Model Context Protocol (MCP) server designed for efficient handling of various filesystem operations. This project is a pure Rust rewrite of the JavaScript-based `@modelcontextprotocol/server-filesystem`, offering enhanced performance and type safety.

**Project Information:**
- Version: 0.4.0
- MCP SDK Version: 0.8.1
- MCP Schema Version: 0.9.4
- Repository: https://github.com/rust-mcp-stack/rust-mcp-filesystem

## Analysis Objectives

This analysis report aims to evaluate whether the project supports or can support the following two file writing output methods:

1. **Streaming Output**: Real-time output of content being written during the file writing process
2. **Line-by-Line Output**: Output content line by line as it's being written

## Current Implementation Analysis

### 1. File Writing Tool Implementation

#### 1.1 write_file Tool

**File Location:** `src/tools/write_file.rs`

**Current Implementation:**
```rust
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
```

**Core Service Layer:** `src/fs_service/io/write.rs`

```rust
pub async fn write_file(&self, file_path: &Path, content: &String) -> ServiceResult<()> {
    let allowed_directories = self.allowed_directories().await;
    let valid_path = self.validate_path(file_path, allowed_directories)?;
    tokio::fs::write(valid_path, content).await?;
    Ok(())
}
```

**Key Findings:**
- Uses `tokio::fs::write()` to write all content at once
- Returns a simple success message without detailed write process information
- No progress tracking or intermediate status reporting mechanism

### 2. MCP Protocol Response Format Analysis

#### 2.1 CallToolResult Structure

The project uses the `CallToolResult` structure provided by the MCP SDK to return tool execution results. From code analysis:

```rust
Ok(CallToolResult::text_content(vec![TextContent::from(
    format!("Successfully wrote to {}", &params.path),
)]))
```

**Response Types:**
- `text_content`: Text content
- `image_content`: Image content
- `audio_content`: Audio content

#### 2.2 MCP Protocol Architecture

**Transport Layer:** `StdioTransport`
- Uses standard input/output for communication
- Based on JSON-RPC 2.0 protocol
- Request-response pattern (non-streaming)

**Server Capabilities:**
```rust
capabilities: ServerCapabilities {
    experimental: None,
    logging: None,
    prompts: None,
    resources: None,
    tools: Some(ServerCapabilitiesTools { list_changed: None }),
    completions: None,
    tasks: None,
}
```

### 3. Protocol Layer Limitations

From source code analysis:

1. **Request-Response Model:** MCP protocol follows a traditional request-response pattern, with tool calls returning a single `CallToolResult`
2. **No Built-in Streaming Support:** Current MCP SDK (v0.8.1) `CallToolResult` structure does not support streaming responses
3. **No Progress Notification Mechanism:** While the protocol supports notifications, they're mainly used for server state changes like `roots_list_changed`, not for tool execution progress

## Feasibility Analysis for Streaming/Line-by-Line Output Support

### Option 1: Simulate Line-by-Line Output with Text Content ⭐ Feasible (Limited)

**Implementation Approach:**
After writing the file, return a response containing a summary of written content, displaying partial or full content by lines.

**Advantages:**
- Simple implementation, no protocol layer modifications needed
- Can provide preview of written content
- Suitable for small files and debugging scenarios

**Disadvantages:**
- Not true real-time streaming output
- Large files would result in oversized responses
- Still returns everything at once, not progressive

**Implementation Example:**
```rust
pub async fn write_file_with_preview(
    &self, 
    file_path: &Path, 
    content: &String
) -> ServiceResult<String> {
    // Write file
    let valid_path = self.validate_path(file_path, allowed_directories)?;
    tokio::fs::write(valid_path, content).await?;
    
    // Generate line-by-line preview
    let lines: Vec<&str> = content.lines().collect();
    let preview = if lines.len() > 100 {
        format!("Wrote {} lines:\nFirst 10 lines:\n{}\n...\nLast 10 lines:\n{}", 
            lines.len(),
            lines.iter().take(10).map(|l| format!("  {}", l)).collect::<Vec<_>>().join("\n"),
            lines.iter().rev().take(10).rev().map(|l| format!("  {}", l)).collect::<Vec<_>>().join("\n")
        )
    } else {
        format!("Wrote {} lines:\n{}", 
            lines.len(),
            lines.iter().enumerate().map(|(i, l)| format!("  {}: {}", i+1, l)).collect::<Vec<_>>().join("\n")
        )
    };
    
    Ok(preview)
}
```

### Option 2: Use Server Logs (stderr_message) for Progress Output ⭐ Feasible (Recommended)

**Implementation Approach:**
Utilize the MCP server's `stderr_message` functionality to send progress notifications during the writing process.

**Advantages:**
- Can achieve true real-time progress output
- Does not affect the tool's normal return value
- Clients can choose whether to display these messages
- Suitable for large file writing scenarios

**Disadvantages:**
- stderr messages are typically for logging, not a standard data return channel
- Requires client support for stderr message handling
- May mix with other log messages

**Implementation Example:**
```rust
pub async fn write_file_with_progress(
    &self,
    file_path: &Path,
    content: &String,
    runtime: Arc<dyn McpServer>
) -> ServiceResult<()> {
    let allowed_directories = self.allowed_directories().await;
    let valid_path = self.validate_path(file_path, allowed_directories)?;
    
    // Send start message
    let _ = runtime.stderr_message(format!("Starting to write file: {}", file_path.display())).await;
    
    // Write line by line and report progress
    let lines: Vec<&str> = content.lines().collect();
    let total_lines = lines.len();
    let mut file = tokio::fs::File::create(&valid_path).await?;
    
    for (i, line) in lines.iter().enumerate() {
        use tokio::io::AsyncWriteExt;
        file.write_all(line.as_bytes()).await?;
        file.write_all(b"\n").await?;
        
        // Report progress every 100 lines
        if (i + 1) % 100 == 0 || i == total_lines - 1 {
            let progress = ((i + 1) as f64 / total_lines as f64 * 100.0) as u32;
            let _ = runtime.stderr_message(
                format!("Write progress: {}/{} lines ({}%)", i + 1, total_lines, progress)
            ).await;
        }
    }
    
    let _ = runtime.stderr_message(
        format!("✓ Successfully wrote {} lines to {}", total_lines, file_path.display())
    ).await;
    
    Ok(())
}
```

### Option 3: Extend MCP Protocol to Support Streaming Responses ❌ Not Feasible (Requires Upstream Support)

**Implementation Approach:**
Modify MCP SDK and protocol specification to support streaming responses.

**Feasibility:**
- Requires modifications to `rust-mcp-sdk` and `rust-mcp-schema` libraries
- Requires updates to MCP protocol specification
- Requires cooperation and support from all MCP clients
- Beyond the scope of a single project

**Conclusion:** Not feasible (unless MCP protocol itself adds this feature)

### Option 4: Use Tasks API (If Available) ⚠️ To Be Evaluated

**Current Status Analysis:**
According to server capabilities configuration:
```rust
tasks: None,
```

The current server does not have Tasks capability enabled. The MCP protocol may support a Tasks API for long-running operations, but further research is needed.

**If Tasks API Supports Progress Reporting:**
- Can treat file writing as a task
- Report progress through task status updates
- Clients can query task progress

**Requires Research:**
- Specific specifications of MCP Tasks API
- Level of Tasks support in rust-mcp-sdk
- Whether Tasks supports progress callbacks

## Conclusions and Recommendations

### Current Status

**Does NOT Support Real-time Streaming/Line-by-Line Output**

The current `rust-mcp-filesystem` project's `write_file` tool:
- ❌ Does not support real-time streaming output
- ❌ Does not support line-by-line output
- ✅ Uses atomic write operation (`tokio::fs::write`)
- ✅ Returns only simple success/failure messages

### Recommended Implementation Approach

#### Short-term Solution (Immediately Feasible)

**Option 2: Use stderr_message for Progress Output**

**Recommendation Rationale:**
1. No need to modify MCP protocol or SDK
2. Can achieve real progress reporting
3. Low implementation complexity
4. Does not affect existing API compatibility

**Implementation Steps:**
1. Modify `WriteFile::run_tool` method to receive `runtime: Arc<dyn McpServer>` parameter
2. Add progress reporting logic in `FileSystemService::write_file`
3. For large files (e.g., > 1000 lines), use streaming write + progress reporting
4. Add optional parameter `show_progress: bool` to let users choose whether to display progress

**Example Configuration:**
```rust
pub struct WriteFile {
    pub path: String,
    pub content: String,
    /// Whether to show write progress (default: auto-enabled for content > 1000 lines)
    #[serde(default)]
    pub show_progress: Option<bool>,
}
```

#### Mid-term Solution (Requires Evaluation)

**Evaluate MCP Tasks API Feasibility**

If MCP protocol's Tasks API is mature enough and supports progress reporting:
1. Implement Task interface for large file write operations
2. Report progress through task status updates
3. Provide more standardized progress tracking mechanism

#### Long-term Solution (Depends on Upstream)

**Promote MCP Protocol to Support Streaming Responses**

1. Propose streaming response requirements to MCP protocol community
2. Participate in rust-mcp-sdk development to add streaming response support
3. Implement after protocol standardization

### Technical Trade-offs

| Option | Implementation Difficulty | Real-time | Compatibility | Recommendation |
|--------|--------------------------|-----------|---------------|----------------|
| Option 1: Text Content Simulation | ⭐ | ❌ | ✅ | ⭐⭐ |
| Option 2: stderr Progress Output | ⭐⭐ | ✅ | ✅ | ⭐⭐⭐⭐⭐ |
| Option 3: Protocol Extension | ⭐⭐⭐⭐⭐ | ✅ | ❌ | ⭐ |
| Option 4: Tasks API | ⭐⭐⭐ | ✅ | ⚠️ | ⭐⭐⭐ |

### Other Considerations

#### 1. Performance Impact
- **Atomic Write vs Streaming Write:** Current `tokio::fs::write` is a highly optimized atomic operation; switching to streaming write may affect performance
- **Progress Reporting Overhead:** Frequent progress messages may increase I/O overhead

**Recommendation:** Enable progress reporting only for larger files (e.g., > 10KB or > 1000 lines)

#### 2. User Experience
- **Default Behavior:** Keep existing behavior as default to ensure backward compatibility
- **Optional Feature:** Let users choose through parameters whether they need progress output
- **Client Support:** Not all MCP clients will display stderr messages

#### 3. Security and Permissions
- Progress output should not leak sensitive file content
- Should only report metadata such as line counts, byte counts

## Appendix: Related Code Files

### Core Files
- `src/tools/write_file.rs` - Write file tool definition
- `src/fs_service/io/write.rs` - Filesystem service write implementation
- `src/handler.rs` - MCP request handler
- `src/server.rs` - MCP server initialization

### Reference Files
- `src/tools/edit_file.rs` - Edit file tool (demonstrates diff output)
- `src/fs_service/io/read.rs` - Read operations (demonstrates streaming processing)
- `Cargo.toml` - Dependency configuration

## References

1. **MCP Protocol Specification:** https://spec.modelcontextprotocol.io/
2. **rust-mcp-sdk:** https://github.com/rust-mcp-stack/rust-mcp-sdk
3. **rust-mcp-schema:** https://github.com/rust-mcp-stack/rust-mcp-schema
4. **Project Documentation:** https://rust-mcp-stack.github.io/rust-mcp-filesystem

---

**Report Date:** 2026-02-09  
**Analyst:** GitHub Copilot  
**Project Version:** 0.4.0
