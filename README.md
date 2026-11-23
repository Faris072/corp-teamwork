# corp-teamwork

An open-source (MIT), token-efficient 5-role multi-agent teamwork protocol plugin for Antigravity (`agy`).

## Roles

1. **Lead / Orchestrator**: Breaks down tasks, performs fast read-only triage (`rg`/`read_file`), delegates to workers, and reports final results to the user.
2. **Architect & Researcher**: Designs system architecture, investigates complex bugs, and writes blueprints.
3. **Builder**: Implements clean, modular code changes according to the brief.
4. **Guardian**: Verifies builds/tests, reviews code quality, and checks security.
5. **Assistant**: Handles lightweight tasks (Q&A, status checks, config/doc updates, minor tweaks).

## Installation

```bash
git clone https://github.com/<your-org>/corp-teamwork.git
agy plugin install ./corp-teamwork
agy plugin enable corp-teamwork
```

> Note: `<your-org>` is an intentional placeholder for forks/self-hosted clones (replace with `Faris072` for upstream).

## License

[MIT](./LICENSE)
