# from-beginner-to-master · 从入门到精通系列教程

一套覆盖 **AI 智能体、AI 编码助手生态、容器与数据工程、AI 视频创作** 等方向的系列技术教程。每册教程按「入门篇 → 进阶篇 → 高级篇（→ 精通篇）」由浅入深组织，目标是把读者从「首次接触某项技术」带到「在生产环境中熟练运用」。

各册教程均基于写作时点的官方文档与官方发布说明逐项核实，文首附有**版本与核实信息表**（基准版本、核实日期、资料来源、运行环境、读者假设），并在时效性易变处标注「以官方文档为准」。

## 教程总览

每册教程位于独立目录中，同时提供 **Markdown**（在线阅读）与 **PDF**（离线 / 打印）两种格式，两者内容相同。

### AI 智能体与记忆系统

| 教程 | 内容简介 | 基准版本 |
| --- | --- | --- |
| [Agentic AI 智能体应用开发](<Agentic AI 智能体应用开发-从入门到精通/Agentic AI 智能体应用开发-从入门到精通.md>) | 主流智能体开发框架（LangGraph、OpenAI Agents SDK、Claude Agent SDK、Microsoft Agent Framework、CrewAI、Google ADK）与 MCP、A2A 规范的实战教程 | 多框架，含 MCP 2026-07-28、A2A v1.0 |
| [Hindsight](<Hindsight-从入门到精通/hindsight-从入门到精通.md>) | 为 AI 智能体构建长期记忆能力：从首次集成到生产环境运用 | hindsight-api 1.1.0 |
| [Agentmemory](<Agentmemory-从入门到精通/agentmemory-从入门到精通.md>) | AI 编码代理的持久记忆引擎：安装、接入与日常使用 | v0.9.30 |

### AI 编码助手生态

| 教程 | 内容简介 | 基准版本 |
| --- | --- | --- |
| [Context-Mode](<Context-Mode-从入门到精通/context-mode-从入门到精通.md>) | 面向 AI 编码代理的上下文窗口优化工具：安装、hooks 与收益实测 | v1.0.169 |
| [Understand Anything](<Understand-Anything-从入门到精通/understand-anything-从入门到精通.md>) | 面向 AI 编码助手的代码库知识图谱技能插件：分析项目、生成可共享的知识图谱与交互式面板 | main 分支（持续更新） |
| [Codebase-Memory-MCP](<Codebase-Memory-MCP-从入门到精通/codebase-memory-mcp-从入门到精通.md>) | 代码库索引与检索（MCP）：智能体接入、三类检索、关系分析与图查询 | v0.12.1 |
| [CodeGraph](<CodeGraph-从入门到精通/codegraph-从入门到精通.md>) | 代码图谱工具：让智能体更快、更准确地理解代码库 | v1.6.2 |
| [Open-Code-Review](<Open-Code-Review-从入门到精通/open-code-review-从入门到精通.md>) | 阿里巴巴开源的 AI 代码审查 CLI：三种评审模式与全量扫描、规则定制、Agent 集成与 CI/CD 流水线 | v1.13.x |

### 容器与数据工程

| 教程 | 内容简介 | 基准版本 |
| --- | --- | --- |
| [Docker](<Docker-从入门到精通/docker-从入门到精通.md>) | 容器技术系统教程：镜像、网络、存储、Compose 与生产实践 | Docker Engine 29.x / Compose v2 |
| [异构数据库迁移](<Database-Migration-从入门到精通/database-migration-从入门到精通.md>) | 数据迁移全景：Addax、Apache SeaTunnel、Flink CDC、Debezium、pgloader、Canal、mydumper、SymmetricDS | 各工具 2026-10 基线 |

### AI 视频与创作

| 教程 | 内容简介 | 基准版本 |
| --- | --- | --- |
| [ComfyUI](<ComfyUI-从入门到精通/comfyui-从入门到精通.md>) | 节点式 AI 图像生成工作流：节点、采样器、模型与工作流编排 | v0.42.x |
| [MoneyPrinterTurbo](<MoneyPrinterTurbo-从入门到精通/moneyprinterturbo-从入门到精通.md>) | 开源全自动短视频生成工具：部署、配置与批量生产 | v1.3.9 |
| [ViMax](<ViMax-从入门到精通/vimax-从入门到精通.md>) | 港大 HKUDS 开源的智能体视频创作框架：多种入口与工作流、一致性调优与实战 | v1.2.0 |
| [Pixelle-Video](<Pixelle-Video-从入门到精通/pixelle-video-从入门到精通.md>) | AI 全自动短视频引擎：安装、首个视频、风格与声音定制、工作流扩展 | v0.1.15（含主分支新特性） |
| [OpenMontage](<OpenMontage-从入门到精通/openmontage-从入门到精通.md>) | AI 编程助手驱动的开源视频制作系统：自然语言驱动的全流水线视频制作 | 官方仓库 main 分支 |

## 如何使用

1. **按方向选择教程**：根据上表按主题找到对应的教程目录。
2. **选择阅读格式**：
   - 在线阅读：点击上表链接或直接打开各目录下的 `.md` 文件；
   - 离线阅读 / 打印：使用同目录下与 `.md` 同名的 `.pdf` 文件，两种格式内容相同。
3. **按顺序学习**：各册教程的章节顺序即推荐学习路径；已具备相关经验的读者可按目录跳读，各章亦可独立查阅。

## 说明与声明

- **时效性**：各册教程的版本号、命令与行为均以文首标注的核实日期为准（多册集中于 2026 年 10 月下旬，新册可能延后至 11 月初）。所涉工具大多迭代较快，若与官方文档存在出入，以官方文档为准。
- **性质**：本系列为教学性整理文档，非官方出版物；各教程涉及的商标与项目权利归属其各自权利人。
- **成本与密钥**：涉及第三方模型 API 费用的教程仅说明计费结构，实际价格与免费额度以各服务商现行政策为准；切勿将真实密钥提交至任何仓库。
