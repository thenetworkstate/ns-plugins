---
name: ns-git-workflow
description: Git workflow conventions for the NS monorepo. Use when committing, merging, rebasing, resolving merge conflicts, pushing, creating PRs, fixing pre-commit hook failures, handling migration conflicts, or managing lock files. Triggered by git operations, commit messages, branch management, and CI check failures.
metadata:
  author: ns
  version: "1.0.0"
---

# NS Git Workflow

## Branching Model

- Default branch is **`staging`**, not `main`. All PRs target `staging`.
- Feature branches follow Conventional Commit prefixes: `feat/*`, `fix/*`, `docs/*`, `refactor/*`, etc.
- Never force-push to `staging` or `main`.
- Push feature branches with `git push -u origin <branch>`.

## Commit Message Format

Enforced by commitizen (`.commitlintrc.json`):

```
<type>(<optional-scope>): <subject>
```

| Rule | Detail |
|------|--------|
| Types | `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert` |
| Subject case | Never `UPPER-CASE`, `PascalCase`, or `Start Case` |
| Header max length | 100 characters |
| Trailing period | None |

Always pass the commit message via HEREDOC:

```bash
git commit -m "$(cat <<'EOF'
feat(auth): add OAuth2 token refresh flow
EOF
)"
```

If the commitizen hook rejects the message, fix the issue and create a **new** commit. Never amend — the failed commit didn't happen, so `--amend` would modify the previous (unrelated) commit.

## What to Stage

- Stage specific files by name. Avoid `git add -A` or `git add .`.
- **Never commit**: `.env`, secrets, `node_modules/`, `.next/`, `dist/`, `build/`, `.venv/`
- **Lock files** (`pnpm-lock.yaml`, `uv.lock`): only stage when dependencies were intentionally changed.
- **Migrations**: before committing, verify the migration number doesn't conflict with what's on `staging`. Check with `git log staging -- apps/api/<app>/migrations/`.

## Pre-commit Hooks

Installed via `pnpm pre-commit:install`. Run manually with `pnpm pre-commit:run`.

### Hook Reference

| Hook | Scope | Auto-fixes? | Common failure | Fix |
|------|-------|-------------|----------------|-----|
| **trailing-whitespace** | All files | Yes | Whitespace trimmed | `git add` fixed files, re-commit |
| **end-of-file-fixer** | All files | Yes | Missing newline added | `git add` fixed files, re-commit |
| **check-yaml** | `*.yaml` | No | Invalid YAML | Fix syntax |
| **check-json** | `*.json` | No | Invalid JSON | Fix syntax |
| **check-toml** | `*.toml` | No | Invalid TOML | Fix syntax |
| **check-merge-conflict** | All files | No | Leftover `<<<<<<<` markers | Resolve conflicts |
| **check-case-conflict** | All files | No | Two files differ only in case | Rename one |
| **mixed-line-ending** | All files | Yes | CRLF → LF | `git add` fixed files, re-commit |
| **detect-private-key** | All files | No | Private key in staged file | Remove the key, use Doppler |
| **check-added-large-files** | All files | No | File > 5 MB | Remove or use Git LFS |
| **prettier** | JS/TS/JSON/MD/YAML | Yes | Formatting changed | `git add` fixed files, re-commit |
| **eslint** | `apps/web/**/*.{js,ts,tsx,jsx}` | Yes (partial) | Lint errors or warnings | Fix lint issues; remember `--max-warnings=0` means ANY warning fails |
| **black** | `apps/api/**/*.py` | Yes | Formatting changed | `git add` fixed files, re-commit |
| **isort** | `apps/api/**/*.py` | Yes | Import order changed | `git add` fixed files, re-commit |
| **ruff** | `apps/api/**/*.py` | Yes | Lint violations | Fix issues, `git add`, re-commit. **Gotcha**: `--exit-non-zero-on-fix` means ruff always fails on first run even after auto-fixing — you must re-stage and commit again |
| **detect-secrets** | All (excl. locks) | No | New secret detected | If false positive: update `.secrets.baseline` via `detect-secrets scan --baseline .secrets.baseline`. If real: remove the secret, use Doppler |
| **commitizen** | Commit message | No | Bad message format | Rewrite message following format above |
| **drf-security-checks** | `apps/api/**/*.py` | No | Missing security attributes on DRF views | Add the missing attribute or suppress with `# security: allow-<check>` |
| **metadata-check** | `apps/web/**/*.{ts,tsx}` | No | Page missing metadata | Add metadata export to page |
| **update-protected-paths** | `apps/web/app/(protected)/**/page.*` | Yes | Allowlist file changed | `git add` the updated allowlist, re-commit |
| **gitleaks** | All staged files | No | Secret/token found | Remove secret; if false positive add to `.gitleaksignore` |

### Key Pattern: Auto-fix Hooks

Hooks that auto-fix (prettier, black, isort, ruff, trailing-whitespace, end-of-file-fixer, mixed-line-ending, update-protected-paths) modify files **but do not stage them**. After a failure:

1. Review what changed (`git diff`)
2. Stage the fixes (`git add <files>`)
3. Create a **new** commit (do not amend)

## Merge Conflict Resolution

### Django Migrations

The #1 merge conflict source. **Never** delete one side's migration.

1. Keep BOTH migration files (they have different numbers)
2. Create a merge migration:
   ```bash
   pnpm run docker:makemigrations <app_name> --merge
   ```
3. Stage all three files (both original + the new merge migration)

### Lock Files

| File | Resolution |
|------|-----------|
| `pnpm-lock.yaml` | Delete it, run `pnpm install`, stage the regenerated file |
| `uv.lock` | Delete it, run `cd apps/api && uv lock`, stage the regenerated file |

### Other Common Conflicts

| File | Resolution |
|------|-----------|
| `.secrets.baseline` | Regenerate: `detect-secrets scan --baseline .secrets.baseline` |
| `.gitleaksignore` | Accept both sides, deduplicate entries |
| `protected-paths.ts` | Accept either side, then let the `update-protected-paths` hook regenerate on next commit |

## Push Rules

- **Never push unless the user explicitly asks.**
- Never push directly to `staging` or `main`.
- For feature branches: `git push -u origin <branch-name>`.

## CI Checks on PRs

| Check | Trigger | What it does |
|-------|---------|-------------|
| **Gitleaks** | PR + push to main/staging/dev | Scans full history for leaked secrets |
| **Semgrep** | PR + push to staging | Static analysis for security patterns |
| **Vercel Preview** | PR to staging | Builds and deploys a preview of `apps/web` |

## Common Gotchas Checklist

- **ESLint `--max-warnings=0`**: even a single warning fails the hook. Fix all warnings, don't suppress with `// eslint-disable` unless truly justified.
- **Ruff `--exit-non-zero-on-fix`**: always fails on first pass when it auto-fixes. Re-stage and re-commit.
- **detect-secrets false positives**: update `.secrets.baseline`, don't remove the baseline file.
- **Protected paths auto-update**: if you add/remove a page under `(protected)/`, the hook will regenerate the allowlist — stage the change.
- **API schema drift**: after changing DRF serializers or views, run `pnpm api:schema` and `pnpm api:schema:update` to keep downstream types in sync.
- **DRF security check**: new views need `permission_classes`, `authentication_classes`, `throttle_classes`, and `http_method_names`. Use `# security: allow-<check>` comments to suppress specific findings with justification.
- **Migration numbering**: before committing a new migration, check staging for conflicts: `git log staging -- apps/api/<app>/migrations/`.
