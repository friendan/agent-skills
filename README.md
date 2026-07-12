# agent-skills
agent-skills


https://github.com/anthropics/skills
https://www.skills.sh/

Zed Ide写死的的skills文件夹：C:\Users\admin\.agents\skills 如何自定义skills目录?
解决方法：使用mklink /J 目录联接（首选，无需管理员权限，跨盘完美支持）
1、删除C盘原skills文件夹（关键，否则无法创建链接，改名字也可以，方便还原）
2、打开CMD执行：mklink /J "C:\Users\admin\.agents\skills" "D:\agent-skills\Zed\skills"


Zed includes a built-in create-skill skill — invoke it with /create-skill and the agent walks you through the process.

Global	~/.agents/skills/	Every project
Project-local	<worktree>/.agents/skills/	Only the current project

Zed supports AGENTS.md as the primary instruction file for personal and project-level agent guidance.

Personal instructions apply to every project you open with the Zed Agent.
Create or edit:
~/.config/zed/AGENTS.md
On Windows, the equivalent file is under %APPDATA%\Zed\AGENTS.md.

Project instruction files apply to the current project. Zed uses the first matching file in this list:

.rules
.cursorrules
.windsurfrules
.clinerules
.github/copilot-instructions.md
AGENT.md
AGENTS.md
CLAUDE.md
GEMINI.md
Project instructions override personal AGENTS.md when they conflict.

============================================ AGENTS.md ====================================
版本1：程序员版（专业开发场景，适配Codex / Claude Code）
适合日常项目开发、代码调试、工程重构等场景，规则严谨贴合开发规范。

## 编码前思考
- 明确假设，不确定时询问而非猜测。
- 存在歧义时，列出多种解释，不默默选定单一方案。
- 如果任务有明显更简单的做法，直接指出优化思路。
- 发现代码矛盾、逻辑不一致时及时暂停，请求信息澄清。
## 简洁优先
- 用最少的代码解决问题，拒绝冗余实现。
- 不为一次性需求创建抽象层、复杂架构。
- 不盲目增加扩展性、可配置性，应对“未来可能用到”的场景。
- 若代码可大幅精简，主动重写优化。
- 校验标准：以资深工程师视角判断，代码若过于复杂，立即简化。
## 精准修改
- 仅修改与当前任务直接相关的代码内容。
- 不顺手优化相邻代码、注释、排版格式。
- 不重构原本可以正常运行的代码模块。
- 严格匹配项目现有代码风格，保留原有编码习惯。
- 因本次修改产生的无效导入、废弃变量，可直接删除。
- 发现项目中原有的死代码、冗余内容，仅做文字提醒，不擅自删除。
## 目标驱动执行
- 执行任务前，定义清晰、可落地的成功标准。
- 将“修复Bug”转化为：编写用例复现问题，再调试至用例正常通过。
- 将“新增校验功能”转化为：针对异常输入编写测试用例，保证全部通过。
- 将“代码重构”转化为：完成重构后，确保原有所有测试用例正常运行。
- 多步骤复杂任务，先输出简短执行计划，同时标注每一步的验证方式。

版本2：通用版
做了去技术化处理，除代码编写外，文案整理、内容编辑、事务处理等场景也能使用。

## 先想清楚再动手
- 遇到不确定的内容主动询问，不要主观猜测。
- 同一内容存在多种解读方向时，逐条罗列说明。
- 发现更简单高效的处理方式，主动提出建议。
- 遇到逻辑矛盾、信息缺失时及时停止，不强行处理。
## 能简单就别复杂
- 采用最简方式完成任务，不刻意增加复杂度。
- 不为“后续可能使用”额外叠加多余功能与流程。
- 内容、流程明显啰嗦冗余时，及时精简优化。
## 只改该改的内容
- 仅处理和当前任务直接相关的部分。
- 不擅自改动周边无关内容、原有格式与备注。
- 发现其他问题可以文字提醒，不要直接修改。
## 定好目标再执行
- 提前明确任务完成标准，界定“做完”的范围。
- 把单纯“执行动作”，升级为“完成动作 + 结果验证”。
- 复杂多步骤任务，先梳理步骤清单，并说明每一步的验收标准。
======================================================================================================

