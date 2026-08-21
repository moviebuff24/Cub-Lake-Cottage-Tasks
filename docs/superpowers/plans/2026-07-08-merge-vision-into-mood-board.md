# Merge Vision into Mood Board Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the Mood Board tile grid in `app/page.tsx` with the `<VisionBoards />` component and delete the standalone Vision section, without touching the retired tile data in Firebase.

**Architecture:** Single-file change to `app/page.tsx`. The `VisionBoards` component (`components/vision-boards.tsx`) is self-contained (manages its own Firebase state, takes no props) and simply moves render location. All inspiration-tile state, handlers, and render code are removed, and the Firebase photo-metadata write switches from replacing the whole `photos` node to writing three property subpaths so the orphaned inspiration data is never clobbered.

**Tech Stack:** Next.js 16 (static export to GitHub Pages), React, Firebase RTDB + Storage, Tailwind, dnd-kit, lucide-react.

**Spec:** `docs/superpowers/specs/2026-07-08-merge-vision-into-mood-board-design.md`

## Global Constraints

- Site deploys via GitHub Pages **static export** (`output: 'export'` in `next.config.mjs`). No server routes.
- Local `npm run build` **always fails at prerender** with Firebase "Can't determine Database URL" (no `.env.local` locally). **"Compiled successfully" is the pass signal** — the prerender error is expected and not a regression.
- **Never write the whole `photos` node to Firebase.** Only subpaths: `photos/propertyPhotos`, `photos/customPropertySlots`, `photos/propertyOrder`. The keys `photos/inspirationPhotos`, `photos/customInspirationSlots`, `photos/inspirationOrder`, and `inspirationCaptions` must remain untouched in the database.
- The merged Mood Board keeps the element id `vision` so both nav menus scroll to it.
- No test suite exists in this repo. The verification cycle is: `npm run build` (compile check) + manual check on the deployed site (Task 2).
- If `git push` hangs or rejects auth, use: `git push "https://${GITHUB_TOKEN}@github.com/moviebuff24/cub-lake-cottage.git" main`
- Do not modify `components/vision-boards.tsx`.

---

### Task 1: Rewire page.tsx — VisionBoards into Mood Board, remove Vision section and tile-grid code

All edits are in `app/page.tsx`. They are one atomic change (the removed state is referenced throughout the file), so the file only compiles again once all steps in this task are done. Line numbers below refer to the file **before** any edits.

**Files:**
- Modify: `app/page.tsx`

**Interfaces:**
- Consumes: `VisionBoards` from `@/components/vision-boards` (already imported at line 40; no props).
- Produces: n/a (leaf change; no other file imports from `page.tsx`).

- [ ] **Step 1: Remove the `Flame` icon import** (only used by the tile grid). In the lucide-react import block (~line 14), delete the line:

```tsx
  Flame,
```

Leave every other icon import alone (including `Upload`, which was already unused before this change — not ours to clean here).

- [ ] **Step 2: Delete the `inspirationBoard` constant** (~lines 64–69):

```tsx
const inspirationBoard = [
  { id: 'hottub', label: 'Hot Tub Dreams', icon: Sparkles, color: 'sunset' },
  { id: 'decor', label: 'Cabin Decor', icon: Home, color: 'pine' },
  { id: 'firepit', label: 'Fire Pit', icon: Flame, color: 'sunset' },
  { id: 'dock', label: 'Dock Vibes', icon: Ship, color: 'lake' },
]
```

- [ ] **Step 3: Remove inspiration state and simplify shared state types** (inside `CubLakeCottage`, ~lines 100–123).

Delete these declarations entirely:

```tsx
  const [inspirationPhotos, setInspirationPhotos] = useState<Record<string, PhotoUpload | null>>({ hottub: null, decor: null, firepit: null, dock: null })
```
```tsx
  const [customInspirationSlots, setCustomInspirationSlots] = useState<Array<{ id: string; label: string }>>([])
```
```tsx
  const [inspirationOrder, setInspirationOrder] = useState<string[]>(['hottub', 'decor', 'firepit', 'dock'])
```
```tsx
  const [inspirationCaptions, setInspirationCaptions] = useState<Record<string, string>>({})
  const captionWriteTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null)
```
```tsx
  const inspirationOrderWriteRef = useRef<ReturnType<typeof setTimeout> | null>(null)
```
```tsx
  const inspirationInputRef = useRef<HTMLInputElement>(null)
```

Replace these declarations (types lose the property/inspiration split):

```tsx
  const [addSlotModal, setAddSlotModal] = useState<{ type: 'property' | 'inspiration' } | null>(null)
```
becomes
```tsx
  const [addSlotModal, setAddSlotModal] = useState(false)
```

```tsx
  const [activeUploadTarget, setActiveUploadTarget] = useState<{ type: 'property' | 'inspiration'; id?: string } | null>(null)
  const [lightboxPhoto, setLightboxPhoto] = useState<{ type: 'property' | 'inspiration'; id: string; label: string; isCustom: boolean } | null>(null)
```
becomes
```tsx
  const [activeUploadTarget, setActiveUploadTarget] = useState<string | null>(null)
  const [lightboxPhoto, setLightboxPhoto] = useState<{ id: string; label: string; isCustom: boolean } | null>(null)
```

- [ ] **Step 4: Trim the Firebase photos subscription** (~lines 276–292). The subscription keeps reading the whole `photos` node but now ignores the inspiration keys:

```tsx
  // Subscribe to Firebase photos metadata — inspiration* keys under photos/
  // are retired mood-board tile data and are intentionally ignored
  useEffect(() => {
    const photosRef = dbRef(db, 'photos')
    const unsubscribe = onValue(photosRef, (snapshot) => {
      const data = snapshot.val()
      isFirebasePhotoUpdate.current = true
      if (data) {
        setPropertyPhotos(data.propertyPhotos || { front: null, lake: null, dock: null, living: null, kitchen: null })
        setCustomPropertySlots(data.customPropertySlots ? Object.values(data.customPropertySlots) as Array<{ id: string; label: string }> : [])
        if (data.propertyOrder?.length) setPropertyOrder(data.propertyOrder)
      }
      setPhotosLoaded(true)
    })
    return () => unsubscribe()
  }, [])
```

- [ ] **Step 5: Delete the `inspirationCaptions` subscription** (~lines 294–301):

```tsx
  // Subscribe to inspiration tile captions — separate path so main photo writes don't clobber them
  useEffect(() => {
    const captionsRef = dbRef(db, 'inspirationCaptions')
    const unsubscribe = onValue(captionsRef, (snapshot) => {
      setInspirationCaptions(snapshot.val() || {})
    })
    return () => unsubscribe()
  }, [])
```

- [ ] **Step 6: Switch the photo-metadata write to subpaths** (~lines 303–311). **This is the data-safety-critical edit.** Replace:

```tsx
  // Write photo metadata to Firebase only when changed locally
  useEffect(() => {
    if (!photosLoaded) return
    if (isFirebasePhotoUpdate.current) {
      isFirebasePhotoUpdate.current = false
      return
    }
    set(dbRef(db, 'photos'), { propertyPhotos, inspirationPhotos, customPropertySlots, customInspirationSlots, propertyOrder, inspirationOrder }).catch(err => console.error('Failed to sync photos:', err))
  }, [propertyPhotos, inspirationPhotos, customPropertySlots, customInspirationSlots, propertyOrder, inspirationOrder, photosLoaded])
```

with:

```tsx
  // Write photo metadata to Firebase only when changed locally.
  // Targets subpaths (never the whole photos node) so the retired mood-board
  // tile data under photos/inspiration* stays intact in the database.
  useEffect(() => {
    if (!photosLoaded) return
    if (isFirebasePhotoUpdate.current) {
      isFirebasePhotoUpdate.current = false
      return
    }
    set(dbRef(db, 'photos/propertyPhotos'), propertyPhotos).catch(err => console.error('Failed to sync photos:', err))
    set(dbRef(db, 'photos/customPropertySlots'), customPropertySlots).catch(err => console.error('Failed to sync photos:', err))
    set(dbRef(db, 'photos/propertyOrder'), propertyOrder).catch(err => console.error('Failed to sync photos:', err))
  }, [propertyPhotos, customPropertySlots, propertyOrder, photosLoaded])
```

- [ ] **Step 7: Delete `updateInspirationCaption`** (~lines 376–385):

```tsx
  const updateInspirationCaption = (id: string, caption: string) => {
    setInspirationCaptions(prev => {
      const next = { ...prev, [id]: caption }
      if (captionWriteTimerRef.current) clearTimeout(captionWriteTimerRef.current)
      captionWriteTimerRef.current = setTimeout(() => {
        set(dbRef(db, 'inspirationCaptions'), next).catch(err => console.error('Failed to sync captions:', err))
      }, 800)
      return next
    })
  }
```

- [ ] **Step 8: Simplify the upload/remove/slot handlers to property-only** (~lines 403–491). Replace each function as follows.

In `handleFileUpload`, replace the target bookkeeping — the lines

```tsx
    const target = activeUploadTarget
```
and
```tsx
      if (target.type === 'property' && target.id) {
        setPropertyPhotos(prev => ({ ...prev, [target.id!]: upload }))
      } else if (target.type === 'inspiration' && target.id) {
        setInspirationPhotos(prev => ({ ...prev, [target.id!]: upload }))
      }
```
become
```tsx
    const targetId = activeUploadTarget
```
and
```tsx
      setPropertyPhotos(prev => ({ ...prev, [targetId]: upload }))
```
(everything else in `handleFileUpload` — size check, storage upload, error handling — stays exactly as is).

Replace `triggerUpload`:

```tsx
  const triggerUpload = (id: string) => {
    setActiveUploadTarget(id)
    propertyInputRef.current?.click()
  }
```

Replace `removePhoto`:

```tsx
  const removePhoto = (id: string) => {
    const photo = propertyPhotos[id]
    setPropertyPhotos(prev => ({ ...prev, [id]: null }))

    if (photo?.storagePath) {
      deleteObject(storageRef(storage, photo.storagePath)).catch(err =>
        console.error('Failed to delete from storage:', err)
      )
    }
  }
```

Replace `removeCustomSlot`:

```tsx
  const removeCustomSlot = (id: string) => {
    removePhoto(id)
    setCustomPropertySlots(prev => prev.filter(s => s.id !== id))
    setPropertyOrder(prev => prev.filter(oid => oid !== id))
  }
```

Replace `confirmAddSlot`:

```tsx
  const confirmAddSlot = () => {
    if (!newSlotLabel.trim() || !addSlotModal) return
    const id = `custom-${Date.now()}`
    setCustomPropertySlots(prev => [...prev, { id, label: newSlotLabel.trim() }])
    setPropertyOrder(prev => [...prev, id])
    setNewSlotLabel('')
    setAddSlotModal(false)
  }
```

- [ ] **Step 9: Delete `handleInspirationDragEnd`** (~lines 515–533):

```tsx
  const handleInspirationDragEnd = (event: DragEndEvent) => {
    const { active, over } = event
    if (!over || active.id === over.id) return
    const knownIds = new Set(inspirationOrder)
    const allIds = [
      ...inspirationOrder,
      ...inspirationBoard.map(c => c.id).filter(id => !knownIds.has(id)),
      ...customInspirationSlots.map(s => s.id).filter(id => !knownIds.has(id)),
    ]
    const from = allIds.indexOf(String(active.id))
    const to = allIds.indexOf(String(over.id))
    if (from === -1 || to === -1) return
    const newOrder = arrayMove(allIds, from, to)
    setInspirationOrder(newOrder)
    if (inspirationOrderWriteRef.current) clearTimeout(inspirationOrderWriteRef.current)
    inspirationOrderWriteRef.current = setTimeout(() => {
      set(dbRef(db, 'photos/inspirationOrder'), newOrder)
    }, 400)
  }
```

- [ ] **Step 10: Remove inspiration derived arrays** (~lines 535–550). Replace the whole "Ordered tile arrays for rendering" block with:

```tsx
  // Ordered tile arrays for rendering
  const allPropertyTiles = [
    ...photoCategories.map(c => ({ ...c, isCustom: false as const })),
    ...customPropertySlots.map(s => ({ id: s.id, label: s.label, icon: ImageIcon, hasImage: false, isCustom: true as const })),
  ]
  // Ensure any tile whose ID isn't in the saved order yet still shows up
  const knownPropertyIds = new Set(propertyOrder)
  const fullPropertyOrder = [...propertyOrder, ...allPropertyTiles.map(t => t.id).filter(id => !knownPropertyIds.has(id))]
  const orderedPropertyTiles = fullPropertyOrder.map(id => allPropertyTiles.find(t => t.id === id)).filter((t): t is NonNullable<typeof t> => t != null)
```

- [ ] **Step 11: Remove the hidden inspiration file input** (~line 562). Delete only the second input:

```tsx
      <input type="file" ref={inspirationInputRef} className="hidden" accept="image/*" onChange={handleFileUpload} />
```

- [ ] **Step 12: Rename the nav links.** In the mobile menu array (~line 594):

```tsx
                { label: 'Vision', id: 'vision' },
```
becomes
```tsx
                { label: 'Mood Board', id: 'vision' },
```

In the desktop nav (~line 672):

```tsx
              <button onClick={() => scrollToSection('vision')} className="opacity-70 hover:opacity-100 transition-all hover:tracking-wide">Vision</button>
```
becomes
```tsx
              <button onClick={() => scrollToSection('vision')} className="opacity-70 hover:opacity-100 transition-all hover:tracking-wide">Mood Board</button>
```

- [ ] **Step 13: Update the property tile grid callers** (~lines 792–863) to the new handler signatures. Four small edits inside the property grid JSX:

```tsx
onClick={() => photo ? setLightboxPhoto({ type: 'property', id: cat.id, label: cat.label, isCustom }) : triggerUpload('property', cat.id)}
```
becomes
```tsx
onClick={() => photo ? setLightboxPhoto({ id: cat.id, label: cat.label, isCustom }) : triggerUpload(cat.id)}
```

```tsx
onClick={(e) => { e.stopPropagation(); isCustom ? removeCustomSlot('property', cat.id) : removePhoto('property', cat.id) }}
```
becomes
```tsx
onClick={(e) => { e.stopPropagation(); isCustom ? removeCustomSlot(cat.id) : removePhoto(cat.id) }}
```

```tsx
onClick={(e) => { e.stopPropagation(); removeCustomSlot('property', cat.id) }}
```
becomes
```tsx
onClick={(e) => { e.stopPropagation(); removeCustomSlot(cat.id) }}
```

```tsx
onClick={() => { setAddSlotModal({ type: 'property' }); setNewSlotLabel('') }}
```
becomes
```tsx
onClick={() => { setAddSlotModal(true); setNewSlotLabel('') }}
```

- [ ] **Step 14: Replace the Mood Board tile grid with VisionBoards** (~lines 869–969). Replace the entire block from `{/* Inspiration board — Mood Board */}` down to (and including) its closing `</div>` after `</DndContext>` with:

```tsx
          {/* Mood Board — vision boards: photos, links, and notes per idea */}
          <div id="vision">
            <div className="mb-6">
              <h3 className="text-sm font-semibold text-muted-foreground uppercase tracking-widest flex items-center gap-3">
                <span className="w-8 h-px bg-border" />
                Mood Board
                <Sparkles className="w-4 h-4" style={{ color: '#d4a574' }} />
                <span className="flex-1 h-px bg-border" />
              </h3>
              <p className="text-sm text-muted-foreground text-center mt-2">
                Collect ideas in boards — pin photos, links, and notes for each project
              </p>
            </div>
            <VisionBoards />
          </div>
```

- [ ] **Step 15: Delete the standalone Vision section** (~lines 1112–1134). Remove the entire block:

```tsx
      {/* Vision Section */}
      <section id="vision" className="px-6 py-20 md:px-12 lg:px-20 relative overflow-hidden">
        <div className="absolute top-0 right-0 w-1/2 h-full opacity-50 pointer-events-none">
          <div className="absolute top-20 right-20 w-64 h-64 rounded-full blur-3xl" style={{ backgroundColor: 'rgba(70, 130, 180, 0.15)' }} />
          <div className="absolute bottom-40 right-40 w-48 h-48 rounded-full blur-3xl" style={{ backgroundColor: 'rgba(212, 165, 116, 0.15)' }} />
        </div>
        <div className="max-w-6xl mx-auto relative">
          <div className="inline-flex items-center gap-2 px-4 py-2 rounded-full bg-secondary text-sm font-medium mb-6">
            <Sparkles className="w-4 h-4" style={{ color: '#d4a574' }} />
            <span>The Vision</span>
          </div>
          <h2 className="font-serif text-4xl md:text-5xl font-medium mb-4 text-balance leading-tight">
            This is just the <span className="italic">beginning</span>
          </h2>
          <p className="text-muted-foreground text-lg leading-relaxed mb-10">
            Every great adventure starts with a single step. Our cottage on Cub Lake
            represents more than just a property — it&apos;s where memories will be made,
            where mornings start with lake views, and where life slows down just enough
            to really be enjoyed.
          </p>
          <VisionBoards />
        </div>
      </section>
```

(The `VisionBoards` import at the top of the file stays — it's now used in the Mood Board block.)

- [ ] **Step 16: Simplify the Add Photo/Inspo Slot modal to property-only copy** (~lines 1264–1313). Three edits inside the `{addSlotModal && (` block:

`onClick={() => setAddSlotModal(null)}` → `onClick={() => setAddSlotModal(false)}` (appears twice: backdrop and Cancel button), and `onKeyDown={e => e.key === 'Escape' && setAddSlotModal(null)}` → `onKeyDown={e => e.key === 'Escape' && setAddSlotModal(false)}`.

```tsx
            <h3 id="add-slot-title" className="font-serif text-xl font-medium mb-2">
              {addSlotModal.type === 'property' ? 'Add a photo spot' : 'Add inspiration'}
            </h3>
            <p className="text-sm text-muted-foreground mb-5">
              {addSlotModal.type === 'property'
                ? 'Name the room or area — you can upload a photo after.'
                : 'Name this inspiration category — you can upload a photo after.'}
            </p>
```
becomes
```tsx
            <h3 id="add-slot-title" className="font-serif text-xl font-medium mb-2">
              Add a photo spot
            </h3>
            <p className="text-sm text-muted-foreground mb-5">
              Name the room or area — you can upload a photo after.
            </p>
```

```tsx
              placeholder={addSlotModal.type === 'property' ? 'e.g. Back Deck, Garage, Basement…' : 'e.g. Kayak Storage, Landscaping…'}
```
becomes
```tsx
              placeholder="e.g. Back Deck, Garage, Basement…"
```

Also update the comment `{/* Add Photo/Inspo Slot Modal */}` → `{/* Add Photo Slot Modal */}`.

- [ ] **Step 17: Update the lightbox to property-only** (~lines 1316–1365). Three edits inside the `{lightboxPhoto && (` block:

```tsx
              src={(lightboxPhoto.type === 'property' ? propertyPhotos : inspirationPhotos)[lightboxPhoto.id]?.url}
```
becomes
```tsx
              src={propertyPhotos[lightboxPhoto.id]?.url}
```

```tsx
                onClick={() => {
                  const { type, id } = lightboxPhoto
                  setLightboxPhoto(null)
                  triggerUpload(type, id)
                }}
```
becomes
```tsx
                onClick={() => {
                  const { id } = lightboxPhoto
                  setLightboxPhoto(null)
                  triggerUpload(id)
                }}
```

```tsx
                onClick={() => {
                  const { type, id, isCustom } = lightboxPhoto
                  isCustom ? removeCustomSlot(type, id) : removePhoto(type, id)
                  setLightboxPhoto(null)
                }}
```
becomes
```tsx
                onClick={() => {
                  const { id, isCustom } = lightboxPhoto
                  isCustom ? removeCustomSlot(id) : removePhoto(id)
                  setLightboxPhoto(null)
                }}
```

- [ ] **Step 18: Sanity-check for leftovers.** Search `app/page.tsx` for the strings `inspiration`, `Inspo`, and `Flame` (case-insensitive). Expected: exactly **two matches**, both in comments — the "inspiration* keys … intentionally ignored" comment from Step 4 and the "photos/inspiration* stays intact" comment from Step 6. Any match in actual code (an identifier, JSX, or string literal) is a missed edit from the steps above — fix it before building.

- [ ] **Step 19: Build to verify the change compiles.**

Run (from `cottage-website-design/`): `npm run build`

Expected: output contains **"Compiled successfully"**, followed by a prerender failure with a Firebase "Can't determine Database URL" error. Per Global Constraints, that prerender error is expected locally and is NOT a failure. Any TypeScript/compile error before "Compiled successfully" IS a failure — fix it.

- [ ] **Step 20: Commit.**

```bash
git add app/page.tsx
git commit -m "Merge Vision boards into Mood Board, remove standalone Vision section"
```

---

### Task 2: Deploy and verify on the live site

**Files:**
- None modified. Push + manual verification only.

**Interfaces:**
- Consumes: the Task 1 commit on `main`.
- Produces: live deployment at moviebuff24.github.io/cub-lake-cottage.

- [ ] **Step 1: Push to main.**

Run (from `cottage-website-design/`): `git push origin main`

If it hangs or rejects with "Invalid username or token", use:
`git push "https://${GITHUB_TOKEN}@github.com/moviebuff24/cub-lake-cottage.git" main`

- [ ] **Step 2: Wait for the GitHub Actions deploy to finish.**

Run: `gh run watch --repo moviebuff24/cub-lake-cottage` (or `gh run list --repo moviebuff24/cub-lake-cottage --limit 1` and poll).
Expected: the `deploy` workflow for the new commit completes with conclusion `success`.

- [ ] **Step 3: Manual verification on the live site** (Joe or agent-with-browser walks through):

1. The Property section shows "Mood Board" heading with the vision boards (named boards with photos/links/notes) directly under it — no tile grid (Hot Tub Dreams / Cabin Decor / Fire Pit / Dock Vibes are gone).
2. The old Vision section at the bottom (badge + "This is just the beginning") is gone; the page goes Notepad → footer.
3. Desktop nav and mobile menu both say "Mood Board" and scroll to the merged section.
4. Property photos: upload to an empty tile, open a filled tile's lightbox, Replace/Delete, and drag-reorder still work and persist after a refresh.
5. Vision boards still load their existing content (they read the same Firebase paths as before — nothing about the component changed).
6. In Firebase console (or by temporarily checking the RTDB REST endpoint), `photos/inspirationPhotos` still contains the old tile data — confirming the subpath-write change protected it. Trigger at least one property-photo change first so a write has actually happened.

- [ ] **Step 4: Report results to Joe** — deployed URL, what was verified, anything that didn't check out.
