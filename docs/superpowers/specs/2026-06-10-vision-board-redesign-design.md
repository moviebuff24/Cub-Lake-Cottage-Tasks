# Vision Board Redesign — Design Spec
**Date:** 2026-06-10
**Status:** Approved

## Overview

Replace the current flat photo grid in the Vision section with a grid of named section containers. Each container holds mixed content (photos, links, notes) and matches the existing Mood Board card aesthetic. Users can create as many sections as they want and add any mix of content types to each.

## Layout

The Vision section keeps its existing prose copy ("This is just the beginning…") at the top, full-width. Below the copy, the right-side photo panel is replaced by a full-width grid of section containers. The grid uses the same responsive columns as the Mood Board (`grid-cols-2 md:grid-cols-3`).

Each section container is a white card with:
- Section name in the header row, "＋ Add" button on the right
- Item grid inside (2 columns, square tiles)
- Consistent with existing card border/radius/shadow style

An "Add Section" dashed card sits at the end of the grid, matching the current "Add Category" button pattern.

## Content Types

Three item types, all rendered as square tiles at the same size:

| Type | Visual | Tap action |
|---|---|---|
| Photo | Image fills tile, remove ✕ on hover | Opens file picker to replace |
| Link | Light blue tile — title + domain | Opens URL in new tab |
| Note | Light yellow tile — note text | Opens inline edit |

## Interactions

**Adding a section:** Click "Add Section" → modal asking for section name → creates empty section.

**Adding an item — type picker flow:**
1. Click "+ Add" on a section → small popup appears with three buttons: 📷 Photo · 🔗 Link · 📝 Note
2. Photo → closes popup, triggers file picker immediately
3. Link → popup stays open, shows URL + title fields inline, Save closes it
4. Note → popup stays open, shows textarea inline, Save closes it

**Removing an item:** ✕ button on hover, same as existing photo tiles. No confirm needed (notes/links are low-stakes; photos match existing behavior).

**Removing a section:** ✕ on the section header, two-step confirm ("Delete section and all its items? Yes / ✗") since this is destructive.

**Section ordering:** Drag-to-reorder using the same dnd-kit + GripVertical handle pattern as existing tiles. Order persisted to Firebase with 400ms debounce.

## Data Model (Firebase RTDB)

New paths under `vision/`, completely separate from the existing `photos/` tree:

```
vision/
  sectionOrder      string[]   ordered list of section IDs
  sections/
    {sectionId}/
      id            string
      label         string
      createdAt     number
  items/
    {itemId}/
      id            string
      sectionId     string
      type          'photo' | 'link' | 'note'
      order         number     position within section
      -- photo fields --
      url           string
      storagePath   string
      name          string
      -- link fields --
      url           string
      title         string
      -- note fields --
      text          string
```

Items are stored flat (not nested under sections) to simplify Firebase subscriptions — one `onValue` listener on `vision/items` loads everything, filtered client-side by `sectionId`.

## Migration

On first load, if `photos/visionPhotos` contains any entries, they are moved into a new section called "Photos" and the old path is left in place (not deleted, to avoid data loss). The migration runs once: guarded by checking whether `vision/sections` already exists.

## Code Structure

`page.tsx` is currently ~1,550 lines. All Vision Board logic is extracted to `components/VisionBoard.tsx`:

- All Vision-related state (`visionSections`, `visionItems`, `visionSectionOrder`)
- Firebase subscriptions for `vision/`
- All Vision rendering and interaction handlers
- `AddItemModal` and `AddSectionModal` sub-components (defined in the same file)

`page.tsx` renders `<VisionBoard />` in place of the current Vision section JSX. No other sections are affected.

## Out of Scope

- Item-level drag-to-reorder within a section (can add later)
- Link preview auto-fetch (title is typed manually — no server-side fetch available on static Pages)
- Section drag-to-reorder on mobile touch (dnd-kit PointerSensor handles desktop; touch can be added later with TouchSensor)
