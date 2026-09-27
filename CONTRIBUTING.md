# Contributing to Orqen

First off — thank you. Orqen is a community project and every contribution matters.

Read this document before opening an issue or submitting a pull request.
It will save you and the maintainers time.

---

## Before You Start

- Read [`orqen-docs/00-INDEX.md`](./orqen-docs/00-INDEX.md) to understand the project structure
- Read [`orqen-docs/DECISIONS.md`](./orqen-docs/DECISIONS.md) before touching any core component
- Read [`orqen-docs/03-ARCHITECTURE.md`](./orqen-docs/03-ARCHITECTURE.md) to understand the layer boundaries
- Read [`orqen-docs/11-PROJECT-ISOLATION.md`](./orqen-docs/11-PROJECT-ISOLATION.md) for any feature touching multiple projects

If you skip these and submit a PR that reverses a deliberate architectural decision,
it will be closed with a pointer to the relevant doc.

---

## Ways to Contribute

- **Bug reports** — open an issue with the bug report template
- **Feature requests** — open an issue with the feature request template
- **Documentation improvements** — fix errors, improve clarity, add examples
- **New language SDKs** — implement the spec in `orqen-docs/10-SDK-SPEC.md`
- **Core server contributions** — see the roadmap and open issues labeled `good first issue`
- **Tests** — more coverage is always welcome

---

## Getting Started

### 1. Fork and clone

```bash
git clone https://github.com/YOUR_USERNAME/orqen.git
cd orqen
```

### 2. Set up your environment

Follow [`orqen-docs/13-DEVELOPER-GUIDE.md`](./orqen-docs/13-DEVELOPER-GUIDE.md) for full setup instructions.

### 3. Create a branch

```bash
git checkout -b feat/your-feature-name
# or
git checkout -b fix/your-bug-description
```

Branch naming:
- `feat/` — new feature
- `fix/` — bug fix
- `docs/` — documentation only
- `test/` — tests only
- `refactor/` — code restructure with no behavior change

---

## Making Changes

### Code style

- Follow standard Go conventions (`gofmt`, `golint`)
- All exported functions and types must have doc comments
- No `panic()` in production code paths — always return errors
- Wrap errors with context: `fmt.Errorf("package: operation: %w", err)`
- Handlers must read `project_id` from request context — never from request body

### Isolation rule (mandatory)

Any feature that adds a new project-scoped resource must pass the checklist in
`orqen-docs/11-PROJECT-ISOLATION.md`. PRs that skip this will not be merged.

### Tests

- Run `go test -race ./...` before every commit — the race detector is mandatory
- New features must include tests
- Bug fixes must include a test that would have caught the bug
- Test coverage should not decrease

```bash
go test -race ./...
```

---

## Submitting a Pull Request

1. Make sure `go test -race ./...` passes
2. Make sure `golangci-lint run ./...` passes with no new warnings
3. Keep PRs focused — one concern per PR
4. Fill in the PR template completely
5. Link the issue your PR addresses (if applicable)
6. Do not squash your commits before review — maintainers may want to see the history

### PR title format

```
feat: add weighted acquire strategy support
fix: recover jobs on worker heartbeat timeout
docs: clarify CRS Exclusive policy in 07-CRS-INTEGRATION
test: add DAG cycle detection edge cases
```

---

## Commit Message Format

```
<type>: <short description>

<optional longer explanation>

<optional: Refs #issue-number>
```

Types: `feat`, `fix`, `docs`, `test`, `refactor`, `chore`

Keep the first line under 72 characters.

---

## Architectural Decisions

Do not open a PR that reverses or works around a decision in `DECISIONS.md` without
first opening an issue to discuss it. The decisions log exists precisely to prevent
well-intentioned but misaligned changes.

If you believe a decision should be revisited, open an issue titled
`[DECISION] Reconsider D-00X: <title>` and make your case. That's the right process.

---

## Reporting Bugs

Open an issue using the Bug Report template. Include:

- What you expected to happen
- What actually happened
- Steps to reproduce
- Orqen version and Go version
- Relevant logs (redact any secrets or payloads)

---

## Suggesting Features

Open an issue using the Feature Request template. Before you do, check:

- Is this feature in V1 scope? See `orqen-docs/02-REQUIREMENTS.md`
- Is this feature explicitly a non-goal? See `orqen-docs/01-PROJECT.md`
- Has someone already requested it?

---

## Building a New Language SDK

Orqen SDKs are independent repositories. To build one:

1. Read `orqen-docs/10-SDK-SPEC.md` fully — this is the contract your SDK must implement
2. Read `orqen-docs/05-WORKER-PROTOCOL.md` — the HTTP protocol your SDK speaks
3. Use the compliance checklist at the end of `10-SDK-SPEC.md` as your acceptance criteria
4. Open an issue to announce you're working on a new SDK so we can coordinate

---

## Code of Conduct

By participating in this project you agree to abide by the
[Code of Conduct](./CODE_OF_CONDUCT.md).

---

## License

By contributing to Orqen, you agree that your contributions will be licensed
under the [MIT License](./LICENSE).
