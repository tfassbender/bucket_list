# Product & UI design (initial draft)

Everything here is a **draft** to start from and will be refined as we build. Loose feature ideas
(for example images, item details and reordering) are collected in [ideas.md](ideas.md).

## Core concepts

- **User**: an account (email, display name, password hash). See [accounts_and_sharing.md](accounts_and_sharing.md).
- **Space**: a container that is visible to its members. Each user has one personal space and can
  create or join shared spaces.
- **Bucket list**: a named list inside a space (for example "Travel", "Experiences", "Food to try").
- **Item**: one bucket-list entry.

## Domain model (draft)

```
User        { id, email, displayName, passwordHash, emailVerified, createdAt, ... }

Space       { id, name, personal: Boolean, createdAt,
              members: [ Membership { userId, role: OWNER|MEMBER, joinedAt } ],
              invites: [ Invite { tokenHash, expiresAt, createdBy } ],
              lists:   [ BucketList ] }

BucketList  { id, title, description?, version,
              createdBy, createdAt, updatedBy, updatedAt,
              items: [ Item ]            // array order = display order
              images: [ ImageRef ] }

Item        { id, title, notes?, done: Boolean, doneAt?, doneBy?,
              tags: [String], targetDate?, version,
              createdBy, createdAt, updatedBy, updatedAt,
              images: [ ImageRef ] }

ImageRef    { id, contentType, width, height, uploadedBy, uploadedAt }
```

The whole `Space` is persisted as one JSON file and the image binaries as separate files (see
[tech_stack.md](tech_stack.md#storage-json-files--image-files-decided)). `version` is used for
optimistic locking (see [sync.md](sync.md)). Ids are UUID strings, typed in Kotlin as
`@JvmInline value class`es (`UserId`, `SpaceId`, …).

Ideas for later (images, comments, activity feed, …) are listed in [ideas.md](ideas.md).

## Screens (first version)

1. **Login / register** (plus verify email and reset password).
2. **Home**: the user's spaces (personal first), each with its lists and a progress count (done/total).
3. **List view**: the items in a list, with an open/done toggle, quick-add at the top, reordering
   (drag and drop that also works on touch), a tag filter and show/hide done items. Tapping an item opens its details for editing.
4. **Space settings**: rename the space, see members, create an invite link, leave the space.
5. **Account**: display name, change password, log out.

## UX principles

- **Works on desktop, optimized for mobile.** Most use will be on a phone (adding an idea on the go), so
  layouts are single column and work from about 360px wide, with large tap targets. Desktop gets the same
  layout with more room.
- **German UI by default**, English available; all texts come from translation files.
- **Fast capture.** Adding an item takes one field and Enter; details are optional and can be added later.
- **Clear attribution.** Small "added by …" / "done by …" hints in shared spaces.
- **Light and dark theme**, following the system setting.
- Icons are Font Awesome; no emoji used as UI icons.
- PWA (installable on the home screen) is a likely follow-up and is cheap with a static SPA.

## Images

Still an idea (see [ideas.md](ideas.md)), but the storage is planned so images can be added without
restructuring anything:

- Files live in `data/images/<spaceId>/<imageId>.<ext>`, together with a small thumbnail.
- **The client resizes before uploading** (for example to a 2000 px long edge, as JPEG/WebP), because
  phone photos are 5 to 10 MB. The server still enforces a size limit and an allow-list of image types.
- Images are served through the backend with the same space-membership check as all other data. A plain
  `<img src>` can't send the bearer token, so either the client fetches images as blobs, or the API hands
  out short-lived signed image URLs. To be decided when we implement images.

## REST API sketch

```
/api/auth/...                                    see accounts_and_sharing.md
GET    /api/spaces                               spaces I'm a member of
POST   /api/spaces                               create a shared space
GET    /api/spaces/{spaceId}                     space incl. lists (+ items)
PATCH  /api/spaces/{spaceId}                     rename
POST   /api/spaces/{spaceId}/invites             create invite → returns link
POST   /api/invites/{token}/accept               join a space
DELETE /api/spaces/{spaceId}/members/me          leave
POST   /api/spaces/{spaceId}/lists               create list
PATCH  /api/spaces/{spaceId}/lists/{listId}      edit / move to other space
DELETE /api/spaces/{spaceId}/lists/{listId}
POST   /api/spaces/{spaceId}/lists/{listId}/items
PATCH  /api/spaces/{spaceId}/lists/{listId}/items/{itemId}       { baseVersion, … } → 200 | 409
POST   /api/spaces/{spaceId}/lists/{listId}/items/{itemId}/move  { afterItemId | null }
DELETE /api/spaces/{spaceId}/lists/{listId}/items/{itemId}
```

Updates carry a `baseVersion` and get **409 `VERSION_CONFLICT`** if the data on the server is newer;
check-off, delete and move are sent as intent and always apply. See [sync.md](sync.md).
