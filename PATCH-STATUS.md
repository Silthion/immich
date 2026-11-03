# PATCH-STATUS: people for shared-album viewers

NEEDS REVIEW

Status: STOPPED after step 2 (design premise contradicted by the code)

Base: `69e0cb8606137c4f8dc0e7207a672ff8be39f358` (v3.3.4). Branch: `album-viewer-people`.

## Checklist

- [x] 1. Status file created and pushed
- [x] 2. Read relevant code; line references checked (see findings)
- [ ] 3. Repository: find missing viewer rows / remove stale unnamed rows
- [ ] 4. Service/job wiring, name fallback, triggers
- [ ] 5. Unit tests
- [ ] 6. Medium tests (invariants 1–6)
- [ ] 7. pnpm install, type check, lint, unit tests, SQL snapshots
- [ ] 8. Final status

## Log

- Step 1: branch created from v3.3.4 commit, status file pushed.

## Step 2 findings (why I stopped)

Line references are approximately right (`withSharing` ~L103, `withPerson` L169, `getAllForUser` L393, `inSharedAlbum` utils/database.ts L291, asset.repository ~L794/~L885, `handleRecognizeFaces` ~L542-633, `upsertPeopleUsers` person.service.ts ~L836).

**The premise "viewer rows are only created by facial recognition" is wrong for v3.3.4.**
`server/src/schema/functions.ts` (`person_user_after_insert`, trigger on `person_user`, migration `1790616293884-PersonSharing.ts`) already inserts a `person` row `(ownerId=sharedWith, personGroupId)` for every newly inserted share. It copies the sharer's `name`/`birthDate`, and on conflict fills in an empty name/birthDate. Both `upsertPeopleUsers` paths (`createAll`, `createAllForOwner`) go through it.

Consequences for the plan:
- The viewer row, name fallback and `otherPeople` already exist after a share. The planned sync job, `mapPerson` name fallback and "named" filter fallback would duplicate this, and the trigger's copied name makes the "empty name" case rare.
- `getAllForUser` already counts only own/partner faces plus faces in albums shared with the viewer. Invariant 2 therefore already holds for the People list, and the existing album-scoped paths cover the rest.
- Thumbnails already fall back to the sharer's thumbnail (`getForThumbnail` `sharedThumbnailPath`), so no thumbnail job is needed for display.

Real remaining gaps I can see by reading (not reproduced; no DB or Docker run yet):
1. Viewer rows get `faceAssetId = null` and `thumbnailPath = ''` (only the sharer's thumbnail is used as fallback).
2. People created for the owner after a "share all" call are not shared with the viewer.
3. Rows are created at share time regardless of album membership. Visibility is still album-scoped by the queries, so there is no privacy leak.
4. If a viewer reports seeing nothing, the likely causes are config rather than missing rows: no `person_user` share, a share whose viewer is not in the same cluster group, the `minimumFaces` filter with an unnamed person, or the viewer's `person` row being deleted (which cascades to delete the share via `person_delete_shares`).

## Proposed alternative

Do not add the sync job. First reproduce with the owner/viewer setup on a real instance or a medium test: share a person, add the album, and call `GET /people` and `GET /people/:id` as the viewer. If the viewer still sees nothing, find which of the cases above applies. A minimal patch might then be only: (a) set `faceAssetId` and queue thumbnail generation for trigger-created rows, and (b) share newly created owner people automatically when a "share all" share exists. Please confirm which direction you want before I implement anything.

No code changes were made.
