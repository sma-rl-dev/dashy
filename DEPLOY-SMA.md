# Dashy - tester-env deployment (SMA)

Deployment notes for the `dashy` tester environment. This file and the
`tester-env*` files beside it are deployment tooling and live **inside** the
canonical fork, branch `tester-env-baseline`:

| Field | Value |
| --- | --- |
| Fork | https://github.com/sma-rl-dev/dashy (`isFork: true`, parent `Lissy93/dashy`) |
| Fork branch | `tester-env-baseline` |
| Submodule | `apps/dashy` (parent corpus), pinned to the fork tip below |

## Baseline

| Field | Value |
| --- | --- |
| Upstream | https://github.com/Lissy93/dashy |
| Release tag | `4.9.18` (annotated tag `7ae846cbc6568fc063d7293950b83849dd18aae5`) |
| Commit | `fc204e45fae4b0857cdaace68142d4f9f4a3ccd6` (= `refs/heads/master` at triage time) |
| Release-tag decision | Latest stable semver tag, not a prerelease, published as multi-arch Docker images on 2026-11-13; the GitHub *Releases* list only shows minors because `release.yml` creates patch releases as drafts. No reason to fall back to 4.9.0. |
| Build path | Upstream multi-stage `Dockerfile`, unchanged: `node:24-alpine` build stage (`yarn install --frozen-lockfile` + `yarn build`/Vite) -> production-deps stage -> non-root runtime (uid 1000, tini, `PORT=8080`, `/healthz` healthcheck) |

## Quick start

```bash
./tester-env deploy    # build image from source, restore seed, start, wait for /healthz
./tester-env seed      # re-apply the deterministic conf.yml + ops.yml tree (idempotent)
./tester-env verify    # assert served config + write-to-disk path (no browser needed)
./tester-env reset     # remove container/network, restore pristine user-data
./tester-env stop      # stop container, keep user-data
./tester-env logs      # tail container logs
./tester-env status    # container state, URL, user-data path, credentials
```

Options: `--run-id <id>` (isolates compose project, container, data dir) and
`--port <port>` (host port, default `8112`).

## Access

- Host URL: `http://localhost:8112/`
- Browser-visible URL (browser MCP container): `http://host.docker.internal:8112/`,
  or the bridge gateway on this host: `http://172.16.0.1:8112/` (checked from a
  container on the default bridge).
- Dashy builds no absolute URLs, so no origin/base-URL environment variable is
  needed. `scripts/rl-env` sets `TESTER_ENV_BROWSER_ORIGIN` and the CLI only
  prints it.
- Login (client-side built-in auth, `appConfig.auth.users` in `conf.yml`):
  - `admin` / `tester-admin-2026` (`type: admin`, can write config to disk)
  - `viewer` / `tester-viewer-2026` (`type: normal`)
  - Passwords are fixed SHA-256 hashes in the seed; override the *documented*
    credentials with `DASHY_ADMIN_USER`/`DASHY_ADMIN_PASSWORD` only by editing
    the seed hashes (the env vars are informational for the CLI).

## State, seed and reset

- The whole app state is `user-data/`, bind-mounted from
  `<checkout>/.tester-env-data` (`DASHY_DATA_DIR` overrides; `--run-id` suffix for
  parallel runs) at `/app/user-data`.
- Seed tree: `tester-env-seed/user-data/`
  - `conf.yml`: "Northwind Ops" dashboard (theme `nord`, layout `auto`) with
    exactly 4 sections / 14 items / 3 sub-items:
    - Core Services (2 cols, 5 items): Service Desk (alias `helpdesk`),
      Monitoring (provider Grafana), Deployments (provider GitHub), Status Page,
      and the long-title edge case "Incident Communications and Customer Status
      Update Portal".
    - Facilities (1 col, 3 items): Room Booking, IT Requests (3 sub-items: New
      laptop / VPN access / Software licence), Visitor Management.
    - People & Access (1 col, 3 items): Directory, Payroll, Onboarding
      (`displayData.hideForUsers: [viewer]`).
    - Reference (`displayData.collapsed: true`, 3 items): Style Guide, API
      Standards, Design Assets.
  - `ops.yml`: local sub-page "Operations" with 2 sections / 7 items — On-Call
    (Runbook, Escalation Policy, Incident Board) and Infrastructure (Kubernetes,
    Backups, Patch Schedule, Production Access hidden from `viewer`).
  - Built-in auth users: `admin` (type admin) and `viewer` (type normal).
  - All icons are emoji or bundled `simple-icons`; all URLs use the reserved
    `.example` TLD. No favicon/generative CDN, FontAwesome kit, widget, RSS,
    cloud-backup or remote sub-page traffic is reachable from this seed.
  - Dashy's status/ping indicators are deliberately **off** (`appConfig.statusCheck:
    false`, no per-item `statusCheck`): they would HTTP-probe the item URLs, which
    makes the visible state network-dependent. Scenarios must not assert status dots.
  - There are no timestamp/date fields in Dashy's config model, so assertions use
    stable names, counts, tags, aliases and visibility rules instead of dates.
- `seed` and `reset` both restore the seed tree over the bind directory. They
  replace the *contents* but keep the directory inode, because a running
  container holds the bind mount: deleting and recreating the directory leaves
  the container looking at an empty deleted inode.
- `reset` = `docker compose down -v --remove-orphans` + restore seed tree. It
  never removes the image (content-addressed `IMAGE_TAG`s are shared between
  runs; use `scripts/rl-env gc --prune-images`).
- Config backups are disabled (`DISABLE_CONFIG_BACKUPS=true` in
  `docker-compose.tester-env.yml`), so "Write to Disk" leaves no timestamped
  `config-backups/` files inside the web-served directory.

### Seed contract (static manifest)

`tester-env-seed.json` (same directory as the CLI, i.e. `apps/dashy/` in the
corpus) lists the stable requirement ids a scenario may rely on, without spending
a browser run:

| Requirement | Seed guarantee |
| --- | --- |
| `admin-login` / `viewer-login` | `admin`/`tester-admin-2026` (admin), `viewer`/`tester-viewer-2026` (normal), fixed SHA-256 hashes |
| `dashboard-sections` | 4 home sections in order, Reference collapsed |
| `dashboard-items` | 14 home items (5/3/3/3), stable business titles |
| `item-metadata` | provider + description + tags on items (e.g. Grafana/alert history, GitHub) |
| `search-filter` | single-result queries `grafana`→Monitoring, `helpdesk`→Service Desk (alias), `vpn`→IT Requests (sub-item), `alert history`→Monitoring; `ops`→4 items |
| `sub-items` | IT Requests has 3 named sub-items |
| `role-visibility` | Onboarding and Production Access hidden from `viewer` (13 home / 6 Operations items for viewer) |
| `sub-page-navigation` | Operations sub-page: 2 sections / 7 items |
| `collapsible-section` | Reference collapsed until expanded |
| `long-title-edge-case` | 55-character item title in Core Services |
| `config-edit-persistence` | admin Write to Disk persists through reload (bind mount writable, backups off) |
| `deterministic-reset` | state is only the two seed YAML files; reset/seed restore them byte for byte |
| `service-status-indicators` | **absent** by design (network egress), do not assert |

### Verification (`./tester-env verify`)

Runs against the live instance, no browser needed, 25 checks:

- `/healthz` ok, `/` SPA shell, `/schema.json` served.
- `/conf.yml` and `/ops.yml` are **byte-identical** to the seed files
  (`curl | cmp`), which is stronger than "contains the title" and exposes any
  editor drift.
- Manifest-backed structure: 4 home sections, 14 items, 3 sub-items, 2 auth
  users, the four section names, Reference `collapsed: true`, `helpdesk` alias,
  one `hideForUsers: viewer` rule per page, Operations = 2 sections / 7 items.
- `POST /config-manager/save` with the seed config returns `{"success":true}`;
  `seed` then restores the exact tree (`diff -r` empty), `/conf.yml` is the seed
  again, and no `config-backups/` directory exists.

### Determinism evidence

Two full reset cycles on 2026-11-15, each: drift via the app's own
`/config-manager/save` (Monitoring renamed to "Monitoring Smoke", an extra
"Smoke Section" added, plus a stray `config-backups/` dir and `smoke-page.yml`
written into user-data) → `seed` → `diff -r tester-env-seed/user-data
.tester-env-data` empty → `reset` → `deploy` → `verify` 25/25.

| Step | Result |
| --- | --- |
| Cycle 1 `seed` after drift | tree identical, served `Monitoring Smoke` count 0 |
| Cycle 1 `reset` + `deploy` + `verify` | 25/25 ok, tree identical |
| Cycle 2 `seed` after drift | tree identical |
| Cycle 2 `reset` + `deploy` + `verify` | 25/25 ok, tree identical |
| Final seed checksums | `conf.yml 5febe775…`, `ops.yml b980488f…` (same every reset) |

Server log after every start: `Config file is valid, no issues found`.

### Verified reset gate

1. `POST /config-manager/save` with an edited `conf.yml` (title changed, extra
   section and item) returned `{"success":true}` and `GET /conf.yml` served the
   edit — the visual editor's disk-write path works for the container user.
2. `./tester-env seed` while running restored the baseline and the dashboard
   served "Northwind Ops" again.
3. `./tester-env reset` then `diff -r tester-env-seed/user-data .tester-env-data`
   -> identical; `./tester-env deploy` + `verify` passed afterwards.

### Browser proof (browser-checker, 2026-11-15)

Two browser sessions by the `browser-checker` agent against
`http://host.docker.internal:8112/` (browser tools only; no screenshots or files
captured).

**Session 1 (original smoke, 3-section seed).**

- Admin login succeeded.
- Landing page showed "Northwind Ops" with 3 sections: Core Services, Facilities,
  Reference.
- Typing "Monitoring" in the search filtered the dashboard to that item.
- Edited Monitoring -> "Monitoring Smoke", clicked Save to Disk, reloaded, and
  the change persisted.
- Reverted Monitoring to its seeded title, saved, reloaded, and the revert
  persisted.
- Navigated to the Operations sub-page (`/home/operations`) and saw On-Call
  (Runbook, Incident Board) and Infrastructure (Kubernetes, Backups).

**Session 2 (expanded 4-section seed, PASS).**

- Admin login (`admin` / `tester-admin-2026`) landed on "Northwind Ops" with
  **4 home sections / 14 items**: Core Services, Facilities, People & Access,
  Reference; Reference rendered **collapsed initially** and expanded to 3 docs
  items on click.
- Facilities -> IT Requests showed **3 tooltip-labelled sub-items**.
- Search results (each query cleared afterwards): `grafana` -> Monitoring only;
  `alert history` -> Monitoring only; `helpdesk` -> Service Desk (alias);
  `vpn` -> IT Requests (sub-item match).
- Operations sub-page: On-Call + Infrastructure, **7 items**, including
  Production Access.
- Signed out and logged in as `viewer` / `tester-viewer-2026`: **Onboarding and
  Production Access were absent**, both of which the admin session did see.
- No "Monitoring Smoke" / smoke-seed names remain: the browser-visible tree
  matches the pristine seed after the reset cycles.

## Build cost (measured on this host)

| Step | Time |
| --- | --- |
| Cold deploy (`deploy` incl. `yarn install` + Vite build + start + health) | ~87 s |
| Source-edit rebuild (`COPY . .` + `yarn build`, yarn layers cached) | ~12 s |
| No-op redeploy (all layers cached) | ~2 s |

The dependency layer (`package.json` + `yarn.lock`) is a separate Docker layer,
so mutations to `src/` only re-run the Vite build. `.tester-env-data*` is
excluded in `.dockerignore` so UI-driven config edits never invalidate the build
context.

## rl-env integration

The CLI follows `docs/APP_CLI_PATTERN.md`: it honours `IMAGE_TAG` (never bakes
`RUN_ID` into the image name) and scopes container/network/data dir by `RUN_ID`.

`scenario.meta.json` needs:

```json
"deploy": {
  "app_cli": "tester-env",
  "copy_paths": ["tester-env-seed"],
  "default_port": 8112,
  "url_template": "http://localhost:{port}/"
}
```

- `tester-env-seed` **must** be in `copy_paths`: `scripts/rl-env` copies the CLI
  plus `copy_paths` (and `docker-compose.tester-env.yml` automatically) into the
  run worktree. Without it, `deploy` fails with a pointer to this requirement.
- `RUN_ID`+`--port` isolation and the `scripts/docker-run-scope` ownership
  labels were verified with a second concurrent run
  (`--run-id sim2 --port 18113`, labels `tester-env.project/app/run-id` present
  on container and network).

## Caveats

- **Auth is client-side only** (Dashy's default built-in auth): the login screen
  gates the SPA, but server endpoints (`/config-manager/save`, `/system-info`,
  `/status-check`, ...) are not protected. Set `ENABLE_HTTP_AUTH=true` in the
  compose service for server-side enforcement; `verify` would then need the
  session token.
- **Server-side update check still egresses once per container start**
  (`services/utils/update-checker.js` fetches `raw.githubusercontent.com` with no
  env gate; `appConfig.disableUpdateChecks` only stops the frontend hourly
  check). It is async, non-blocking and only logs a line; on an isolated host it
  logs "Unable to check for updates". Suppressing it would require patching
  app source.
- Dashy keeps a second config layer in browser `localStorage` ("Save" local
  mode, theme/layout preferences). `reset` restores files on disk only; start a
  run from a clean browser profile if a previous run saved locally.
- The container runs as the invoking uid/gid. If the checkout is owned by a
  different user than the one running Docker, set `DASHY_UID`/`DASHY_GID`.
- `node:24-alpine` floats and the Dockerfile runs `apk upgrade`, so the image is
  not bit-reproducible; pin digests only if a scenario needs it.
- Scenario authors should test CRUD through the visual editor (or the
  `/config-manager/save` endpoint for setup), never by hand-editing `conf.yml`
  as the tested workflow.

## Open items

- None blocking. The expanded seed (4 home sections / 14 items, Operations =
  2 sections / 7 items) is confirmed in a browser (session 2 above), and the
  original 3-section smoke (session 1) covers the admin edit/save/reload flow.
- Status-indicators, update-check suppression and the localStorage caveat above
  stay documented limits, not open tasks.
- Move to regression mining: derive mutant patches against baseline
  `fc204e45fae4b0857cdaace68142d4f9f4a3ccd6`; no scenario exists yet.
