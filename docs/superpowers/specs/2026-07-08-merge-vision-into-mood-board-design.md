# Merge Vision into Mood Board — Design

**Date:** 2026-07-08
**Status:** Approved by Joe

## Goal

Replace the Mood Board tile grid with the VisionBoards component and remove the
standalone Vision section from the Cub Lake Cottage site. One place for
inspiration content instead of two.

## Current state

`app/page.tsx` renders two separate inspiration features:

- **Mood Board** — a subsection inside The Property section: a drag-sortable
  tile grid (Hot Tub Dreams, Cabin Decor, Fire Pit, Dock Vibes, plus custom
  tiles), each tile pinning one photo with an optional caption. Backed by
  Firebase at `photos/inspirationPhotos`, `photos/customInspirationSlots`,
  `photos/inspirationOrder`, and `inspirationCaptions`.
- **Vision** — a standalone section (`id="vision"`) at the bottom of the page:
  a badge, "This is just the beginning" heading, an intro paragraph, and the
  self-contained `<VisionBoards />` component (named boards with photo grids,
  link previews, and notes).

## Decisions (confirmed with Joe)

1. **Replace the tile grid entirely.** The Mood Board subsection becomes just
   `<VisionBoards />`. Old tile photos are no longer displayed; their data
   stays in Firebase untouched.
2. **Nav link renamed**, not removed: "Vision" → "Mood Board" in both the
   desktop nav and the mobile menu, scrolling to the merged subsection.
3. **Drop the Vision intro text.** No "This is just the beginning" heading or
   paragraph. The Mood Board keeps its current compact heading style (small
   uppercase label + sparkle icon), with the helper line updated to describe
   boards instead of tiles.

## Changes

All in `app/page.tsx`:

1. **Mood Board subsection** (inside The Property section):
   - Keep the compact heading row ("Mood Board" + Sparkles icon).
   - Update the helper line under it to describe the boards.
   - Replace the DndContext tile grid + "Add Category" button with
     `<VisionBoards />` (self-contained, no props).
   - Give the subsection `id="vision"` so nav scroll targets it.
2. **Delete the standalone Vision section** — the whole `<section id="vision">`
   block: badge, heading, paragraph, background blobs, and its `VisionBoards`
   usage (the component now renders in the Mood Board spot instead).
3. **Nav**: rename "Vision" to "Mood Board" in the desktop nav and the mobile
   menu array. Scroll target stays `vision`.
4. **Dead code cleanup** — remove everything that only served the tile grid:
   - State: `inspirationPhotos`, `customInspirationSlots`, `inspirationOrder`,
     `inspirationCaptions`, `captionWriteTimerRef`, `inspirationOrderWriteRef`,
     `inspirationInputRef`.
   - Data: the `inspirationBoard` constant.
   - Handlers: `handleInspirationDragEnd`, `updateInspirationCaption`, the
     `'inspiration'` branches of `handleFileUpload`, `triggerUpload`,
     `removePhoto`, `removeCustomSlot`, and the lightbox.
   - Derived arrays: `allInspirationTiles`, `fullInspirationOrder`,
     `orderedInspirationTiles`.
   - The hidden inspiration file input and the `inspirationCaptions` Firebase
     subscription.
   - Narrow the `'property' | 'inspiration'` union types to just property (or
     drop the type parameter where it becomes pointless).

## Data safety (critical)

The page currently writes the **entire** `photos` node to Firebase RTDB:

```ts
set(dbRef(db, 'photos'), { propertyPhotos, inspirationPhotos, ... })
```

If inspiration fields are simply dropped from state, the next write would
**erase the old mood-board photos from Firebase**. To honor "data stays in
Firebase, just not displayed":

- Change the photo-metadata write effect to target subpaths instead of
  replacing the whole node: `photos/propertyPhotos`,
  `photos/customPropertySlots`, `photos/propertyOrder`.
- `photos/propertyOrder` already has a subpath write in the drag handler —
  unchanged.
- Never write to `photos/inspirationPhotos`, `photos/customInspirationSlots`,
  `photos/inspirationOrder`, or `inspirationCaptions` again; they remain
  orphaned but intact.
- The Firebase subscription can keep reading the whole `photos` node and just
  ignore the inspiration keys.

## Error handling

No new failure modes: `VisionBoards` already handles its own Firebase errors,
and the subpath writes keep the existing `.catch(console.error)` pattern.

## Verification

1. `npm run build` — "Compiled successfully" is the pass signal. The prerender
   Firebase "Can't determine Database URL" error at the end is expected locally
   (no `.env.local`) and not a regression.
2. Manual check on the deployed site: Mood Board shows the vision boards inside
   The Property section, old Vision section gone, both nav menus say
   "Mood Board" and scroll correctly, property photo upload/reorder still
   syncs, and Firebase still contains the old inspiration data.
3. Deploy: push to `main`; GitHub Actions builds and publishes to GitHub Pages.
   If the credential helper hangs, push with
   `git push "https://${GITHUB_TOKEN}@github.com/moviebuff24/cub-lake-cottage.git" main`.

## Out of scope

- Migrating old tile photos into a vision board (Joe chose full replacement).
- Deleting the orphaned Firebase data.
- Any changes to `components/vision-boards.tsx`.
