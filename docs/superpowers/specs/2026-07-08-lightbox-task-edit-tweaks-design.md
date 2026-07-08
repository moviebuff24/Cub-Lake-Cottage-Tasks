# Mood-Board Lightbox, Task Editing, and Timeline Tweaks — Design

**Date:** 2026-07-08
**Status:** Approved by Joe ("Make all changes live")

Four small changes, batched:

## 1. Mood-board photo lightbox (`components/vision-boards.tsx`)

Vision-board photo thumbnails become clickable: hover shows the same darkened
"View photo" overlay the property tiles use; click opens a fullscreen lightbox
matching the property-photo one (black backdrop, photo up to 75vh, photo name
as label, X to close, backdrop/Escape close). Below the photo: a **Delete**
button only (no Replace — boards hold photo lists, not slots), reusing the
existing `handleRemovePhoto` (RTDB remove + Storage cleanup). The small
hover ✕ on thumbnails keeps working (stopPropagation so it doesn't open the
lightbox). Lightbox state is local to each `BoardTile` — no new props.

## 2. Task edit modal (`app/page.tsx`)

A pencil icon joins the notes/delete hover icons on each task card. It opens
a modal in the Add Task style, pre-filled with **task name, due date, and
month**. Saving updates the task in state (existing Firebase sync persists
it); changing month moves the task to that month group automatically (grouping
derives from `task.month`). Category is intentionally excluded. Empty due date
saves as `'TBD'` (same as Add Task). Save disabled when the name is empty.

## 3. Collapse fully-completed months by default (`app/page.tsx`)

`getDefaultMonthOpen` currently keeps a fully-completed month expanded if it
is the current month. Remove that exception: any month whose tasks are all
complete starts collapsed on page load. No live auto-collapse when the last
task is checked mid-session (avoids yanking the list); takes effect next load.

## 4. First Stay → September 1 (`app/page.tsx`)

`FIRST_STAY` changes from `2026-08-01` to `2026-09-01`; the hero card label
changes from "Aug 1" to "Sep 1". Days-to-go recalculates automatically.

## Verification

`npm run build` → "Compiled successfully" (local prerender Firebase error is
expected). Push to `main`, GitHub Actions deploys. Live checks: "Sep 1" in
hero, "View photo" overlay markup in vision boards, pencil edit works,
completed months collapsed on load.
