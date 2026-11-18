# Contributing to vast-on-demand

Thank you for your interest in contributing to `vast-on-demand`.

## Reporting Issues

When opening a bug report, please include:

- Your OS and Python version (`python3 --version`).
- How you started the orchestrator (`./start-orchestrator.sh` or `python3 orchestrator/main.py ...` flags).
- Relevant log output (with any API keys, tokens, or IP addresses redacted).
- Steps to reproduce the issue.

## Submitting Pull Requests

1. Fork the repository and create a branch from `main`.
2. Keep changes focused and avoid adding third-party Python dependencies—the orchestrator intentionally uses the Python standard library only.
3. Add or update unit tests in [`tests/`](tests/) for any behavior changes.
4. Verify your changes locally before opening a PR:
   ```bash
   python3 -m unittest discover -s tests -v
   bash -n entrypoint.sh start-orchestrator.sh deploy/ssh_tunnel.sh
   ```
5. Make sure no secrets, `.env` files, or local machine paths are included in your diff.
