LLM-Helps: Project-Agnostic Agent Guides

- Purpose: This repo holds opinionated AGENTS.md playbooks you can reuse across projects. They instruct your coding agent how to write code for these kinds of projects (Django, React/TS), not how to build any one specific app.

- How to use: Symlink the relevant AGENTS.md into a project’s root so the agent treats it as the top-level directive for that project.

Examples
- Django project: ln -s "$(pwd)/django/AGENTS.md" /path/to/your-django-app/AGENTS.md
- React project:  ln -s "$(pwd)/react/AGENTS.md"  /path/to/your-react-app/AGENTS.md

Scope & Behavior
- The AGENTS.md applies to the entire directory tree rooted where it resides. Place the symlink in the project root to cover the whole project.
- If multiple AGENTS.md exist in nested folders, the most deeply nested file takes precedence for files under its folder.
- Direct user/dev instructions always override AGENTS.md.

What these guides do
- Django guide: Defines stack assumptions (Postgres primary, Mongo auxiliary), service/view boundaries, OpenAI REST view patterns, testing, linting, CI, and a reference project structure. It tells the agent how to implement features safely, typed, and with tests.
- React guide: Defines a shadcn/ui-first, Client/Service split, strict TypeScript, accessibility, testing (Vitest/RTL), and component generation rules. It tells the agent how to scaffold components, wire services, and enforce consistency.

Notes for Agents
- Treat AGENTS.md as coding policy. Follow it verbatim unless the user says otherwise.
- Ask for clarifications first, propose a short plan, then implement minimal, high-quality changes with tests and linting.
- Prefer safety, typing, and small PR-sized increments.

Maintenance
- Keep these AGENTS.md files concise and opinionated. Update them when conventions change; projects that symlink them benefit immediately.

