# Claude Code 全栈 Agent Pack

本仓库收录了一套基于 Claude Code 设计逻辑的全栈开发 Agent 配置，适用于 VS Code Copilot Chat/Claude Code 环境下的自动化开发、代码审查与安全保障。

## 目录结构

```
.github/
  agents/
    main-brain.agent.md           # 主大脑智能体，统一调度与分发
    fullstack-backend.agent.md    # 后端开发专用 Agent
    fullstack-frontend.agent.md   # 前端开发专用 Agent
    fullstack-researcher.agent.md # 只读分析/研究 Agent
    fullstack-verifier.agent.md   # 质量与安全验证 Agent
  instructions/
    security.instructions.md      # 全局安全开发规范指令
```

## 设计理念

- **分工明确**：每个 Agent 只负责单一职责，主大脑负责任务拆解与调度，Worker 只做本职工作。
- **安全优先**：内置 OWASP Top 10 安全防护指令，所有开发和验证流程强制执行 SQL 注入、XSS、越权、CSRF、路径穿越等安全检查。
- **自动化审查**：Verifier Agent 自动执行类型检查、Lint、单元测试、契约一致性和安全扫描，输出结构化报告。
- **可复用与移植**：只需将 .github/agents/ 和 .github/instructions/ 目录复制到任意项目，即可获得全栈安全开发与自动化审查能力。
- **极简集成**：无需修改业务代码，纯配置驱动，支持 VS Code Copilot Chat/Claude Code 全自动加载。

## 使用方法

1. 将 `.github/agents/` 和 `.github/instructions/` 目录复制到你的项目根目录。
2. （可选）如需批量迁移，直接解压 `fullstack-agent-pack.zip`。
3. 在 VS Code Copilot Chat/Claude Code 环境下，自动识别并加载 Agent，无需额外配置。
4. 参考 `security.instructions.md`，所有开发和审查任务自动遵循安全规范。

## 适用场景

- 企业级全栈项目安全开发
- 自动化代码审查与合规验证
- 多人协作下的角色分工与责任隔离
- 需要强安全保障的敏感业务系统

## 致谢

本 Agent Pack 基于 Claude Code 官方设计理念与最佳实践，结合实际项目安全需求优化而成。

---

如有建议或问题，欢迎 issue 交流。
