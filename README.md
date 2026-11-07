# 同幕 · CoStage

**人与多 Agent 的协作空间。**  
A shared workspace for people and AI agents.

CoStage 让人和多个 Agent 在同一房间协作：人可以 @ Agent 发起任务，Agent 可以委派其他 Agent，并在授权范围内共享上下文。产品面向 AI 角色扮演、剧本杀和辅助办公，支持 Agent 配置版本管理、多模型接入与运行过程观测。

当前处于系统设计阶段，仓库包含设计文档和调研资料，尚未包含产品实现。

## 系统设计

已选架构：**自研 Agent 循环和策略，LangGraph 负责持久执行。**

- 自研模块负责动作策略、上下文组装与压缩、记忆管理和细粒度观测。
- LangGraph 负责阶段结果持久化、暂停与恢复。
- 业务系统持有房间、Task、权限、发布版本与工具执行账本。

完整职责划分、状态归属和恢复边界见系统设计文档。

| 文档 | 内容 |
| --- | --- |
| [系统设计 v3](docs/system-design.md) | 产品场景、方案比较、组件关系、消息与委派流转、版本管理、恢复边界和可观测性，含三张 Mermaid 图 |
| [GitHub 调研快照](docs/research/github-snapshot-2026-11-07.json) | 2026-11-07（北京时间）采集的 29 个项目的仓库元数据与近期发布记录 |
| [飞书设计文档](https://zcnlg5wqqjid.feishu.cn/docx/HV0XdSdxLo6g9BxhIXNcRGlpnPh) | 同版设计文档与图示，访问取决于飞书文档权限 |

调研快照记录采集时点的项目状态；容量、并发和预算参数在设计文档中标为设计假设。
