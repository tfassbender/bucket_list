# Accounts, authentication and sharing

Goal: several people can use the app, and two (or more) accounts can work **together** on bucket lists,
but nobody can see anything they weren't explicitly given access to. Detailed permissions come later.

## Authentication (Decided: same approach as `../mr_right_puzzle` / `../avalons_final_stand`)

Both sibling projects implement the same user management. **`../mr_right_puzzle` is the primary
reference** because it uses the same simple feature-package structure we want here:
`backend/src/main/kotlin/com/tfassbender/mrright/auth/` and `rest/AuthResource.kt`, `rest/AuthDtos.kt`.
`../avalons_final_stand` (`application/auth/`, `adapter/security/JwtTokenIssuer.kt`) is the original,
with a hexagonal layout. Take the code over and adapt it (package names, config prefix `bucketlist.`).

- **Login with email + password.** The email is normalised to lowercase; the display name is unique
  regardless of case.
- Passwords are hashed with **BCrypt** (`at.favre.lib:bcrypt`, cost 10). Validate the 72-byte limit before
  hashing.
- Sessions are **HS256 JWTs** issued with `quarkus-smallrye-jwt-build` and verified by `quarkus-smallrye-jwt`.
  The user id is the `upn`/`sub` claim and the `user` group goes with `@RolesAllowed("user")`. A dev JWK is
  committed to the repo; production overrides the key location. Tokens are valid for **30 days**, as in
  `mr_right_puzzle`, so you don't have to log in on your phone again every day.
- The account stores the chosen UI **language** (default `de`).
- The SPA keeps the token in `localStorage` and sends it as `Authorization: Bearer …`.
- Email flows: email verification, resend verification, forgot/reset password and change password.
  Raw tokens are only ever emailed; only their **SHA-256 hashes** and expiry times are stored.
  "Resend" and "forgot" always respond silently, so nobody can probe which emails are registered.
- **Dev/test mode without real email** (hard requirement): `bucketlist.auth.email-disabled=true` in the
  `%dev`, `%test` and `%e2e` profiles. In that mode registration verifies automatically and returns a
  token, and the mailer is mocked. Production keeps real verification.
- Endpoints under `/api/auth`: `register`, `login`, `me`, `verify-email`, `resend-verification`,
  `forgot-password`, `reset-password`, `change-password`, `config`.

## Sharing model

### Options considered

| Option | How it works | Verdict |
|---|---|---|
| A. One shared account | Both people log in with the same credentials. | No: no record of who changed what, a shared password, nothing can be private. |
| B. Connected accounts | Two accounts are "linked"; each sees everything the other created. | Simple, but all-or-nothing: no private lists, and it gets unclear with a third person (does A see C's lists through B?). |
| C. Share each list | Every list has its own access list of users. | Flexible, but you have to share every new list by hand, which is tedious for a couple. |
| **D. Shared spaces** | Lists belong to a **space**. Every member of a space sees and edits all of its lists. | **Recommended.** |

### Decided: shared spaces (option D)

- On registration, every user automatically gets a **personal space** that only they can see.
- A user can create a **shared space** (for example "Tobias & partner") and **invite** someone.
  Everything inside that space is visible to both, which gives the "connected accounts" feeling of
  option B without its drawbacks:
  - private lists stay possible (they live in the personal space),
  - visibility is explicit and never transitive,
  - it also works for other groups later (family, friends' trip).
- **Invitations by code or link**: the owner generates a single-use invite token that expires (only its
  hash is stored, like the auth tokens) and sends it over any messenger. The invitee opens the link
  while logged in and joins. No email sending is needed, so it works in dev mode too. Inviting by email
  can come later.
- Every list and item records **`createdBy` / `updatedBy` / timestamps**, so each person can see who
  added or changed what. A small "recent activity" view per space is a possible follow-up.
- Working on the same space at the same time is handled by refetching and version numbers, not live
  updates. See [sync.md](sync.md).
- Lists can be **moved** between spaces (for example, start a list privately and move it to the shared
  space).

### Authorization rule (first version)

- Every space-scoped endpoint checks **"is the caller a member of this space?"**. If not, it returns
  **404** rather than 403, so the existence of a space isn't revealed.
- For now all members are equal (they can read, create, edit and delete). The data model still stores
  a role on each membership (`OWNER` / `MEMBER`) so that finer permissions (for example only the owner
  can delete the space or remove members, or a read-only `VIEWER` role) can be added later without a
  migration.

### Open questions

- Can a member leave a shared space, and what happens to the space if the owner leaves?
- Should deleting items need confirmation, or have a trash/undo instead?
- Is there a cap on how many members a space can have?
