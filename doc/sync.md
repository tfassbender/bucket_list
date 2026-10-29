# Keeping data fresh & not overwriting each other's changes

Two people (or one person on two devices) work on the same space. There are **no live updates**, so we
need two things:

1. **Pull**: show the latest server data when someone enters or returns to a page.
2. **Don't overwrite**: a save based on stale data must not silently replace a newer change.

References: `../todo_organizer_2` (commit `e64b9af`, "reload the current file when the tab becomes
visible") and `../mr_right_puzzle/doc/architecture.md` (section "Saving & Sync"). Status: **Proposed**.

## 1. Pulling changes

The same triggers as in `todo_organizer_2`, `file_handling.js`:

- **On navigation**: opening a page (home, a list, an item) fetches its data from the server.
- **When the tab becomes visible again** (`visibilitychange` → `visible`), for example after switching
  apps or unlocking the phone: refetch the data of the current page.
- **When a page is restored from the back/forward cache** (`pageshow` with `event.persisted`), which
  doesn't always fire `visibilitychange`.
- **When the device comes back online** (`online` event).

If we pick TanStack Query (a candidate, see [tech_stack.md](tech_stack.md#libraries)), it already does
most of this (`refetchOnWindowFocus`, `refetchOnReconnect`, refetch on mount), and we only add the
`pageshow` case. Without it, a small hook listens to these events itself.

**Never overwrite what the user is typing.** As in `todo_organizer_2` (`editor.isClean(generation)`), a
refetch must not replace an edit form that has unsaved input. Forms keep their own draft state and only
take server data when they're not dirty. If the item changed on the server meanwhile, the save that
follows runs into the version check below.

**Before the tab is hidden**, pending debounced saves are sent at once (`visibilitychange` → `hidden`,
`fetch` with `keepalive`), because mobile browsers may freeze a hidden tab before the timer fires. This
is the same `debouncedSave.flush()` idea as in `todo_organizer_2`.

## 2. Optimistic locking with version numbers

Use a **server-side version number** rather than comparing timestamps. Client and server clocks can
differ, and two saves within the same millisecond would look identical. This is the same approach as
`mr_right_puzzle`, but finer-grained.

- Every **list** and every **item** carries a `version: Long` that the server increments on each change.
  Lists and items also keep `updatedAt` / `updatedBy`, but only for display ("changed by X, 5 min ago").
- An update sends the version it was based on: `PATCH …/items/{id}` with `{ baseVersion: 7, title: … }`.
  - If the server version is still 7, the server applies the change and responds **200** with the new
    item at version 8.
  - If the server version is newer, it responds **409 `VERSION_CONFLICT`** with the current server item
    in the body.
- Because versions are kept **per item and per list** instead of per space, two people editing *different*
  items never conflict. A conflict only happens when both edited the same item.

### Handling a conflict in the client

- If the user changed only fields that the server version didn't change, merge automatically and
  re-send with the new base version.
- Otherwise show a small dialog with both versions and who made the other change: "Keep mine"
  (re-send with the server's version as the base) or "Take theirs".
- If the item was deleted in the meantime (**404**), tell the user and offer to re-create it.

### Operations that don't need a version check

Some changes have one obvious result, so they are sent as **intent**, not as a full object:

- **Check off / un-check**: `PATCH { done: true }`. It's the same outcome whoever is first, so it always
  applies.
- **Delete**: deleting something already deleted is fine.
- **Reorder**: send "move item X after item Y" (or to a position), never the whole order array.
  The server applies the move to its current order, so a concurrent add or reorder by the other person
  isn't lost. A move of a deleted item returns 404, and the client refetches.
- **Add item**: always applies, and the new item gets version 1.

## Open questions

- Should a "this list changed meanwhile" banner show up after a refetch brings in changes from the
  other person?
- Should there be an offline mode with a local queue of changes (IndexedDB, as in `mr_right_puzzle`),
  or is "requires a connection, retry on reconnect" enough for the first version?
