# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A small web app for maintaining bucket lists, built so that the owner and their partner (and possibly
others later) can work on lists together. Accounts are isolated by default; content is only visible to
members of the **space** it belongs to (see `doc/accounts_and_sharing.md`).

**Status: planning.** No code yet. The repo currently contains only `README.md` and the decision docs
in `doc/`.

## Documentation (`doc/`)

`doc/` holds Markdown files that record decisions. Keep them up to date when a decision is made or
changes. Each decision is marked **Decided**, **Proposed** or **Open**.

- `doc/tech_stack.md`: backend, frontend, storage, i18n and testing choices, with the alternatives that
  were considered.
- `doc/accounts_and_sharing.md`: authentication and the sharing model (personal and shared spaces,
  invite links, membership check).
- `doc/sync.md`: refetching data when a page is entered, and optimistic locking with per-item/per-list
  version numbers (409 on conflict).
- `doc/design.md`: domain model, screens, UX principles, image storage and REST API sketch (draft).
- `doc/ideas.md`: loose feature ideas (item/list details, reordering, images, …). These are **not
  requirements yet**; don't implement them until they've been refined into `design.md`.

Don't hard-code anything marked **Open**, and don't silently turn **Proposed** into **Decided**. Ask
first, then update the doc.

## Decided so far

- **Backend:** Quarkus + Kotlin, Gradle Kotlin DSL, JVM 21. Base package `com.tfassbender.bucketlist`.
  Ports: backend 4721, Vite 5175 (siblings use 4711 to 4720 and 5174).
- **Frontend:** React + TypeScript + Vite + React Router, built as a static SPA served by Quarkus. Other
  frontend libraries (data fetching, drag-and-drop, i18n, …) are only candidates; decide them when the
  feature needs them.
- **Styling:** plain CSS, no Tailwind. Use CSS Modules (`Component.module.css`) and the design tokens in
  `src/styles/tokens.css` (CSS custom properties, with a dark theme). No raw hex colors or pixel spacing
  in component styles. Mobile-first: base styles for the phone, `min-width` media queries for desktop.
- **Storage:** JSON files (`users/<id>.json`, `spaces/<id>.json`) plus plain image files, **no database**.
- **Auth:** email + password, BCrypt, HS256 JWT, taken over from `../mr_right_puzzle`.
- **Sharing:** shared spaces; every space endpoint checks membership and returns 404 to non-members.
- **No live updates:** data is refetched when a page is entered or the tab becomes visible; lost updates
  are prevented with version numbers (`doc/sync.md`).
- **i18n:** translation files, **German is the default**, English second. Never hard-code user-facing
  text; every new key goes into both `de.json` and `en.json`.
- **Platform:** works on desktop, **optimized for mobile** (~360 to 390 px wide, touch).
- **Repo layout:** one Gradle project with the modules `backend` / `frontend` / `e2e`.
- **Backend structure:** packages by feature (`auth`, `space`, `image`, plus `storage` and `rest`),
  loosely hexagonal. Each feature's `model` subpackage is pure Kotlin (no Quarkus, Jakarta, Jackson or
  file I/O); only `rest` knows HTTP. A small ArchUnit `ArchitectureTest` enforces this (rules in
  `doc/tech_stack.md`). Don't add interfaces/ports unless there's a second implementation.
- **Testing:** JUnit 5 + RestAssured + ArchUnit (backend), Vitest (frontend), Playwright Java (e2e).
- **Not decided yet:** the scope of the first version, hosting, deployment details. Don't start
  implementing features before the scope is agreed.

## Sibling reference projects (read-only)

Next to this repo in `D:\intelliJ_workspace\` are related projects. Never modify them from here.

- **`../mr_right_puzzle`** is the closest reference: Quarkus + Kotlin, packages by feature, ArchUnit,
  mobile-first, de/en, and Preact (whose API matches React). Look at:
  - user management: `backend/src/main/kotlin/com/tfassbender/mrright/auth/`, `rest/AuthResource.kt`, `rest/AuthDtos.kt`;
  - storage: `storage/AtomicJsonFile.kt`, `storage/JsonMapper.kt`;
  - SPA routing / error mapping: `rest/SpaRouting.kt`, `rest/RestExceptionMapper.kt`;
  - optimistic locking with `version` + 409: `doc/architecture.md`, section "Saving & Sync";
  - Gradle/Kotlin/Quarkus build setup (the Kotlin plugin version must match the Quarkus BOM).
- **`../avalons_final_stand`**: the original of the same auth code (hexagonal layout).
- **`../todo_organizer_2`**: reload on tab return, as described in `doc/sync.md`. See commit `e64b9af`
  (`visibilitychange`/`pageshow` reload, flushing pending saves when the tab is hidden, not overwriting
  unsaved edits).
- `../capture_royale`, `../avalon`: older siblings (Java backend, React frontend).

## Build, run & test commands

Not set up yet. Once the project is scaffolded, document the commands here (expected:
`./gradlew :backend:quarkusDev`, `cd frontend && npm run dev`, `./gradlew :backend:test`, `npm test`).

## Working conventions

- Solo project; the main branch is `main`.
- The backend sends language-neutral data: error **codes**, never texts. The frontend translates them.
- Models, services and repositories are tested as plain JUnit 5 (no `@QuarkusTest`); use `@QuarkusTest` +
  RestAssured only for the REST layer.
- E2E tests create their own disposable users (email-disabled dev/test mode), run in desktop and mobile
  viewports, and never depend on debug endpoints in production code.
