# Bucket List

A small web app for keeping bucket lists and working on them together, for example as a couple.
Lists live in **spaces**: everyone has a private personal space and can invite others into a shared
space, where all members see and edit the same lists. It works on desktop and is optimized for mobile.

> **Status:** planning. There is no code yet; the decisions are documented in [`doc/`](doc/).

## Tech stack

- **Backend:** Quarkus + Kotlin, Gradle (Kotlin DSL)
- **Frontend:** React + TypeScript + Vite, React Router, plain CSS
- **Storage:** JSON files and image files, no database
- **Auth:** email + password, JWT
- **i18n:** German (default) and English
- **Tests:** JUnit 5, RestAssured, ArchUnit, Vitest, Playwright

## Documentation

- [doc/tech_stack.md](doc/tech_stack.md): tech stack, repo layout and backend structure
- [doc/accounts_and_sharing.md](doc/accounts_and_sharing.md): accounts and sharing
- [doc/sync.md](doc/sync.md): refreshing data and avoiding lost updates
- [doc/design.md](doc/design.md): domain model, screens, API sketch
- [doc/ideas.md](doc/ideas.md): feature ideas
- [mockup/](mockup/): HTML/CSS mockups
