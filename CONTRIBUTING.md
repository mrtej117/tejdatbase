# Contributing to the AI Travel Assistant

First off, thank you for considering contributing to the AI Travel Assistant! It's people like you that make this open-source project such a great tool.

## 1. Code of Conduct
By participating in this project, you agree to maintain a respectful, inclusive, and professional environment for all contributors.

## 2. Contribution Workflow
We follow a standard Git Feature Branch Workflow.

1. **Fork & Clone:** Fork the repository on GitHub, then clone it to your local machine.
2. **Branch:** Create a branch for your feature or bug fix: `git checkout -b feature/add-new-endpoint` or `git checkout -b fix/postgres-connection-leak`.
3. **Develop:** Make your changes locally. Refer to [20_Development_Setup.md](docs/20_Development_Setup.md) to boot the local database stack.
4. **Test:** Write unit tests and verify your code locally. Ensure all existing tests pass.
5. **Commit:** Write meaningful, descriptive commit messages.
6. **Push:** Push the branch to your fork.
7. **Pull Request (PR):** Open a PR against our `main` branch. 

## 3. Pull Request Guidelines
- **Keep it small:** PRs should do one thing and do it well. If you are adding a new feature and fixing an unrelated bug, open two separate PRs.
- **Link issues:** If your PR fixes an open issue, include `Fixes #123` in the PR description.
- **Documentation:** If you change the database schema, you *must* update the relevant Markdown documentation in the `/docs` folder (e.g., [08_Database_Schema.md](docs/08_Database_Schema.md)).

## 4. Coding Standards

### Backend (Python / FastAPI)
- **Formatting:** We strictly use `black` for code formatting and `isort` for import sorting. Run them before committing.
- **Typing:** Type hints are mandatory. Run `mypy` to ensure type safety.
- **Naming Conventions:** Use `snake_case` for variables, functions, and database columns. Use `PascalCase` for Classes and SQLAlchemy Models.

### Database (PostgreSQL / Alembic)
- **Migrations:** Never modify database schema using raw SQL in pgAdmin. You must generate an Alembic migration script.
- **Naming:** All tables and columns must use `snake_case`.

## 5. Documentation Standards
If you are contributing to this documentation suite:
- **Tone:** Professional, clear, and educational (beginner to production-level).
- **Format:** Use standard GitHub Flavored Markdown.
- **Diagrams:** Use *only* GitHub-compatible Mermaid diagrams (`flowchart`, `sequenceDiagram`, `erDiagram`). Do not use experimental syntax like `architecture-beta`.
- **Linking:** Cross-link to other documents heavily to avoid repeating information.

## 6. Getting Help
If you are stuck, please open a Draft Pull Request and tag a maintainer. We are happy to help you finish your contribution!
