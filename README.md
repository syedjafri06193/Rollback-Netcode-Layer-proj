# Rollback Netcode Layer

Deterministic lockstep networking with rollback, client-side prediction, and lag compensation for 2–8 player matches.

> **Status: design phase.** This repository holds the complete design and
> build guide; implementation has not started yet.

## Documentation

All documentation lives in [`1—Documentation/`](1—Documentation/README.md).

- [Design & build guide](1—Documentation/design-guide.md)
- [rollback-netcode-layer](1—Documentation/rollback-netcode-layer.pdf) (PDF)

## What the guide covers

1. Executive summary and scope
2. Reality check
3. Determinism ★
4. State save and restore ★
5. The rollback loop
6. Rollback-safe simulation ★
7. Transport and topology
8. Desync detection and diagnosis ★
9. The server-authoritative alternative
10. Performance
11. Tech stack and setup
12. Repository layout
13. Milestone ladder
14. Reference implementations
15. Testing
16. Stretch goals
17. References

- Appendix A — Decision record
- Appendix B — Quick reference card

<!-- project-metrics:start -->
## Project Metrics

[![Stars](https://img.shields.io/github/stars/syedjafri06193/Rollback-Netcode-Layer-proj?style=flat-square)](https://github.com/syedjafri06193/Rollback-Netcode-Layer-proj/stargazers) [![Forks](https://img.shields.io/github/forks/syedjafri06193/Rollback-Netcode-Layer-proj?style=flat-square)](https://github.com/syedjafri06193/Rollback-Netcode-Layer-proj/forks) [![Open issues](https://img.shields.io/github/issues/syedjafri06193/Rollback-Netcode-Layer-proj?style=flat-square)](https://github.com/syedjafri06193/Rollback-Netcode-Layer-proj/issues) [![Last commit](https://img.shields.io/github/last-commit/syedjafri06193/Rollback-Netcode-Layer-proj/improvements?style=flat-square)](https://github.com/syedjafri06193/Rollback-Netcode-Layer-proj/commits/improvements) [![Commits](https://img.shields.io/github/commit-activity/t/syedjafri06193/Rollback-Netcode-Layer-proj?style=flat-square)](https://github.com/syedjafri06193/Rollback-Netcode-Layer-proj/commits/improvements) [![Top language](https://img.shields.io/github/languages/top/syedjafri06193/Rollback-Netcode-Layer-proj?style=flat-square)](https://github.com/syedjafri06193/Rollback-Netcode-Layer-proj) [![Repo size](https://img.shields.io/github/repo-size/syedjafri06193/Rollback-Netcode-Layer-proj?style=flat-square)](https://github.com/syedjafri06193/Rollback-Netcode-Layer-proj)

### Code

| Metric | Value |
|---|---|
| Lines of code | — (design-docs repo) |
| Source files | 0 |
| Tracked files | 4 |
| Documentation | 3 Markdown files (8,504 words), 1 PDF |
| Commits | 6 |
| Contributors | 1 |
| Active | 2026-09-11 → 2026-10-28 |

<sub>Code stats as of 2026-10-28, counted with pygount (comments and blank lines excluded). Badges update live.</sub>
<!-- project-metrics:end -->
