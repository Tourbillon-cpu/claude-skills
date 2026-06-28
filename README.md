# 🛠️ Claude Code 自定义技能集

个人根据与 AI Agent 的对话与项目经验编写的 Claude Code 自定义技能（Custom Skills），主要用于实验数据处理、记忆文件维护和日常效率提升。

## 📦 技能列表

### 1. [Origin DataLab](./origin-datalab/SKILL.md) — 实验数据作图

用 Python 操控 **Origin/OriginPro** 进行实验数据作图和数据分析。

- **两种工作模式：** originpro 脚本（正式报告） & matplotlib 平替（免 Origin）
- **标准工作流：** 读数据 → 清洗 → 画图 → 标注 → 导出
- **避坑指南：** 收录了 originpro API 的各种踩坑经验
- **适用场景：** 材料/物理/化学实验报告、论文插图

### 2. [Session Cleanup](./session-cleanup/SKILL.md) — 会话收尾清理

会话结束时自动提醒清理临时文件，保持系统整洁。

- **自动识别：** 区分可删文件 vs 需保留文件
- **安全机制：** 删除前让用户确认，不擅自操作
- **适用范围：** 缓存文件、临时克隆的仓库、网页抓取缓存等

### 3. [Memory Maintenance](./memory-maintenance/SKILL.md) — 记忆文件维护

记忆文件的创建规范与维护手册，用于定期清理和整理 Claude Code 的记忆系统。

- **目录规范：** `user/` / `project/` / `reference/` 三层结构
- **生命周期管理：** permanent（常驻） / temporary（做完即删）
- **清理清单：** 内容重复检查、过期信息更新、索引同步
- **适用场景：** 记忆文件熵增时"对抗熵增"

## 🚀 使用方法

1. 将各 skill 文件夹放入 Claude Code 的 skills 目录：
   ```
   .claude/skills/
   ```
2. 在 Claude Code 中通过触发关键词自动调用：
   - 处理实验数据 → `origin-datalab`
   - 收尾清理 → `session-cleanup`
   - 记忆文件维护 → `memory-maintenance`

## 📝 许可

本项目采用 [MIT License](LICENSE) 开源。
