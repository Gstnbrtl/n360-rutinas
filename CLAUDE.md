# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**N360** is a single-file personal training management app (`index.html`). It is a standalone HTML file — no build system, no package manager, no server. Open directly in a browser.

`clientes.html` is a standalone prototype (Supabase-auth version) that was used as a reference when building the Clientes view into the main app. It is no longer the primary file.

## Architecture

Everything lives in `index.html` (~3900 lines): all CSS, HTML, and JavaScript in one file.

### Views (single-page routing)
Four views toggled via `showView(id)` using `.app-view` / `.app-view.on`:
- `view-editor` — routine builder (the default/landing view)
- `view-clientes` — client management with stats, search, filter, table
- `view-ejercicios` — exercise database management
- `view-circuito` — circuit builder

### Data layer — dual storage model
All data flows through **two persistence layers**:

| Layer | What it holds | Key(s) |
|---|---|---|
| **Supabase** | clients, routines, custom_exercises, db_overrides, saved_circuits | tables with `user_id` column |
| **localStorage** | local cache + extra client data | `n360_clients`, `n360_clientes_extra`, `n360_custom_ex`, `n360_db_muscles`, `n360_db_warmups`, `n360_cir_ex`, `n360_saved_circuits` |

On login → `loadAllFromCloud()` fetches everything from Supabase and populates the in-memory objects + localStorage. Writes go to localStorage immediately via `saveToStorage()`, then sync to Supabase via the `saveToCloud()` family of functions.

`n360_clientes_extra` is localStorage-only — it holds phone, email, payment status, due date, and amount for each client (keyed by client UUID). It is never synced to Supabase.

### In-memory state (global variables)
```
clients       — {[cid]: {name, objetivo, notas, routines[]}}
extraData     — {[cid]: {telefono, email, estado, vencimiento, monto}}
customExercises — {[muscle]: [...]}
DB            — built-in exercise database
CIR_DB        — circuit exercise database
savedCircuits — []
currentUser   — Supabase auth user object (null before login)
```

### Auth flow
A login overlay (`#loginOverlay`) is shown on load. On successful Supabase auth, `onLogin(user)` is called, which: hides overlay → calls `loadAllFromCloud()` → calls `loadExtra()` → renders all views.

### Supabase credentials
```
URL: https://zhdkyuytpudvyhjuqqre.supabase.co
Key: sb_publishable_DjDpU1FvokBk3CIUVtENVQ_R4m8UEls  (publishable/anon key)
```

### Design system
CSS variables defined in `:root`. Key tokens:
- Colors: `--bg:#09090F`, `--surface:#13131C`, `--red:#FF3B30`, `--green:#30D158`, `--blue:#0A84FF`, `--amrap:#FF9F0A`, `--tabata:#BF5AF2`
- Fonts: `--font-display` (Bebas Neue), `--font-cond` (Barlow Condensed), `--font-body` (Barlow)
- Body uses a radial gradient background; nav buttons use per-view accent colors (editor=red, clientes=green, ejercicios=blue, circuito=purple).
- Day tabs are rounded pills (`border-radius:20px`) with a glow shadow on the active tab.
- All new UI components must use these variables — no hardcoded colors.

## Editing the file

The file path (`C:\Users\Gastón Bertola\Desktop\N360\index.html`) contains a non-ASCII character (`ó`) in the directory name. This causes failures with:
- PowerShell `[System.IO.File]::ReadAllText` / `WriteAllText`
- The Edit tool when the `old_string` contains emoji or special Unicode characters

**Reliable approaches:**
- **Edit tool**: works for changes whose anchor strings are ASCII-safe (no emoji, no `═`, no accented chars in the matched region).
- **Python script** at a path without special chars (e.g. `C:/Windows/Temp/fix.py`), using `glob.glob(r'C:\Users\Gast*\Desktop\N360\index.html')` to locate the file, then `open(fpath, encoding='utf-8')`.
- **PowerShell** `Get-Content`/`Set-Content` with the wildcard: `(Get-ChildItem 'C:\Users\Gast*\Desktop\N360\index.html').FullName`.

## Key functions to know

| Function | Purpose |
|---|---|
| `showView(id)` | Switch active view |
| `renderClients()` | Rebuild the clients table; reads `clients` + `extraData` |
| `loadExtra()` / `saveExtra()` | Read/write `n360_clientes_extra` localStorage key |
| `getEstadoInfo(estado, vencimiento)` | Returns `{key, cls, label}` for payment badge; auto-detects overdue by date |
| `saveToStorage()` | Persist `clients` to localStorage |
| `saveToCloud()` | Full sync of clients+routines to Supabase |
| `loadAllFromCloud()` | Fetch all user data from Supabase on login |
| `onLogin(user)` | Entry point after auth; calls load functions and initial renders |
| `showToast(msg)` | Show a temporary notification |
| `uid()` | Generate a UUID for new records |
| `addToWarmupDB(eId,sId,rId,dId)` | Save a warmup exercise from the editor to `DB.warmups`, then calls `saveDBOverridesToCloud()` + `rebuildWuDatalist()` |
| `rebuildWuDatalist()` | Refresh the `<datalist id="dl_wu">` from `DB.warmups` |
| `addToExDB(bid, e)` | Save an exercise from a block row to `customExercises[muscle]`, then calls `saveCustomExercisesToCloud()` + `fillEx(bid)` |
| `addToCirDB(sid)` | Save a circuit exercise text input value to `cirCustomEx[cat]`, then calls `saveCirCustomEx()` + `renderCirStations()` |
| `cleanupRoutines()` | Keep only the most-recent routine per client; deletes older ones from Supabase in batches of 50. Button auto-removes after success. |
| `importRoutine(data)` | Receives a routine JSON object (same shape as `collect()`), finds/creates the client, replaces their routines with this one, loads it in the editor, saves to storage + cloud. |

### AI-generated routine workflow
When the user provides client data (name, goal, level, available days, and any physical limitations), Claude designs a routine using exercises from the DB. Equipment is NOT asked separately — exercises are picked from the existing DB entries and can be slightly adjusted in name if needed (e.g. "con barra liviana") but must derive from real DB entries and calls `importRoutine(data)` via the browser console. The data format mirrors `collect()` output:
```json
{
  "alumno": "Nombre",
  "objetivo": "Hipertrofia / Fuerza / ...",
  "days": [{
    "title": "Día 1 – Tren Inferior",
    "warmup": [{"e": "Thrusters con banda", "s": "2", "r": "10", "d": "60 seg"}],
    "blocks": [{
      "type": "normal",
      "muscle": "Cuádriceps",
      "cfg": {},
      "exs": [{"e": "Sentadilla con barra", "type": "normal", "s": "4", "r": "10", "d": "90", "i": ""}]
    }]
  }]
}
```
`type` per block: `"normal"` | `"amrap"` (cfg: `time`, `rest`) | `"tabata"` (cfg: `work`, `rest`, `rounds`, `brk`).  
Each block supports up to 4 exercises (`exs` array). Fields: `e`=exercise, `s`=series, `r`=reps, `d`=rest in seconds, `i`=notes.

### Exercise input pattern
Warmup rows and circuit stations use `<input type="text" list="dl_*">` + datalist (autocomplete) + a `<button class="btn-savedb">+</button>` to save custom entries to the respective database. The same `+` button pattern is used on every exercise row in Normal/AMRAP/Tabata blocks.
