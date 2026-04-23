# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**N360** is a single-file personal training management app (`index.html`). Standalone HTML — no build system, no package manager, no server. Open directly in a browser. Deployed automatically to Vercel on every push to `main`.

`clientes.html` is an old prototype, no longer maintained.

## Architecture

Everything lives in `index.html` (~520 KB, ~5000+ lines): all CSS, HTML, and JavaScript in one file.

### Views (single-page routing)
Toggled via `showView(id)` using `.app-view` / `.app-view.on`. When switching to a view, `showView()` calls the matching render function:

| View id | Purpose | Render call |
|---|---|---|
| `view-editor` | Routine builder — default landing view | `renderDays()` if `dpanels` is empty |
| `view-clientes` | Client management | `renderClients()` |
| `view-ejercicios` | Exercise database | `renderExercises()` |
| `view-circuito` | Circuit builder | `renderSavedCircuits()` |
| `view-portal` | Client-facing portal (clients only) | via `loadClientPortal()` |
| `view-blog` | Blog admin | — |

### Auth flow
`initAuth()` is called at script load. It checks for an existing Supabase session and calls `onLogin(user)` if found. `doLogin()` handles the login form submission. Both call `onLogin()`.

`onLogin(user)`:
1. Hides login overlay
2. Detects role (`user_metadata.role === 'client'` → `loadClientPortal()`, else trainer flow)
3. Trainer flow: calls `renderDays()` immediately (so editor appears before cloud data), then `await loadAllFromCloud()`, then `loadExtra()`, `renderClients()`, `renderDays()` again, `setTimeout(patchAddButtons, 100)`

`init()` function exists in the file but is **never called** — it is dead code. `initAuth()` is the real entry point.

### Data layer — dual storage model

| Layer | What it holds | Key(s) |
|---|---|---|
| **Supabase** | clients, routines, custom_exercises, db_overrides, saved_circuits | tables with `user_id` column |
| **localStorage** | local cache + extra client data | `n360_clients`, `n360_clientes_extra`, `n360_custom_ex`, `n360_db_muscles`, `n360_db_warmups`, `n360_cir_ex`, `n360_saved_circuits`, `n360_ex_videos` |

On login → `loadAllFromCloud()` fetches everything from Supabase and populates in-memory objects + localStorage. Writes go to localStorage via `saveToStorage()`, then sync to Supabase via the `saveToCloud()` family.

`n360_clientes_extra` is localStorage-only (phone, email, payment status, due date, amount per client UUID — never synced to Supabase).

### In-memory state (global variables)
```
clients         — {[cid]: {name, objetivo, notas, routines[]}}
extraData       — {[cid]: {telefono, email, estado, vencimiento, monto}}
customExercises — {[muscle]: [...]}
activeRoutines  — {[cid]: routineId}  — which routine each client sees in the portal
exVideos        — {[exerciseName]: videoUrl}  (localStorage-only)
DB              — built-in exercise + warmup database
CIR_DB          — circuit exercise database
savedCircuits   — []
cirStations     — [{id, ex, cat, elementos, nota, blockLabel?}]
cirBlockDefs    — [{muscle, count}]  (not persisted)
currentUser     — Supabase auth user object (null before login)
blogPosts       — []  (stored in db_overrides.blog_posts)
```

### Active routine logic
`activeRoutines[cid]` stores which routine ID the client sees in their portal. It is persisted inside `db_overrides.active_routines` (no schema change needed — existing JSONB column).

- **On load**: `loadAllFromCloud()` auto-sets `activeRoutines[cid]` for any client that doesn't have one (picks last in array, which is newest since routines are loaded ordered by `id asc`). Saves to cloud if any were auto-set.
- **On new save/import**: `saveAsNew()` and `importRoutine()` set `activeRoutines[cid] = routine.id` automatically.
- **Manual override**: Admin clicks "★ Activar" on any routine in `openClientModal()` → calls `setActiveRoutine(cid, routineId)`.
- **Portal**: `loadClientPortal()` reads `active_routines` from the trainer's `db_overrides` and loads that specific routine; falls back to latest if not set.

### db_overrides table
One row per trainer (`user_id`). JSONB columns hold: `muscles`, `warmups`, `cir_db`, `cir_custom`, `active_routines`, `ex_videos`, `ex_descriptions`, `blog_posts`. `saveDBOverridesToCloud()` saves everything except `ex_videos` (saved separately via `saveExVideosToCloud()`).

### Design system
CSS variables in `:root`:
- Colors: `--bg:#09090F`, `--surface:#13131C`, `--red:#FF3B30`, `--green:#30D158`, `--blue:#0A84FF`, `--amrap:#FF9F0A`, `--tabata:#BF5AF2`
- Fonts: `--font-display` (Bebas Neue), `--font-cond` (Barlow Condensed), `--font-body` (Barlow)
- Nav accent colors: editor=red, clientes=green, ejercicios=blue, circuito=purple
- All new UI must use these variables — no hardcoded colors.

## Editing the file

The file path (`C:\Users\Gastón Bertola\Desktop\N360\index.html`) contains a non-ASCII character (`ó`). This breaks PowerShell `ReadAllText`/`WriteAllText` and the Edit tool when anchor strings contain emoji or `═` characters.

**Reliable approaches (in order of preference):**
- **Edit tool**: works when anchor strings are ASCII-safe.
- **Python script** saved to `C:/Windows/Temp/fix.py`, using `glob.glob(r'C:\Users\Gast*\Desktop\N360\index.html')` to locate the file. Use `py -3` to invoke Python. Always pipe stdout through `io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')` to handle Unicode output.
- For complex multi-patch edits, write a self-contained Python script to `C:/Windows/Temp/` and run it with `py -3 C:/Windows/Temp/fix.py`.

## Key functions

| Function | Purpose |
|---|---|
| `showView(id)` | Switch active view; calls view-specific render |
| `renderDays()` | Rebuild editor day tabs + panels + warmup rows |
| `renderClients()` | Rebuild client table; reads `clients` + `extraData` |
| `loadExtra()` / `saveExtra()` | Read/write `n360_clientes_extra` localStorage |
| `getEstadoInfo(estado, vencimiento)` | Returns `{key, cls, label}` for payment badge; auto-detects overdue |
| `saveToStorage()` | Persist `clients` to localStorage |
| `saveToCloud()` | Full sync of clients+routines to Supabase |
| `saveDBOverridesToCloud()` | Save DB.muscles, DB.warmups, CIR_DB, cirCustomEx, activeRoutines to db_overrides |
| `loadAllFromCloud()` | Fetch all user data from Supabase on login; auto-sets activeRoutines |
| `onLogin(user)` | Entry point after auth |
| `setActiveRoutine(cid, routineId)` | Mark a routine as active for portal; saves to cloud |
| `openClientModal(cid)` | Show client detail modal with routine list and ★ Activar buttons |
| `showToast(msg)` | Temporary notification |
| `uid()` | Generate UUID |
| `importRoutine(data)` | Load routine JSON into editor + save; auto-sets as active |
| `saveAsNew(data)` | Save current editor state as new routine; auto-sets as active |
| `rebuildWuDatalist()` | Refresh `<datalist id="dl_wu">` from `DB.warmups` |
| `patchAddButtons()` | Re-bind "agregar bloque" buttons + rebuild warmup datalist |
| `addToWarmupDB(eId,sId,rId,dId)` | Save warmup exercise to DB.warmups + cloud |
| `addToExDB(bid, e)` | Save exercise to customExercises + cloud |
| `generateCircuit()` | Render circuit result from `cirStations` |
| `loadClientPortal(user)` | Load client-facing portal; uses activeRoutines to pick routine |
| `cleanupRoutines()` | Keep only latest routine per client; batch-deletes from Supabase |

## Routine JSON workflow

When Gastón asks for a routine, **only** create the JSON file — do not provide console commands or import instructions. He imports it himself via the "Importar Rutina" button in the app.

1. Ask for client name if not provided (required to name the file and set `alumno` field).
2. Read `json/` folder for reference routines of similar clients.
3. Save to `C:\Users\Gastón Bertola\Desktop\N360\json\{nombre_cliente}.json` (snake_case).
4. Confirm: "Guardado en `json/nombre_cliente.json`" — nothing else.

### JSON format
```json
{
  "alumno": "Nombre Apellido",
  "objetivo": "Hipertrofia / Fuerza / Rendimiento deportivo / ...",
  "days": [{
    "title": "Día 1 – Tren Inferior",
    "warmup": [{"e": "Sentadilla aérea sobre banco", "s": "2", "r": "15", "d": "60 seg"}],
    "blocks": [{
      "type": "normal",
      "muscle": "Cuádriceps",
      "cfg": {},
      "exs": [{"e": "Sentadilla con barra", "type": "normal", "s": "4", "r": "10", "d": "90", "i": "nota opcional"}]
    }]
  }]
}
```

Block types:
- `"normal"` — `cfg: {}`
- `"amrap"` — `cfg: {"time": "10 min", "rest": "2 min"}`
- `"tabata"` — `cfg: {"work": "20", "rest": "10", "rounds": "8", "brk": "1 min"}`
- `"varios"` — `cfg: {}`

Max 4 exercises per block. Fields: `e`=name, `s`=series, `r`=reps, `d`=rest (seconds or "seg"), `i`=notes. Exercises must derive from real DB entries (slight name adjustments allowed, e.g. "con barra liviana").

## Supabase
```
URL: https://zhdkyuytpudvyhjuqqre.supabase.co
Key: sb_publishable_DjDpU1FvokBk3CIUVtENVQ_R4m8UEls  (anon/publishable)
```
Tables: `clients`, `routines`, `custom_exercises`, `db_overrides`, `saved_circuits`. All have `user_id` column. Routines are loaded ordered by `id asc` so the last element is always the newest.
