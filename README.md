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

