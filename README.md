# agent-skills
agent-skills


https://github.com/anthropics/skills
https://www.skills.sh/

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

