# SDK 对比分析：官方 RMCP vs 社区 rust-mcp-sdk

## 新需求确认

用户询问：**当前仓库使用的 SDK 和 https://github.com/modelcontextprotocol/rust-sdk 有什么关系？是否可以切换？**

## 执行摘要

| 对比项 | rust-mcp-stack/rust-mcp-sdk | modelcontextprotocol/rust-sdk |
|--------|----------------------------|-------------------------------|
| **类型** | 社区实现 | **官方实现** ⭐ |
| **包名** | `rust-mcp-sdk` | `rmcp` |
| **当前版本** | v0.8.3 (2026-02) | **v0.14.0** (更新) |
| **维护者** | rust-mcp-stack 社区 | Model Context Protocol 官方团队 |
| **GitHub Stars** | ~数百 | **~3k** ⭐ |
| **协议支持** | MCP 2025-11-25 | MCP 2025-11-25 |
| **许可证** | MIT | Apache-2.0 |
| **Rust 版本** | 1.80+ | 未知 |
| **推荐度** | ✅ 稳定可用 | ⭐⭐⭐ **强烈推荐**（官方）|

## 详细对比

### 1. 项目背景

#### rust-mcp-stack/rust-mcp-sdk（当前使用）
- **仓库**: https://github.com/rust-mcp-stack/rust-mcp-sdk
- **包名**: `rust-mcp-sdk`
- **版本**: v0.8.3 (latest)
- **说明**: 社区维护的 Rust MCP SDK 实现
- **特点**: 
  - 提供完整的 MCP 服务器和客户端支持
  - 包含多个辅助 crate（rust-mcp-schema, rust-mcp-transport, rust-mcp-macros）
  - 支持多种传输方式（stdio, SSE, streamable HTTP）
  - 包含 OAuth 认证支持

#### modelcontextprotocol/rust-sdk（官方）
- **仓库**: https://github.com/modelcontextprotocol/rust-sdk
- **包名**: `rmcp` ⭐
- **版本**: v0.14.0 (latest)
- **说明**: **Model Context Protocol 官方维护的 Rust SDK**
- **特点**:
  - 官方支持和维护
  - 更活跃的开发（更新频率更高）
  - 社区生态更大（3k+ stars）
  - 设计更简洁（2个主要 crate: rmcp, rmcp-macros）
  - 更多实际项目使用案例

### 2. 架构对比

#### rust-mcp-sdk 架构
```
rust-mcp-sdk (0.8.3)
├── rust-mcp-sdk        (主 SDK)
├── rust-mcp-schema     (协议 schema)
├── rust-mcp-transport  (传输层)
├── rust-mcp-macros     (宏)
└── rust-mcp-extra      (扩展功能)
```

#### rmcp 架构
```
rmcp (0.14.0)
├── rmcp         (核心 SDK)
└── rmcp-macros  (宏)
```

### 3. API 对比

#### 基础服务器创建

**rust-mcp-sdk (当前)**:
```rust
use rust_mcp_sdk::{*, mcp_server::server_runtime};

let transport = StdioTransport::new(TransportOptions::default())?;
let handler = MyHandler::default().to_mcp_server_handler();
let server = server_runtime::create_server(server_info, transport, handler);
server.start().await?;
```

**rmcp (官方)**:
```rust
use rmcp::ServiceExt;
use tokio::io::{stdin, stdout};

let service = MyService::new();
let server = service.serve((stdin(), stdout())).await?;
server.waiting().await?;
```

API 风格不同，官方 rmcp 更简洁。

#### 进度通知

**rust-mcp-sdk**:
```rust
runtime.notify_progress(ProgressNotificationParams {
    message: Some("进度消息".into()),
    progress: 0.5,
    meta: None,
}).await?;
```

**rmcp**:
```rust
server.notify_progress(ProgressToken { ... }, ProgressParams {
    progress_token: token,
    progress: 50,
    total: Some(100),
}).await?;
```

两者都支持进度通知，但 API 略有不同。

### 4. 功能特性对比

| 特性 | rust-mcp-sdk | rmcp (官方) |
|------|--------------|-------------|
| 服务器支持 | ✅ | ✅ |
| 客户端支持 | ✅ | ✅ |
| Stdio 传输 | ✅ | ✅ |
| HTTP/SSE 传输 | ✅ | ✅ |
| 进度通知 | ✅ | ✅ |
| OAuth 认证 | ✅ | ✅ |
| 任务支持 | ✅ | ✅ |
| 批量消息 | ✅ | ✅ |
| Schema 生成 | ✅ (serde) | ✅ (schemars) |
| 宏支持 | ✅ | ✅ |
| 子进程传输 | ✅ | ✅ |
| HTTP 服务器 | ✅ (axum) | ✅ (axum) |
| 覆盖率测试 | ❌ | ✅ (~90%) |

### 5. 生态系统对比

#### rust-mcp-sdk 生态
- rust-mcp-filesystem (本项目)
- 其他少量社区项目

#### rmcp 生态（官方）⭐
根据官方文档，使用 rmcp 构建的项目包括：
- rustfs-mcp - S3 对象存储 MCP 服务器
- containerd-mcp-server - 容器管理
- rmcp-openapi-server - OpenAPI 工具
- nvim-mcp - Neovim 集成
- terminator - 桌面自动化
- stakpak-agent - DevOps 安全代理
- video-transcriber-mcp-rs - 视频转录
- NexusCore MCP - 恶意软件分析
- spreadsheet-mcp - 电子表格分析
- hyper-mcp - WASM 插件扩展
- 还有更多...

**官方 SDK 拥有更大、更活跃的生态系统！**

### 6. 维护状态对比

#### rust-mcp-sdk
- 最新版本: v0.8.3 (2026-02-01)
- 更新频率: 较低
- Issue/PR 活跃度: 中等
- 文档完整度: 良好

#### rmcp（官方）⭐
- 最新版本: v0.14.0
- 更新频率: **高**（官方持续维护）
- Issue/PR 活跃度: **高**
- 文档完整度: **优秀**
- 社区支持: **强**（3k+ stars）

### 7. 迁移可行性分析

#### 是否可以迁移？

✅ **可以迁移，但需要重构代码**

#### 迁移工作量评估

| 模块 | 工作量 | 说明 |
|------|--------|------|
| 依赖更新 | 低 | 更改 Cargo.toml |
| 服务器初始化 | 中 | API 结构不同，需要重写 |
| 工具定义 | 低-中 | 宏语法可能略有不同 |
| Handler 实现 | 中-高 | Trait 定义不同 |
| 传输层 | 低 | 基本兼容 |
| 测试 | 中 | 需要更新测试代码 |
| **总体** | **中-高** | 估计 1-2 周工作量 |

#### 迁移风险

- ⚠️ **Breaking Changes**: API 不完全兼容，需要重写部分代码
- ⚠️ **学习成本**: 需要熟悉新的 API 设计
- ⚠️ **测试覆盖**: 需要全面测试确保功能正常
- ✅ **文档支持**: 官方文档更完善
- ✅ **社区支持**: 更大的用户基础

### 8. 性能对比

两个 SDK 都基于 tokio async runtime，性能差异应该不大。官方 rmcp 有覆盖率测试和更多的优化。

### 9. 版本兼容性

#### rust-mcp-sdk
- MCP 协议: 2025-11-25
- 向后兼容: 支持旧版本客户端

#### rmcp
- MCP 协议: 2025-11-25
- 向后兼容: 支持旧版本客户端

两者都支持最新的 MCP 协议版本。

## 推荐建议

### 短期建议（当前项目）

**不建议立即迁移**，原因：
1. ✅ 当前 rust-mcp-sdk v0.8.3 功能完整且稳定
2. ✅ 已经支持进度通知功能
3. ⚠️ 迁移成本较高（1-2周工作量）
4. ⚠️ 需要全面测试
5. ✅ 项目已接近完成，风险大于收益

**推荐操作**：
- 先升级到 rust-mcp-sdk v0.8.3
- 实现进度通知功能
- 保持关注官方 rmcp 的发展

### 中长期建议（新项目或大版本升级）

**强烈推荐使用官方 rmcp**，原因：
1. ⭐ **官方支持**: Model Context Protocol 官方维护
2. ⭐ **更活跃**: 更新频率高，bug 修复快
3. ⭐ **更大生态**: 3k+ stars，更多项目使用
4. ⭐ **更好文档**: 官方文档更完善
5. ⭐ **长期保障**: 官方项目更有保障

### 迁移时机

考虑在以下情况迁移到官方 rmcp：
- ✅ 项目大版本升级（如 v1.0 -> v2.0）
- ✅ 需要重大重构时
- ✅ 遇到 rust-mcp-sdk 无法解决的问题
- ✅ 需要使用官方生态的其他工具
- ✅ 新项目从零开始

## 迁移指南（如果决定迁移）

### 1. 更新依赖

```toml
[dependencies]
# 从
# rust-mcp-sdk = {version="0.8.3", ...}

# 改为
rmcp = {version="0.14.0", features = ["server", "macros"]}
```

### 2. 重写服务器初始化

参考官方示例重写 `src/server.rs` 和 `src/handler.rs`

### 3. 更新工具定义

可能需要调整宏的使用方式

### 4. 更新测试

根据新 API 更新测试用例

### 5. 验证功能

全面测试所有功能

## 结论

### 当前项目

**建议**: 继续使用 rust-mcp-sdk v0.8.3
- ✅ 稳定可靠
- ✅ 功能完整
- ✅ 已支持进度通知
- ⚠️ 迁移成本高

### 未来项目

**强烈推荐**: 使用官方 rmcp
- ⭐ 官方支持
- ⭐ 活跃维护
- ⭐ 更大生态
- ⭐ 更好保障

## 参考资源

- [官方 rmcp SDK](https://github.com/modelcontextprotocol/rust-sdk)
- [rmcp 文档](https://docs.rs/rmcp)
- [rust-mcp-sdk](https://github.com/rust-mcp-stack/rust-mcp-sdk)
- [MCP 协议规范](https://modelcontextprotocol.io/specification/2025-11-25/)

## 附录：快速决策表

| 情况 | 推荐方案 | 理由 |
|------|---------|------|
| 当前项目接近完成 | 保持 rust-mcp-sdk | 避免不必要风险 |
| 当前项目处于早期 | 考虑迁移到 rmcp | 长期收益 |
| 新项目启动 | **使用 rmcp** ⭐ | 官方支持 |
| 遇到技术问题 | 评估后决定 | 看是否官方能解决 |
| 需要最新特性 | **使用 rmcp** ⭐ | 更新更快 |
| 追求稳定性 | 两者都可 | 都很稳定 |
