# 升级到 rust-mcp-sdk v0.8.3 实现进度通知

## 概述

本指南说明如何升级项目以支持写文件时的进度通知功能。

## 关键发现

✅ **MCP 协议 2025-11-25 和 rust-mcp-sdk v0.8.3+ 原生支持进度通知！**

这意味着可以在文件写入过程中实时向客户端发送进度更新，而无需修改协议或使用非标准特性。

## 升级步骤

### 1. 更新依赖

**文件**: `Cargo.toml`

```toml
[dependencies]
rust-mcp-sdk = {version="0.8.3", default-features = false, features = [
    "server",
    "macros",
    "stdio"
] }
```

### 2. 运行升级命令

```bash
cargo update -p rust-mcp-sdk -p rust-mcp-schema -p rust-mcp-macros -p rust-mcp-transport
cargo build
```

### 3. 修改工具签名（如需实现进度通知）

**文件**: `src/tools/write_file.rs`

添加 `runtime` 参数以支持发送进度通知：

```rust
impl WriteFile {
    pub async fn run_tool(
        params: Self,
        context: &FileSystemService,
        runtime: Arc<dyn McpServer>,  // 新增此参数
    ) -> std::result::Result<CallToolResult, CallToolError> {
        // 实现代码...
    }
}
```

### 4. 更新工具调用（如需实现进度通知）

**文件**: `src/handler.rs`

修改 `invoke_tools!` 宏调用，传递 `runtime` 参数：

```rust
// 当前方式（不需要修改宏，但需要在工具内部获取 runtime）
// 或者修改宏以支持传递 runtime
```

## 实现示例

### 基础实现：带进度通知的写文件

```rust
use rust_mcp_sdk::schema::ProgressNotificationParams;
use tokio::io::AsyncWriteExt;

pub async fn run_tool(
    params: Self,
    context: &FileSystemService,
    runtime: Arc<dyn McpServer>,
) -> std::result::Result<CallToolResult, CallToolError> {
    let content = &params.content;
    let lines: Vec<&str> = content.lines().collect();
    let total_lines = lines.len();
    let file_path = Path::new(&params.path);
    
    // 仅对大文件启用进度通知
    let enable_progress = total_lines > 1000;
    
    if enable_progress {
        // 发送开始通知
        let _ = runtime.notify_progress(ProgressNotificationParams {
            message: Some(format!("开始写入 {} 行", total_lines)),
            progress: 0.0,
            meta: None,
        }).await;
    }
    
    // 执行写入
    context.write_file(file_path, content).await
        .map_err(CallToolError::new)?;
    
    if enable_progress {
        // 发送完成通知
        let _ = runtime.notify_progress(ProgressNotificationParams {
            message: Some(format!("完成写入 {} 行", total_lines)),
            progress: 1.0,
            meta: None,
        }).await;
    }
    
    Ok(CallToolResult::text_content(vec![TextContent::from(
        format!("成功写入 {} 行到 {}", total_lines, params.path),
    )]))
}
```

### 高级实现：分块写入带进度

```rust
pub async fn run_tool(
    params: Self,
    context: &FileSystemService,
    runtime: Arc<dyn McpServer>,
) -> std::result::Result<CallToolResult, CallToolError> {
    let content = &params.content;
    let lines: Vec<&str> = content.lines().collect();
    let total_lines = lines.len();
    let allowed_directories = context.allowed_directories().await;
    let valid_path = context.validate_path(Path::new(&params.path), allowed_directories)
        .map_err(CallToolError::new)?;
    
    // 对大文件使用分块写入和进度报告
    if total_lines > 1000 {
        let _ = runtime.notify_progress(ProgressNotificationParams {
            message: Some(format!("开始写入 {} 行到 {}", total_lines, params.path)),
            progress: 0.0,
            meta: None,
        }).await;
        
        let mut file = tokio::fs::File::create(&valid_path).await
            .map_err(|e| CallToolError::new(e))?;
        
        let progress_interval = (total_lines / 10).max(1); // 每 10% 报告一次
        
        for (idx, line) in lines.iter().enumerate() {
            file.write_all(line.as_bytes()).await
                .map_err(|e| CallToolError::new(e))?;
            file.write_all(b"\n").await
                .map_err(|e| CallToolError::new(e))?;
            
            // 定期报告进度
            if (idx + 1) % progress_interval == 0 || idx + 1 == total_lines {
                let progress = (idx + 1) as f64 / total_lines as f64;
                let _ = runtime.notify_progress(ProgressNotificationParams {
                    message: Some(format!("已写入 {}/{} 行", idx + 1, total_lines)),
                    progress,
                    meta: None,
                }).await;
            }
        }
        
        file.flush().await.map_err(|e| CallToolError::new(e))?;
    } else {
        // 小文件直接写入
        tokio::fs::write(&valid_path, content).await
            .map_err(|e| CallToolError::new(e))?;
    }
    
    Ok(CallToolResult::text_content(vec![TextContent::from(
        format!("成功写入 {} 行到 {}", total_lines, params.path),
    )]))
}
```

## 测试

### 1. 编译检查

```bash
cargo check
cargo clippy
```

### 2. 功能测试

创建一个大文件来测试进度通知：

```bash
# 生成测试内容
python3 -c "for i in range(5000): print(f'Line {i+1}: Test content')" > /tmp/test_large.txt

# 运行服务器并测试 write_file 工具
cargo run -- --allow-write /tmp
```

## 向后兼容性

- ✅ SDK v0.8.3 向后兼容 v0.8.1
- ✅ 进度通知是可选的，不影响基本功能
- ✅ 客户端不支持进度通知时会忽略，不影响工具执行

## 性能考虑

1. **进度通知频率**: 建议每 10% 或每 N 行报告一次，避免过于频繁
2. **小文件优化**: 对小文件（如 < 1000 行）跳过进度通知
3. **异步发送**: 使用 `let _ = runtime.notify_progress()` 避免等待响应

## 相关文档

- [完整分析报告（中文）](./分析报告-流式输出写文件支持.md)
- [完整分析报告（英文）](./Analysis-Report-Streaming-File-Write-Support.md)
- [MCP 协议规范](https://modelcontextprotocol.io/specification/2025-11-25/)
- [rust-mcp-sdk 文档](https://docs.rs/rust-mcp-sdk/)

## 支持

如有问题或需要帮助，请：
1. 查看分析报告获取详细信息
2. 参考 rust-mcp-sdk 的示例代码
3. 在项目 Issues 中提问
