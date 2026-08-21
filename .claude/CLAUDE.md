# Cub Lake Cottage

## What this is
Next.js task-tracker / photo-sharing site for Joe's Cub Lake cottage (10723 Black Bear Rd NE, Kalkaska MI) — the STR property he and his wife bought in 2026. Used to coordinate pre-launch tasks, share photos, and hold "vision board" mood boards for the property.

For the property's business/tax context (Augusta rule vs. STR launch, financial model) see `../Idea Generator/references/cub-lake-tax-strategy.md` and `../Idea Generator/references/whitehall-str-analysis.md`.

## Live site
- **URL:** https://moviebuff24.github.io/cub-lake-cottage
- **Repo:** `github.com/moviebuff24/cub-lake-cottage`
- **Deploy:** GitHub Actions on push to `main` — this is a **static export** (`output: 'export'` in `next.config.mjs`), NOT Vercel, despite a leftover `@vercel/analytics` import. API routes do not work in production (they're silently dropped from static export with no build error) — don't build features that depend on server-side routes.

## Critical gotchas (learned the hard way)
- **`npm run build` will always fail locally** at the prerender step with a Firebase "Can't determine Database URL" error — there's no `.env.local` committed (by design). This is expected, not a regression. "Compiled successfully" in the build output is the real signal that code is valid; GitHub Actions injects the real secrets.
- **Always commit `package.json` AND `package-lock.json`** together when installing a new npm package — CI fails otherwise.
- **Firebase RTDB writes throw synchronously** on any `undefined` nested value — sanitize objects with `JSON.parse(JSON.stringify(obj))` before writing, and wrap in try/catch (a `.catch()` alone won't catch a sync throw).
- **The live task list lives in Firebase RTDB, not in code.** `page.tsx` does `setTasks(data ?? initialTasks)` — once Firebase has a `tasks` node (it does), editing `lib/tasks.ts` has no effect on the live site. To edit live tasks without clobbering Joe's UI changes: GET `/tasks.json`, modify, PUT back (read-modify-write, no clobber).
- **Firebase config is recoverable** from the public GitHub Pages JS bundle if `.env.local` is ever missing locally — it's `NEXT_PUBLIC_*` and baked into the client bundle at build time.
- **Git push sometimes hangs on a stale credential prompt.** Workaround: `git push "https://${GITHUB_PERSONAL_ACCESS_TOKEN}@github.com/moviebuff24/cub-lake-cottage.git" main`
- Link previews (Vision Boards) use `r.jina.ai` (free reader API) for title/description and `s0.wp.com/mshots` for thumbnails — both work client-side with no server, chosen after microlink.com went down and a server API route turned out to be dead in production (see gotcha above).

## Conventions
- Follow the workspace UI style guide (`../.claude/style-guide.md`) for any new UI
- This site has no auth and open RTDB rules — it's a private-use tool for Joe + his wife, not public-facing security-hardened software
