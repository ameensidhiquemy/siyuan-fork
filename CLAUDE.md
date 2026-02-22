# Siyuan Fork — Claude Code Guide

This file provides guidance to Claude Code when working with code in this repository.

## Project Overview

This is a **public AGPL v3 fork of Siyuan** maintained for Purple. It contains UI modifications and feature unlocks that ship with Purple. This repo must remain public to comply with AGPL licensing.

**Upstream:** [siyuan-note/siyuan](https://github.com/siyuan-note/siyuan)
**This fork:** [ameensidhiquemy/siyuan-fork](https://github.com/ameensidhiquemy/siyuan-fork)

### Relationship to Purple

Purple is a **closed-source research tool** that ships a modified Siyuan as its notes/editor component. To stay compliant with AGPL while keeping Purple closed:

- All Siyuan modifications live in this **public** repo
- Purple's backend (Go API, AI services) lives in the **private** `purple` repo
- Purple talks to Siyuan only via HTTP API (`:6806`)
- This repo stands alone — no Purple-specific code import allowed

### What We Modify

- **Subscription gates removed** — Features like cloud sync, WebDAV/S3, snapshots, and reminders are unlocked
- **UI customizations** — Purple branding, layout tweaks, plugin integration points
- **Build configuration** — Custom build flags for Purple distribution

---

## Build & Development Commands

### Prerequisites

```bash
# Install dependencies
pnpm install      # Frontend (app/)
go mod download   # Backend (kernel/)
```

### Build

```bash
# Full build
pnpm build                  # Compile frontend (app/)
pnpm build:go --project /   # Build kernel Go binary (SiYuan-Kernel)

# Dev builds
pnpm start        # Frontend hot-reload on :9876
pnpm start-kernel # Kernel hot-reload on :6806
```

### Running from Source

```bash
# From purple repo (recommended — uses Makefile):
cd ~/PROJECTS/purple
make build-kernel       # Rebuild kernel after Go code changes
make dev-siyuan         # Run kernel in dev mode (:6806)
make dev-siyuan-ui      # Run frontend with hot-reload (:9876)

# Standalone (if not using purple Makefile):
./SiYuan-Kernel --mode dev --port 6806 --accessAuthCode purple123 --workspace ~/siyuan
```

Frontend expects kernel running on `:6806`. If both are running, open `http://localhost:9876` for hot-reload, or `http://localhost:6806` for full stack.

---

## Architecture

### Tech Stack

| Layer | Tech | Path |
|-------|------|------|
| **Frontend** | TypeScript, Svelte, Protyle editor | `app/src/` |
| **Kernel** | Go, HTTP server, SQLite | `kernel/` |
| **Editor** | Protyle (custom block editor) | `app/src/protyle/` |
| **Sync** | Cloud sync, S3, WebDAV | `kernel/model/sync.go` |

### Key Files

```
kernel/
├── model/
│   ├── conf.go              ← IsSubscriber(), IsPaidUser() (patched: always true)
│   ├── sync.go              ← Sync engine
│   └── account.go           ← User account state
├── api/
│   └── router.go            ← HTTP API routes
└── util/
    └── working.go           ← Workspace locking

app/
├── src/
│   ├── util/
│   │   ├── needSubscribe.ts ← Subscription check (patched: always false)
│   │   └── functions.ts     ← isPaidUser() (patched: always true)
│   ├── protyle/             ← Block editor
│   ├── dialog/              ← UI dialogs
│   └── menus/               ← Context menus
└── appearance/themes/       ← CSS themes
```

---

## Purple-Specific Patches

These patches unlock all features for Purple users. They live on feature branches and get merged to `main` for releases.

### Kernel Patches (Go)

**File:** `kernel/model/conf.go`

```go
// Always return true — no subscription check
func IsSubscriber() bool {
    return true
}

func IsPaidUser() bool {
    return true
}
```

### Frontend Patches (TypeScript)

**File:** `app/src/util/needSubscribe.ts`

```ts
// Always return false — no features need subscription
export const needSubscribe = (tip?: string) => false;
export const isPaidUser = () => true;
```

---

## Git Workflow

### Branch Strategy

| Branch | Purpose |
|--------|---------|
| `main` | Stable, matches latest Purple release |
| `feat/*` | Feature branches — one per Purple feature |
| `upstream-sync` | Merge upstream Siyuan updates here first |

### Syncing Upstream

Periodically pull from upstream to get bug fixes and new Siyuan features:

```bash
git remote add upstream https://github.com/siyuan-note/siyuan.git
git fetch upstream
git checkout upstream-sync
git merge upstream/master       # or upstream/dev
git checkout main
git merge upstream-sync         # Resolve conflicts, test
```

### Commit Conventions

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add cloud sync unlock to conf.go
fix: needSubscribe() now returns false correctly
chore: rebuild kernel after Go toolchain update
docs: update CLAUDE.md with new build commands
```

Use co-author tags when AI-assisted:

```
Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

---

## Development Workflow

### Making Changes

1. **Kernel changes (Go):**
   ```bash
   cd ~/PROJECTS/purple
   vi ~/PROJECTS/siyuan-fork/kernel/model/conf.go
   make build-kernel      # Recompile
   make dev-siyuan        # Test
   ```

2. **Frontend changes (TypeScript):**
   ```bash
   vi ~/PROJECTS/siyuan-fork/app/src/util/needSubscribe.ts
   # Changes hot-reload automatically if dev-siyuan-ui is running
   ```

3. **Commit and push:**
   ```bash
   git -C ~/PROJECTS/siyuan-fork add <files>
   git -C ~/PROJECTS/siyuan-fork commit -m "feat: description"
   git -C ~/PROJECTS/siyuan-fork push -u origin <branch>
   ```

### Testing

- **Manual:** Open Siyuan at `http://localhost:6806`, test modified features
- **Automated:** Upstream has Go tests in `kernel/test/`, but we don't run a full test suite yet
- **Focus areas:** Cloud sync UI, WebDAV/S3 settings, snapshot creation, block reminders

### Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| `lock workspace failed` | Another Siyuan process is running | Kill other Siyuan: `pkill SiYuan-Kernel` |
| Frontend 404s | Kernel not running | Start kernel first: `make dev-siyuan` |
| UI changes not updating | Frontend not in watch mode | Run `make dev-siyuan-ui` |
| Go build errors | Module cache stale | `go mod tidy && go mod download` |

---

## Code Style

### Go (Kernel)

- Follow standard Go conventions (`gofmt`, `golint`)
- Use `camelCase` for unexported functions, `PascalCase` for exported
- Error wrapping: `fmt.Errorf("context: %w", err)`
- Keep functions small — extract helpers when >50 lines

### TypeScript (Frontend)

- Use `camelCase` for variables/functions, `PascalCase` for classes/types
- Prefer `const` over `let`, never `var`
- Explicit types for function signatures, infer for local variables
- Component files end in `.svelte`, utilities in `.ts`

### Commits

- Lowercase type prefix: `feat:`, `fix:`, `chore:`, `docs:`
- Imperative mood: "add feature" not "added feature"
- Body optional, but explain **why** for non-trivial changes
- Reference issues: `Closes #123`

---

## Links

- **Upstream repo:** https://github.com/siyuan-note/siyuan
- **Upstream docs:** https://github.com/siyuan-note/siyuan/tree/master/API.md
- **Purple repo:** `~/PROJECTS/purple` (private)
- **This fork:** https://github.com/ameensidhiquemy/siyuan-fork

---

## Critical Rules

- **Never import Siyuan code into the `purple` repo** — AGPL contamination risk
- **Always keep this repo public** — AGPL requires it
- **Document all Purple-specific patches** in this file and commit messages
- **Test both kernel and frontend** after changes — they're tightly coupled
- **Sync upstream regularly** — don't drift too far from mainline Siyuan

---

_Last updated: 2026-02-22 — Subscription gates removed, kernel/frontend patched_
