# AGENTS.md — podkop-ng

This file is the operating manual for AI coding agents (Devin, Claude Code, Cursor, etc.) working in this repository. Read it fully before making any changes. Humans should read it too.

## 1. Project identity

- **Owner / fork status:** `BlindMaster24/podkop-ng` is **not a GitHub fork**. It is an independent personal continuation of `itdoginfo/podkop`, mirrored from upstream and developed separately. Do not assume any upstream-PR workflow.
- **Upstream:** `https://github.com/itdoginfo/podkop` (read-only). Upstream maintainer accepts PRs only after coordination in their Telegram chat; we do not target upstream with PRs from this repo.
- **License:** GPL-2.0-or-later. All code added here remains GPL-2.0-or-later. Preserve the original `LICENSE` file and any per-file copyright headers when modifying upstream files.
- **Scope of changes in this fork:** primary focus is **accessibility (a11y)** of the LuCI/web frontend (screen-reader compatibility, keyboard navigation, ARIA, contrast, focus management, simplified themes). Other changes (backend, performance, UX) are also welcome — there is no fixed scope beyond a11y emphasis.
- **Upstream relationship:** treat this as a **separate project**, not as a maintained downstream. There is no obligation to keep upstream-sync compatibility; do not constrain new work for the sake of easy upstream merges. Occasional cherry-picks from `itdoginfo/podkop` may still happen; **the AI agent — not the human owner — is the one expected to perform them when the owner asks**, using the workflow in §4. AGENTS.md keeps those commands ready for that reason.

## 2. What this software does

Podkop is an OpenWrt package that provides domain-based traffic routing through `sing-box` (VLESS, Shadowsocks, WireGuard, etc.). It edits `dnsmasq` configuration at startup, manages a `sing-box` configuration via shell helpers, sets up a custom routing table (`105 podkop` in `/etc/iproute2/rt_tables`), and exposes a LuCI web UI plus a Clash-compatible API dashboard.

**Runtime requirements:** OpenWrt 24.10+, ≥25 MB free space, packages: `sing-box`, `curl`, `jq`, `kmod-nft-tproxy`, `coreutils-base64`, `bind-dig`. Conflicts with `https-dns-proxy`, `nextdns`, `luci-app-passwall`, `luci-app-passwall2`.

## 3. Repository layout

```
.
├── podkop/                  OpenWrt package (backend, ash shell)
│   ├── Makefile             OpenWrt build recipe (depends on sing-box, jq, etc.)
│   └── files/
│       ├── etc/init.d/podkop          init script
│       ├── etc/config/podkop          default UCI config
│       ├── usr/bin/podkop             main entrypoint (~2.7K LoC ash)
│       └── usr/lib/                   shell library modules:
│           ├── constants.sh           version requirements, env, paths
│           ├── helpers.sh             generic helpers, version checks
│           ├── logging.sh             log() wrapper
│           ├── nft.sh                 nftables ruleset
│           ├── rulesets.sh            sing-box rule-set helpers
│           ├── sing_box_config_facade.sh   high-level config API
│           └── sing_box_config_manager.sh  low-level config builder (~1.5K LoC)
│
├── luci-app-podkop/         LuCI app (legacy/runtime UI surface)
│   ├── Makefile
│   ├── htdocs/luci-static/resources/view/podkop/   compiled JS lands here
│   ├── po/{ru,templates}    gettext translations
│   ├── root/etc/uci-defaults
│   └── root/usr/share/{luci/menu.d,rpcd/acl.d}
│
├── fe-app-podkop/           Modern TypeScript frontend (source of luci-app-podkop JS)
│   ├── src/
│   │   ├── main.ts                    tsup entry
│   │   ├── constants.ts
│   │   ├── helpers/                   pure utility functions
│   │   ├── icons/                     SVG renderers
│   │   ├── partials/                  UI fragments
│   │   ├── validators/
│   │   └── podkop/                    feature modules
│   │       ├── api.ts, types.ts, index.ts
│   │       ├── services/
│   │       └── tabs/{diagnostic,...}
│   ├── tests/                         vitest setup + suites
│   ├── package.json                   yarn-managed (do not switch to npm/pnpm)
│   ├── tsup.config.ts                 builds into ../luci-app-podkop/htdocs/...
│   ├── eslint.config.js, .prettierrc
│   ├── *.js                           locale extraction/distribution scripts
│   └── locales/                       generated .pot/.po artifacts
│
├── sdk/                     Dockerfiles for OpenWrt SDK builds (apk + ipk)
├── Dockerfile-apk           top-level package build (apk variant)
├── Dockerfile-ipk           top-level package build (ipk variant)
├── install.sh               end-user installer (handles opkg and apk)
├── String-example.md        example domain/IP rule strings (user docs)
├── README.md                Russian end-user readme
└── .github/                 CODEOWNERS, PR template, CI workflows
```

## 4. Upstream sync workflow (run by the AI agent on demand)

This repo is developed as a separate project; nothing here syncs to upstream automatically or on a schedule. **This section is the procedure the AI agent runs when the owner asks for one or more upstream commits to be pulled in.** The owner is not expected to run these commands themselves.

Upstream is configured as a remote in fresh local clones:

```sh
git remote add upstream https://github.com/itdoginfo/podkop.git
git remote set-url --push upstream DISABLED
```

To pull upstream changes when asked:

```sh
git fetch upstream
git log upstream/main --oneline ^main      # see what is new
git cherry-pick <sha>...                   # selective (default mode)
# or, only if the owner explicitly asks for a bulk merge:
git merge --no-ff upstream/main
```

Default to selective cherry-picks of the specific commits the owner names. Use `merge --no-ff` only on explicit owner request. When resolving conflicts, accessibility-related work in this repo always wins over conflicting upstream changes. Never push to `upstream`.

## 5. Build, test, lint

### 5.1 Frontend (`fe-app-podkop/`)

The frontend is the active surface for accessibility work. **All changes touching `fe-app-podkop/**` must pass the same checks CI runs.**

```sh
cd fe-app-podkop
yarn install --frozen-lockfile
yarn format        # prettier --write src
yarn lint          # eslint src --ext .ts,.tsx
yarn test --run    # vitest, no watch
yarn build         # tsup; outputs to ../luci-app-podkop/htdocs/luci-static/resources/view/podkop
yarn ci            # all of the above, the same gate CI uses
```

Do **not** edit `luci-app-podkop/htdocs/luci-static/resources/view/podkop/main.js` by hand. It is generated by `tsup` (`yarn build`) and patched by `tsup.config.ts`'s `onSuccess` hook (it converts `export {...}` to `return baseclass.extend({...})` for LuCI's classic loader). Edit `fe-app-podkop/src/**` and rebuild.

When `yarn build` modifies tracked files, commit those changes alongside the source change in the same commit.

### 5.2 Localization

Translation pipeline lives in `fe-app-podkop`:

```sh
yarn locales:actualize     # extract calls, regenerate .pot, regenerate .po, distribute
```

Run this whenever you add or modify user-facing strings. The output ends up in `luci-app-podkop/po/{templates,ru}`. Do not hand-edit `.pot` files.

### 5.3 Shell

Backend shell code (`install.sh`, `podkop/files/usr/bin/**`, `podkop/files/usr/lib/**`) is checked by `differential-shellcheck` at severity `error` on PRs. Run locally before pushing:

```sh
shellcheck -S error install.sh podkop/files/usr/bin/podkop podkop/files/usr/lib/*.sh
```

Target shell is `ash` (BusyBox). Do not introduce bashisms (no arrays, no `[[ ]]`, no `<<<`, no `${var,,}`). Use `local` (BusyBox ash supports it). Match the existing modular structure: helpers in `helpers.sh`, logging through `log()`, nftables work in `nft.sh`, sing-box config through `sing_box_config_facade.sh`.

### 5.4 Package builds

Two flavors — `ipk` (legacy `opkg`) and `apk` (modern OpenWrt 24.10+):

```sh
docker build -f Dockerfile-ipk -t podkop:ipk --build-arg PODKOP_VERSION=0.dev .
docker build -f Dockerfile-apk -t podkop:apk --build-arg PODKOP_VERSION=0.dev .
```

CI (`.github/workflows/build.yml`) runs both on tag pushes and produces a GitHub Release with the artifacts. Tagging a release on `main` triggers the full build/release pipeline.

### 5.5 SDK images

`sdk/Dockerfile-sdk-apk` and `sdk/Dockerfile-sdk-ipk` are larger SDK base images used during development; not part of normal CI but useful for reproducing OpenWrt build environments locally.

## 6. Coding conventions (binding for AI agents)

These conventions apply to **every** change made by an AI agent in this repo. Humans may relax them when justified; agents may not.

1. **Language: English everywhere.**
   - All new code identifiers, file/branch names, commit messages, PR titles/descriptions, GitHub issue titles, AGENTS.md, README sections that you add — English only.
   - Do not introduce Russian into code paths that are not user-facing strings. User-facing strings go through the i18n pipeline (`po/templates`).

2. **Comments: avoid them.**
   - Default state of code: **no comments**. Refactor names and structure until the code explains itself.
   - Only add a comment when (a) it documents a non-obvious external constraint (a kernel quirk, a sing-box version requirement, a LuCI API gotcha) **and** (b) the situation cannot be expressed via a clearer name or structure.
   - When unavoidable, comments must be: English, single-line if possible, ≤80 chars, describe the **why**, never the **what**.
   - Never add comments that narrate the diff (`// added accessibility label`, `// fix for issue 42`). That belongs in the commit message.
   - Prefer deleting the existing Russian comment over translating it, unless it carries information that is not available elsewhere.

3. **Commit messages: Conventional Commits, in English, imperative mood.**
   - Format: `type(scope): subject`
     - `type` ∈ `feat | fix | refactor | perf | docs | test | build | ci | chore | style | revert`
     - `scope` (optional) ∈ `fe | luci | backend | install | sdk | ci | docs | a11y | i18n | upstream-sync` or a directory name.
     - `subject`: imperative, no trailing period, ≤72 chars.
   - Body (optional, after a blank line): wrap at 72 chars, explain **why**, not **what**.
   - Footer (optional): `BREAKING CHANGE:`, `Refs:`, `Co-authored-by:`.
   - Examples:
     - `feat(a11y): add aria-live to diagnostic toast region`
     - `fix(fe): restore focus to settings tab after save`
     - `chore(upstream-sync): cherry-pick itdoginfo@a1b2c3d`
     - `refactor(backend): extract dnsmasq mutation into helpers.sh`
   - One logical change per commit. Do not batch unrelated edits.
   - Never amend pushed commits. Never force-push to `main`.

4. **Branches.**
   - `main` is the default branch and tracks the integrated state.
   - Feature branches: `feat/<short-kebab>`, `fix/<short-kebab>`, `chore/<short-kebab>`, `a11y/<short-kebab>`.
   - Upstream sync branches: `sync/upstream-<yyyy-mm-dd>` or `sync/itdoginfo-<tag>`.

5. **PRs and reviews.**
   - Open PRs against `main`. Squash-merge unless the branch is itself a curated history (e.g. an upstream sync).
   - PR title follows the same Conventional Commits format as commits.
   - PR description must include: motivation, summary of changes, test evidence (output of `yarn ci` or screenshots / screen-reader recordings for a11y work).

6. **Frontend specifics (a11y focus).**
   - Every interactive element must have an accessible name (visible label, `aria-label`, or `aria-labelledby`).
   - Every form field must have a programmatically associated label (`<label for>` or wrapping `<label>`).
   - All custom widgets must implement keyboard interaction matching the WAI-ARIA Authoring Practices for that pattern.
   - Dynamic regions that announce status (toasts, diagnostic results) must use `role="status"`, `role="alert"`, or `aria-live` as appropriate.
   - Focus must be visible and never trapped unless it is a modal; modals must restore focus on close.
   - Color must not be the sole carrier of information; meet WCAG 2.1 AA contrast (≥4.5:1 normal text, ≥3:1 large/UI components).
   - Test with at least one screen reader (Orca, NVDA, or VoiceOver) before merging a11y-tagged changes; attach a recording or transcript to the PR.

7. **Do not touch without coordination.**
   - `LICENSE`, copyright headers in upstream files.
   - `.github/CODEOWNERS` (currently `@itdoginfo`); replace with the new ownership only as a deliberate, separate commit.
   - `podkop/Makefile` `PKG_NAME` — keep it `podkop` so OpenWrt feeds can install over upstream packages, unless we decide to rename the package itself (separate decision).
   - `podkop/files/etc/config/podkop` defaults — changing these breaks user upgrades.
   - `String-example.md` syntax — it is a contract with users.

8. **Secrets and CI.**
   - Never commit `.env`, tokens, or anything matching `*credentials*`.
   - GitHub Actions workflows on this fork inherit from upstream; `secrets.GITHUB_TOKEN` is repo-scoped and safe. Any new secret must be a separately documented repository secret.
   - The build workflow currently expects a `self-hosted` runner only in the upstream wiki repo, not here; this repo's `build.yml` uses `ubuntu-latest`. Keep it that way to avoid relying on private infrastructure.

9. **Filesystem hygiene.**
   - Do not commit `node_modules`, `.idea`, `.DS_Store`, build artifacts outside the patched `main.js`, or anything in `.env`.
   - The `.gitignore` is the source of truth; extend it rather than working around it.

## 7. AI-agent runbook

Concrete protocol when an agent is asked to implement a change:

1. **Plan.** State the goal, list affected files, list the validation steps you will run.
2. **Read first.** Open every file you intend to touch and the closest neighbors. Read `package.json`, the relevant Makefile, and any related test before editing.
3. **Branch.** `git switch -c <type>/<short-kebab>` from up-to-date `main`.
4. **Edit.** Apply the minimal diff. No drive-by formatting. No comment additions unless rule §6.2 applies.
5. **Validate locally:**
   - Frontend changes → `cd fe-app-podkop && yarn ci`.
   - Shell changes → `shellcheck -S error <touched files>`.
   - Translation strings → `yarn locales:actualize` and commit the regenerated artifacts.
   - Build artifact regeneration → run `yarn build` and commit the regenerated `main.js` in the same commit.
6. **Commit.** One Conventional Commit per logical change. English, imperative.
7. **PR.** Open against `main` with the structured description from §6.5.
8. **Wait for CI.** Do not declare done until CI is green. If CI fails, address it in additional commits (not amends).
9. **Sync (only when the owner asks).** When the owner names one or more commits from `itdoginfo/podkop` to pull in, open a `sync/upstream-<date>` branch, cherry-pick those commits (or `merge --no-ff` only on explicit owner request), run the full `yarn ci` plus shellcheck, resolve conflicts (a11y-related conflicts always favor preserving accessibility work), open PR. Do not initiate syncs on your own.

## 8. Things to escalate

Stop and ask the human owner before doing any of the following:

- Renaming the OpenWrt package (`PKG_NAME`).
- Changing the on-disk config schema (`/etc/config/podkop`).
- Removing or changing existing `String-example.md` syntax.
- Replacing the package manager (yarn → npm/pnpm) or build tool (tsup → vite/rollup).
- Adding a non-GPL-compatible dependency.
- Introducing telemetry, analytics, or any phone-home behavior.
- Touching `LICENSE`, copyright headers, or `CODEOWNERS`.
- Force-pushing to `main` or any shared branch.

## 9. Quick reference

| Task                          | Command                                                            |
| ----------------------------- | ------------------------------------------------------------------ |
| Install frontend deps         | `cd fe-app-podkop && yarn install --frozen-lockfile`               |
| Run all frontend checks       | `cd fe-app-podkop && yarn ci`                                      |
| Format frontend               | `cd fe-app-podkop && yarn format`                                  |
| Build frontend bundle         | `cd fe-app-podkop && yarn build`                                   |
| Run frontend tests (no watch) | `cd fe-app-podkop && yarn test --run`                              |
| Update locales after edits    | `cd fe-app-podkop && yarn locales:actualize`                       |
| Lint shell                    | `shellcheck -S error install.sh podkop/files/usr/bin/podkop podkop/files/usr/lib/*.sh` |
| Build ipk package             | `docker build -f Dockerfile-ipk -t podkop:ipk --build-arg PODKOP_VERSION=0.dev .` |
| Build apk package             | `docker build -f Dockerfile-apk -t podkop:apk --build-arg PODKOP_VERSION=0.dev .` |
| Pull upstream changes         | `git fetch upstream && git log upstream/main --oneline ^main`      |

End of AGENTS.md.
