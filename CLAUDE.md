# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

ezBookkeeping is a self-hosted personal finance app: a Go backend (Gin + XORM, entry
point `ezbookkeeping.go` → `cmd/`) plus a Vue 3 frontend in `src/`. The built binary
serves the compiled frontend as static files from `static_root_path` (`public/` in a
package build, `dist/` after `npm run build`). The version string in `package.json` is
the single source of truth used by `build.sh`.

## Commands

### Frontend

```bash
npm run serve   # Vite dev server on :8081
npm run lint    # vue-tsc --noEmit && eslint . --fix
npm run test    # vitest run
npm run build   # production build into dist/
```

The dev server proxies `/api`, `/mcp`, `/oauth2`, `/avatar`, `/pictures`, `/icons`,
`/qrcode`, `/proxy`, `/_AMapService` and `/server_settings.js` to a backend at
`127.0.0.1:8080`, so run the Go server alongside it.

Single test file / single case:

```bash
npx vitest run src/lib/__tests__/math.test.ts
npx vitest run -t "<test name>"
```

### Backend

```bash
go run ezbookkeeping.go server run     # add --conf-path to point at another ini
go vet ./...                           # the lint gate build.sh uses
go test ./...
go test ./pkg/services -run TestAccount -v   # single package / single test
```

CGO is required (`github.com/mattn/go-sqlite3`), so `gcc` must be installed.

Other verbs on the binary: `database update`, `userdata <...>` (`user-add`,
`user-modify-password`, `user-session-*`, `transaction-import`, `transaction-export`,
…), `cron`, `security`, `utility`.

### Whole-project builds

```bash
./build.sh backend | frontend | package | docker   # build.bat / build.ps1 on Windows
```

Options: `-r` release, `-o <file>` package name, `-t <tag>` docker tag, `--no-lint`,
`--no-test` (or `SKIP_TESTS=<pattern>` to skip specific Go tests). `backend` runs
`go vet ./...` + `go test ./...` and `frontend` runs `npm run lint` + `npm run test`
before compiling, so both gates must pass.

## Backend architecture

Request path: `cmd/webserver.go` → `pkg/api` → `pkg/services` → `pkg/models` →
`pkg/datastore`.

- **Routes are registered by hand in `cmd/webserver.go`**, under `/api` and `/api/v1`,
  each wrapped by a binder (`bindApi`, `bindApiWithTokenUpdate`, `bindImage`,
  `bindProxy`, `bindCachedJs`, defined at the bottom of that same file). Adding an
  endpoint means a handler in
  `pkg/api/<area>.go` *and* one line in `cmd/webserver.go`. Handler signatures are the
  `*HandlerFunc` types in `pkg/core/handler.go`, e.g.
  `func(*core.WebContext) (any, *errs.Error)`.
- **Singleton-container DI.** Infrastructure lives in package-level containers —
  `settings.Container`, `datastore.Container`, `duplicatechecker.Container`,
  `uuid.Container`, `storage.Container`, `mail.Container`, `avatars.Container`. APIs
  and services reach them by embedding helper structs (`ApiUsingConfig`,
  `ApiUsingDuplicateChecker`, `ApiUsingAvatarProvider` in `pkg/api/base.go`;
  `ServiceUsingDB`, `ServiceUsingConfig`, `ServiceUsingStorage`, `ServiceUsingUuid`,
  `ServiceUsingMailer` in `pkg/services/base.go`), and are themselves singletons
  (`api.Accounts`, `services.Accounts`). Follow that pattern for new APIs/services
  rather than introducing constructor injection.
- `pkg/datastore` exposes three logical stores — `UserStore`, `TokenStore`,
  `UserDataStore` — reached through `UserDB()`, `TokenDB(uid)`, `UserDataDB(uid)`.
  Use the one matching the data you touch.
- `pkg/core` defines four `Context` flavors (`WebContext`, `CliContext`,
  `CronContext`, `NullContext`); every `log.*` call takes a `core.Context` first.
- Errors are typed values in `pkg/errs`; handlers surface them as
  `errs.Or(err, errs.ErrOperationFailed)`.
- Gin binding validators (`notBlank`, `validCurrency`, `validTransactionAmount`,
  `validTagFilter`, …) live in `pkg/validators` and only work in struct tags once
  registered in `cmd/webserver.go`.
- `pkg/converters/` holds one subpackage per import/export format (ofx, qif, camt, mt,
  iif, beancount, gnucash, fireflyIII, alipay, wechat, jdcom, feidee, custom csv/excel,
  ai). Dispatch happens in `pkg/converters/transaction_data_converters.go` — a new
  format must be wired there.
- `pkg/mcp` implements the MCP tool handlers (one file per tool); `pkg/llm` holds LLM
  providers for receipt/text recognition, whose prompts are Go templates in
  `templates/prompt/*.tmpl`.
- `pkg/settings` parses `conf/ezbookkeeping.ini` (sections: server, mcp, database,
  mail, log, storage, llm, security, auth, user, map, exchange_rates, …).
  `pkg/locales` is *server-side* i18n (emails), separate from the frontend locales.

## Frontend architecture

**Two separate SPAs are built from one `src/`**: a desktop app on Vuetify
(`src/desktop.html` + `desktop-main.ts` + `DesktopApp.vue`) and a mobile app on
Framework7 (`src/mobile.html` + `mobile-main.ts` + `MobileApp.vue`), plus an
`index.html`/`index-main.ts` chooser. All are declared as separate rollup inputs in
`vite.config.ts`. Views are split into `src/views/desktop/` and `src/views/mobile/`
with shared pieces in `src/views/base/`; `src/components` mirrors that split.

Layering, roughly bottom-up:

- `src/consts` — constants and API paths
- `src/core` — framework-free domain types and enums
- `src/models` — API request/response types, mirroring `pkg/models`
- `src/lib` — pure helper logic; this is where most unit tests live
- `src/stores` — Pinia stores, aggregated by `src/stores/index.ts`
- `src/views` / `src/components` — per-platform UI

`src/lib/services.ts` is the single axios layer for every backend call, with paths from
`src/consts/api.ts`; add endpoints there rather than calling axios from components.

i18n: `src/locales/<tag>.json` registered in the `ALL_LANGUAGES` map in
`src/locales/index.ts` (each entry carries `textDirection`, so RTL is supported).

Tests are colocated as `__tests__/*.test.ts` under `src/lib`, `src/core` and
`src/models`, and run in vitest's `node` environment.

`@` aliases to `src/`. `tsconfig.json` is strict with `noUncheckedIndexedAccess` and
`noPropertyAccessFromIndexSignature`, so indexed access needs guarding and
index-signature properties need bracket access.

## Conventions

- `.editorconfig`: 4-space indentation, tabs in `.go`, 2 spaces in `package.json`.
- Go exported identifiers carry doc comments starting with the identifier name; log
  messages use the `[file.FunctionName] message` prefix convention.
- A backend feature typically touches `pkg/models` (entity + request/response types),
  `pkg/services`, `pkg/api`, `cmd/webserver.go`, and then the mirrored `src/models`
  and `src/lib/services.ts` on the frontend.
- `skills/ezbookkeeping/` is a shipped end-user Agent Skill wrapping the public API via
  `scripts/ebktools.sh` / `.ps1` — a product artifact, not dev tooling.
