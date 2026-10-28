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
