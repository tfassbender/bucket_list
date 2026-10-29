# Tech stack

Status of each decision: **Decided** (settled, build on it), **Proposed** (recommendation, waiting for
a final OK) or **Open** (not decided yet, don't hard-code).

## Overview

| Area | Choice | Status |
|---|---|---|
| Backend | Quarkus + Kotlin, Gradle Kotlin DSL, JVM 21 | Decided |
| Frontend | React + TypeScript + Vite (static SPA served by Quarkus), React Router | Decided |
| Styling | Plain CSS: CSS Modules + design tokens as CSS custom properties | Decided |
| Storage | File-based JSON + plain image files, **no database** | Decided |
| Auth | Email + password, BCrypt, HS256 JWT, as in `../mr_right_puzzle` / `../avalons_final_stand` | Decided |
| Sharing | Shared spaces (see [accounts_and_sharing.md](accounts_and_sharing.md)) | Decided |
| Realtime | **No live updates.** Refetch when the page is entered; optimistic locking against lost updates (see [sync.md](sync.md)) | Decided (details Proposed) |
| i18n | Translation files; **German is the default**, English second, more languages possible later | Decided (library Proposed) |
| Platform | Works on desktop, **optimized for mobile** | Decided |
| Repo layout | One Gradle project with the modules `backend` / `frontend` / `e2e`, like the siblings | Decided |
| Backend structure | Packages by feature, loosely hexagonal, guarded by a few ArchUnit tests | Decided |
| Testing | JUnit 5 + RestAssured, ArchUnit, Vitest, Playwright (Java) | Decided |
| Deployment | Single Quarkus JAR that also serves the built SPA | Proposed |
| Hosting / domain | — | Open |
| Scope of the first version | — | Open |

## Backend: Quarkus + Kotlin (Decided)

- Same setup as `../mr_right_puzzle` and `../avalons_final_stand`:
  - `kotlin("jvm")` + `kotlin("plugin.allopen")` (open classes for `@Path`, `@ApplicationScoped`, `@QuarkusTest`),
  - `quarkus-kotlin`, `quarkus-rest-jackson`, `jackson-module-kotlin`, `compilerOptions.javaParameters = true`,
  - **the Kotlin plugin version must match the Kotlin version that the Quarkus BOM enforces** (see the comment
    in `../avalons_final_stand/gradle.properties`, where a version mismatch caused `NoSuchMethodError`s).
- Scaffold with `quarkus create app --kotlin --gradle-kotlin-dsl` or code.quarkus.io; don't write the
  build files by hand.
- Base package: `com.tfassbender.bucketlist`.
- Ports: backend **4721**, Vite dev server **5175**, which proxies `/api` to the backend. Siblings
  already use 4711 to 4720, and `mr_right_puzzle` uses 5174.
- The backend sends **language-neutral data**: error *codes*, not texts. All texts are rendered by the
  frontend, so a backend error code `X` maps to a message key such as `error.X`.

## Repo layout (Decided)

One Gradle project (Kotlin DSL), shaped like `../mr_right_puzzle` and `../avalons_final_stand`:

```
bucket_list/
├── backend/    Quarkus + Kotlin
├── frontend/   React + Vite (npm, wrapped by Gradle tasks); the build output goes into the backend JAR
├── e2e/        Playwright (Java/Kotlin) tests against running dev servers
└── doc/        decision documents
```

## Backend structure (Decided: by feature, loosely hexagonal)

The code is organized **by feature**, with the main idea of hexagonal architecture but without strict
ports and adapters everywhere: the core of each feature (model + rules) knows nothing about HTTP,
Quarkus or the file format. The project is small, so we don't add interfaces just to have them.

```
com.tfassbender.bucketlist
├── auth/            accounts, login, tokens: service + repository
├── space/           spaces, members, invites, lists, items: service + repository
├── image/           image upload/storage (later)
│   └── …/model/     each feature's domain model: plain Kotlin data classes and rules
├── storage/         shared file infrastructure (AtomicJsonFile, JSON mapper)
└── rest/            REST resources, DTOs, exception mapper (the only place that knows HTTP)
```

- **Model** (`<feature>.model`): pure Kotlin; no Quarkus, Jakarta, Jackson or file I/O. Domain rules
  (for example "move item X after item Y", version increments) live here and are unit-tested directly.
- **Service** (`<feature>`): the use cases; checks permissions (space membership), takes the lock and
  calls the repository. It is the boundary the REST layer talks to.
- **Repository** (`<feature>`): a plain class reading and writing that feature's JSON files through
  `storage`. It can be unit-tested against a temp directory. Add an interface only if a second
  implementation actually appears.
- **`rest`**: translates HTTP to service calls and maps model objects to DTOs. DTOs never leak into the
  feature packages.

### ArchUnit rules

A small `ArchitectureTest` (plain JUnit 5, no `@QuarkusTest`) keeps this structure clean:

1. No dependency cycles between the top-level packages.
2. `<feature>.model` doesn't depend on Quarkus, Jakarta, Jackson, Vert.x or `java.nio.file`.
3. Feature packages and `storage` don't depend on `rest`.
4. Only `rest` uses JAX-RS (`jakarta.ws.rs..`).
5. `storage` doesn't depend on any feature package (it's plain infrastructure).

Keep the set small; add a rule only when a real violation shows that it's needed.

## Frontend: React (Decided)

### Why React and not Svelte?

React is a good fit. Size doesn't decide it:
- React 19 + ReactDOM add roughly **60 kB gzipped** of runtime once. Svelte compiles to almost none,
  and Preact is about 4 kB. For an app with a handful of screens that's a small one-time download, and
  the service worker caches it after the first visit.
- The real differences are boilerplate and ecosystem. Svelte needs a bit less code for forms; React has
  the larger ecosystem, which is exactly what this app needs: mature, touch-friendly drag-and-drop for
  reordering, data fetching with refetch on focus, and i18n.
- `../avalon` and `../capture_royale` use React. `../mr_right_puzzle` uses **Preact**, which has the
  React API at 4 kB, so its code is still a useful reference.

There's no strong argument for Svelte here, so **React** it is.

### Libraries

Decided:

| Concern | Choice |
|---|---|
| Build | Vite + TypeScript |
| Routing | React Router |

**Candidates, to be decided later** when the feature that needs them is built:

| Concern | Candidate | Notes |
|---|---|---|
| Server state | TanStack Query | Caching, `refetchOnWindowFocus`/reconnect built in, mutations with rollback. This would cover most of [sync.md](sync.md). The alternative is plain `fetch` plus a small hook. |
| Reordering | dnd-kit (`@dnd-kit/sortable`) | Works with touch and keyboard. |
| i18n | react-i18next | JSON translation files `locales/de.json` (reference) and `locales/en.json`, the same family as `../avalon`. Alternative: Paraglide JS (type-safe, used in `../mr_right_puzzle`, but its compile step loads a plugin from a CDN). |
| Icons | Font Awesome Free | No emoji as icons. |
| PWA | vite-plugin-pwa | Optional; makes the app installable on the home screen. |
| Tests | Vitest + Testing Library | |

### Styling (Decided: plain CSS)

| Option | Pros | Cons |
|---|---|---|
| **Plain CSS: CSS Modules + custom properties** | No extra dependency (Vite supports `*.module.css` out of the box); styles are scoped per component; light/dark theme is just a second set of custom properties; JSX stays readable; `../mr_right_puzzle` does the same. | You name classes yourself; consistency in spacing and colors relies on discipline (the tokens help). |
| Tailwind CSS | Fast to write, consistent spacing/colors from the default scale, good responsive utilities. | Another tool and its class vocabulary; long `className` strings in JSX; for a handful of screens its main advantage (consistency across a large team or codebase) barely matters. |

Decision: **plain CSS**, because the app has only a few screens:
- `src/styles/tokens.css` defines colors, spacing, radii and font sizes as CSS custom properties, with a
  dark-theme override under `@media (prefers-color-scheme: dark)`;
- `src/styles/global.css` holds the reset, base element styles and the layout shell;
- one `Component.module.css` per component, using only the tokens (no raw hex colors or pixel spacing);
- mobile-first: base styles target the phone, `@media (min-width: …)` adds desktop enhancements.

If it starts to feel repetitive, adding Tailwind later is a cheap change.

### Internationalization

- **German is the default**, English is available. The language is chosen in the user menu, saved to the
  account and falls back to the browser language.
- **No hard-coded user-facing strings**; every text goes through a translation key. Keep the effort
  small for now (de + en only), but keep the structure ready for more languages: one JSON file per
  locale, plus a test that checks the keys stay in sync.

## Storage: JSON files + image files (Decided)

No database. The data directory is configured with `bucketlist.data.dir`:

```
data/
├── users/<userId>.json         one file per account
├── spaces/<spaceId>.json       one file per space: members, invites, lists, items
└── images/<spaceId>/<imageId>.<ext>   uploaded images (+ a thumbnail), metadata in the space JSON
```

- All JSON writes go through an `AtomicJsonFile` helper (write a temp file, then move it into place),
  as in the sibling projects.
- A per-space lock serializes concurrent writes to the same space file.
- **Images** are plain files; the space JSON holds only metadata (id, item/list reference, size, who
  uploaded it). Deleting an item or space also deletes its image files. Backups remain "copy the `data/`
  directory". Upload and serving details are in [design.md](design.md#images).

## Realtime: none (Decided)

No SSE or WebSockets. Fresh data comes from refetching when a page is opened or the tab becomes visible
again, and lost updates are prevented with version numbers. See [sync.md](sync.md).

## Testing (Decided)

- Backend: JUnit 5. Models, services and repositories are plain unit tests; REST endpoints use
  `@QuarkusTest` + RestAssured. ArchUnit for the architecture rules above.
- Frontend: Vitest + Testing Library.
- E2E: Playwright (Java) in an `e2e` Gradle module, run in a desktop and a mobile viewport. Tests register
  their own disposable users through the email-disabled dev mode.
