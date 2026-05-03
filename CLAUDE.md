# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Overview

**SST GameHub v2.0** — A gamified training platform for *Seguridad y Salud en el Trabajo* (SST / SSOMA / Occupational Health & Safety) deployed entirely as a **Google Apps Script (GAS) Web App**. There is no Node.js, no npm, no build step. Everything runs in the browser via GAS HTML Service.

**Domain context:** SST = Seguridad y Salud en el Trabajo (Occupational H&S). SSOMA = SST + Medio Ambiente. OP = Operaciones/Planta. Concepts include: EPP (Personal Protective Equipment), peligro (hazard), riesgo (risk), IPERC (hazard identification matrix), PETS (Safe Work Procedures), ATS (Task Safety Analysis), plan de emergencia, señalización, primeros auxilios, incidentes, accidentes laborales.

---

## Deployment & Runtime

- **Platform:** Google Apps Script — code runs server-side in GAS, HTML/CSS/JS run client-side via `HtmlService`
- **Entry point:** `Code.js` → `doGet(e)` routes to the correct HTML page via `?page=PageName`
- **Routing table in Code.js (`PAGE_FILES`):**
  ```
  Portal     → Portal.html
  Admin      → Admin.html
  Kahoot     → kahoot.html
  Mahjong    → Mahjong.html
  Memoria    → Memoria.html
  DragDrop   → DragDrop.html
  Quiz       → Quiz.html
  Simulacion → Simulación.html
  Pupiletras → Pupiletras.html
  Crucigrama → Crucigrama.html
  ```
- **No local run.** To test: copy files into the GAS editor at script.google.com and deploy as Web App.
- **Shared CSS:** All game pages use `<?!= include('css'); ?>` which injects `css.html` at render time. Portal.html and Admin.html have their own inline `<style>`.
- **Server calls from client:** Always via `google.script.run.withSuccessHandler(fn).withFailureHandler(fn).serverFunction(args)`. Never use `fetch()` or XHR — they can't call GAS functions.

---

## File Map

| File | Purpose |
|------|---------|
| `Code.js` | Entire GAS backend (~2200 lines). Auth, Sheets CRUD, AI calls, data serving, Drive upload |
| `css.html` | Shared CSS design system injected into all game pages |
| `Portal.html` | Dashboard with login (DNI), sidebar nav, XP/badge system, game launcher |
| `Admin.html` | Admin control panel — neon glass dark theme, self-contained CSS, batch Memoria upload |
| `kahoot.html` | Kahoot-style quiz battle with arenas, timer, GIF reactions |
| `Memoria.html` | Memory card flip game — renders Drive image URLs as `<img>` cards, emoji as fallback |
| `Mahjong.html` | Mahjong-style tile matching (concept ↔ definition pairs) |
| `DragDrop.html` | Drag-and-drop classification into categories |
| `Simulación.html` | Hazard inspection simulation — click hotspots on a scene |
| `Quiz.html` | Classic multiple-choice quiz |
| `Pupiletras.html` | 15×15 word search — click/drag selection, 8-direction placement, SST vocabulary |
| `Crucigrama.html` | Crossword puzzle — client-side greedy intersect layout, numbered cells, input navigation |

---

## Google Sheets Data Model

**Spreadsheet ID** is stored in `CONFIG.SPREADSHEET_ID` (default hardcoded, overrideable via Script Properties).

| Sheet tab | Purpose | Key columns |
|-----------|---------|-------------|
| `PERSONAL` | User registry for login | `[0]=DNI, [1]=Nombre, [2]=Apellido` |
| `RESULTADOS` | All game scores | `[0]=Fecha, [1]=DNI, [2]=Nombre, [3]=Juego, [4]=Puntaje, [5]=Tiempo, [6]=Detalles` |
| `CONTENIDO_IA` | AI-generated content saved for reuse | `[0]=Fecha, [1]=Juego, [2]=Fuente, [3]=JSON_content` |
| `Temas` | Kahoot arenas | `[0]=Titulo, [1]=Desc, [2]=Estado(ACTIVO/INACTIVO), [3]=ImagenURL` |
| `Preguntas` | Kahoot questions per arena | `[0]=Titulo, [1]=Pregunta, [2-5]=OpA-D, [6]=Correcta(1-4), [7]=GIF_OK, [8]=GIF_FAIL, [9]=ImagenURL` |
| `Registros` | Kahoot play history | `[0]=ID, [1]=Usuario, [2]=Alias, [3]=Capacitacion, [4]=Puntaje, [5]=Tiempo, [6]=Fecha, [7]=Hora` |
| `Acceso` | Kahoot admin password | `[0]=user, [1]=password` |
| `Quiz_Manual` | Manual quiz questions | pregunta, opA, opB, opC, opD, correcta(1-4), explicacion |
| `Mahjong_Manual` | Manual mahjong pairs | concepto, definicion |
| `Memoria_Manual` | Manual memory pairs | **[0]=imagen_url** (Drive URL or emoji fallback), [1]=concepto, [2]=explicacion |
| `DragDrop_Manual` | Manual drag-drop data | categoria, color_hex, elemento, explicacion |
| `Simulacion_Manual` | Manual simulation hazards | escenario_id, titulo, descripcion_escenario, ambiente, nombre_peligro, desc_peligro, severidad, x, y, ancho, alto, solucion |
| `Pupiletras_Manual` | Manual word search words | palabra (A-Z only, no accents), categoria, descripcion |
| `Crucigrama_Manual` | Manual crossword words | palabra (A-Z only, no accents), pista |

Run `SETUP_CrearHojasManuales()` in GAS to auto-create all manual sheets with example rows.

> **Memoria_Manual column 0** was renamed from `emoji` to `imagen_url`. The game reads it and uses `isImgUrl()` to decide whether to render as `<img>` (Drive URL) or as emoji text. Old emoji data still works.

---

## Google Drive — Images Folder

All images uploaded via the Admin Memoria batch form go to a fixed Drive folder:

```javascript
var DRIVE_IMAGES_FOLDER_ID = '1s7uxYhyIkLoDFNL0HhkfZGtM9k8fxKx2';
// https://drive.google.com/drive/folders/1s7uxYhyIkLoDFNL0HhkfZGtM9k8fxKx2
```

`uploadImageToDrive()` uses `DriveApp.getFolderById(DRIVE_IMAGES_FOLDER_ID)` directly — no name search, no folder creation. The GAS account must have **Editor** access to this folder.

Uploaded files are set to `ANYONE_WITH_LINK VIEW` sharing and the URL returned is:
`https://drive.google.com/uc?export=view&id=FILE_ID`

**To diagnose Drive permission issues:** run `testDriveAccess()` from the GAS editor (Run menu). It creates and deletes a test file and logs the result. If it fails, the GAS account needs Editor access to the folder.

---

## Content Priority Chain (for every game)

`getAIContent(gameType, params)` tries in order:
1. **Manual sheet data** — `leerDatosManual(gameType)` reads the `*_Manual` tab
2. **Saved AI content** — reads `CONTENIDO_IA` sheet, picks random matching row
3. **Live AI call** — `callAI(prompt, fileData)` → Gemini 1.5-flash → fallback OpenAI gpt-4o-mini
4. **Hardcoded fallback** — `simulateAIResponse(gameType)` returns minimal placeholder data

---

## AI Engine

- `callAI(prompt, fileData)` — unified entry point, respects `CONFIG.AI_PROVIDER` ('auto'|'gemini'|'openai')
- `callGeminiAI(prompt, fileData)` — tries `gemini-1.5-flash` then `gemini-2.0-flash`
- `callOpenAI(prompt, fileData)` — tries `gpt-4o-mini` then `gpt-3.5-turbo`
- All AI functions return `{ data: parsedJSON | null, error: string | null }`
- AI config stored in Script Properties: `GEMINI_API_KEY`, `OPENAI_API_KEY`, `AI_PROVIDER`, `SPREADSHEET_ID`
- Large file uploads use chunked upload: `uploadFileChunk()` → `processChunkedFile()`
- `buildFilePrompt(gameType, fileName)` has the exact JSON schema prompt for each game type

**AI JSON schemas by game:**
- `quiz` → `{ questions: [{id, question, options[4], correct(0-3), explanation}] }`
- `mahjong` → `{ pairs: [{id, concept, match}] }`
- `memoria` → `{ pairs: [{id, front(IMAGE_URL_or_emoji), back(CONCEPT_UPPERCASE), explanation}] }`
- `dragdrop` → `{ categories: [{name, color}], items: [{id, text, category, explanation}] }`
- `simulacion` → `{ scenario: {title, description, environment}, hazards: [{id, name, description, severity, x, y, width, height, solution}] }`
- `pupiletras` → `{ palabras: [{palabra(A-Z_NO_ACCENTS), categoria, descripcion}] }`
- `crucigrama` → `{ palabras: [{palabra(A-Z_NO_ACCENTS), pista}] }`

**Word normalization rule (Pupiletras & Crucigrama):** words must be A-Z uppercase only, no tildes, no spaces, 4-15 chars. The helper `norm(w)` in both game pages strips accents (Á→A, É→E, Í→I, Ó→O, Ú→U, Ñ→N) and removes non-A-Z chars.

---

## Design System (css.html — shared by all game pages)

**Color palette (CSS variables):**
```css
--cyan: #3bdcff      /* primary accent */
--blue: #5f8dff      /* secondary */
--violet: #8b7bff    /* tertiary */
--green: #39d98a     /* success */
--amber: #f7b84b     /* warning */
--orange: #ff9348    /* action */
--red: #ff6f7d       /* danger */
--bg-950: #06101b    /* darkest bg */
--bg-900: #091523    /* page bg */
--panel: rgba(12,24,40,.82)   /* glass panels */
```

**Fonts:** `Fredoka` (display/headings) + `Nunito` (body). Loaded from Google Fonts.

**Key CSS classes:**
- `.sst-shell` — base page shell
- `.animated-bg` + `.floating-shapes` — animated particle background
- `.ge-particles` + `.ge-confetti` — in-game particles/confetti canvas
- `.topbar` — game topbar with back button, title, stats, action buttons
- `.game-area` — main game content area
- `.modal-bg` / `.modal` — win/end modal overlay
- `.loading` — loading screen with spinner
- `.tone-success/primary/warning/danger` — stat value color tones
- `.btn-back`, `.btn-new`, `.btn-sm`, `.btn-primary`, `.btn-secondary`

**Admin.html** has its own inline CSS (does not use css.html). Theme: neon glass dark with cyan/purple accents. Key admin classes: `.adm-section`, `.adm-card`, `.adm-card-accent-*`, `.adm-table`, `.adm-form-grid`, `.adm-collapsible`, `.adm-badge`, `.adm-btn`, `.adm-grid2`, `.adm-grid3`.

**Admin Memoria batch CSS** (also inline in Admin.html):
- `.mem-batch-row` — grid: 28px | 120px | 1fr | 1fr | 32px
- `.mem-img-zone` — 110×110px drag/drop image zone, `position:relative; overflow:hidden`
- `.mem-img-zone input[type=file]` — `position:absolute;inset:0;opacity:0` (invisible but clickable)
- `.mem-field` — concepto input + explicacion textarea column
- `.mem-ai-badge` — violet badge shown when field was auto-filled by Gemini Vision

---

## Authentication Flow

1. **Portal login:** user enters DNI → `validateLogin(dni)` checks PERSONAL sheet → returns `{success, user:{dni, nombre, apellido}}`
2. User data stored in `sessionStorage` as `sst_user` JSON (legacy key: `sst_u` — both handled in game pages)
3. Game pages read `sessionStorage.getItem('sst_user')` to get `currentUser`
4. Score saved via `saveScore({dni, nombre, juego, puntaje, tiempo, detalles})` → writes to RESULTADOS sheet
5. **Kahoot** has its own separate login/alias system (`validarUsuarioPorDNI`, `guardarIntento`) that bridges to the same RESULTADOS sheet

---

## Admin.html — Sections & JS State

**Sidebar nav sections:** dashboard, kahoot, memoria, mahjong, dragdrop, simulacion, pupiletras, crucigrama, quiz, imagenes, ia

**`gameSchemas`** — columns array per game type (order = column order in sheet):
```javascript
memoria:    ['imagen_url','concepto','explicacion'],
mahjong:    ['concepto','definicion'],
dragdrop:   ['categoria','color_hex','elemento','explicacion'],
simulacion: ['escenario_id','titulo','descripcion_escenario','ambiente','nombre_peligro','desc_peligro','severidad','x','y','ancho','alto','solucion'],
pupiletras: ['palabra','categoria','descripcion'],
crucigrama: ['palabra','pista']
```

**`gameLabels`** — human-readable column labels for table headers:
```javascript
memoria: { imagen_url:'Imagen', concepto:'Concepto', explicacion:'Explicación' }
```

**`renderGameTable(game, records)`** — builds HTML table. For `imagen_url` columns with `http`/`data:` values renders a 52×52px `<img>` thumbnail instead of raw URL text.

**Memoria batch form state (JS):**
- `let memRows = []` — array of `{base64, mimeType, fileName, preview, concepto, explicacion}`
- `let _memNextIdx = 0` — monotonic counter for stable DOM IDs
- `initMemBatch()` — resets state, calls `addMemRows(10)`
- `addMemRows(n)` — appends n rows; each row: image dropzone (drag+click) | concepto input | explicacion textarea | clear button
- `loadMemImage(file, idx)` — FileReader → base64, updates `memRows[idx]` and DOM preview
- `aiCompleteMemoria()` — collects rows with image+empty fields → calls `analizarImagenesMemoria()` → fills concepto/explicacion + shows 🤖 badge
- `saveBatchMemoria()` — validates (skips empty, blocks if image without concepto) → calls `guardarBatchMemoria(toSave)` → on success: resets form + reloads table

---

## Adding a New Game — Checklist

1. Create `GameName.html` with `<?!= include('css'); ?>` in `<head>`
2. Use body class `sst-shell page-gamename game-epic`
3. Add boilerplate: `.animated-bg`, `.ge-particles`, `.ge-confetti`, `.loading`, `.topbar`, `.modal-bg`
4. In `Code.js`: add to `PAGE_FILES` and to `simulateAIResponse` switch
5. If the game needs manual data: add entry to `MANUAL_SHEETS` in Code.js and update `leerDatosManual()`
6. If AI-generated: add a prompt to `buildFilePrompt()` `prompts` object and add validation in `processUploadedFile()`
7. In `Admin.html`: add a new section (`sec-gamename`) with CRUD form + table, add to sidebar nav and dashboard game cards; add to `gameSchemas` and `gameLabels`; add to `helpContexts`
8. Register game scores via `google.script.run.saveScore({...})` at game end
9. In `Portal.html`: add nav button, add to `GS` object, update badge threshold if needed

---

## Key Backend Functions Reference

| Function | Description |
|----------|-------------|
| `doGet(e)` | Routes `?page=` to HTML file |
| `validateLogin(dni)` | Checks PERSONAL sheet, returns user object |
| `getAIContent(gameType, params)` | Priority chain: manual → saved AI → live AI → fallback |
| `saveScore(d)` | Writes to RESULTADOS sheet |
| `getManualSheetRecords(gameType)` | Reads a `*_Manual` sheet, returns `{success, records[]}` |
| `saveManualSheetRecord(gameType, payload)` | Appends/updates a row in `*_Manual` |
| `deleteManualSheetRecord(gameType, row)` | Deletes row from manual sheet |
| `processUploadedFile(b64, name, mime, game)` | Sends file to AI, saves result to CONTENIDO_IA |
| `callAI(prompt, fileData)` | Unified AI call (Gemini→OpenAI fallback) |
| `saveGeneratedContent(gameType, data, src)` | Saves AI output to CONTENIDO_IA sheet |
| `getLeaderboardGlobal()` | Aggregated XP leaderboard from RESULTADOS |
| `getTopScores(limit)` | Top N scores by puntaje (Admin dashboard widget) |
| `getRecentScores(limit)` | Most recent N scores (Admin activity widget) |
| `adminObtenerTemas()` | Gets Kahoot arenas from Temas sheet |
| `adminGuardarPregunta(q)` | Appends question to Preguntas sheet |
| `saveConfigFromAdmin(...)` | Saves API keys to Script Properties + tests connections |
| `getConfigStatus()` | Returns config state for Admin UI init |
| `crearHojasManuales()` | Creates all `*_Manual` sheets if missing (safe to call repeatedly) |
| `SETUP_CrearHojasManuales()` | GAS menu wrapper that calls `crearHojasManuales()` + shows UI alert |
| `leerDatosManualPupiletras()` | Reads Pupiletras_Manual, normalizes words to A-Z uppercase |
| `leerDatosManualCrucigrama()` | Reads Crucigrama_Manual, normalizes words to A-Z uppercase |
| `uploadImageToDrive(base64, fileName, mimeType)` | Uploads image blob to Drive folder `DRIVE_IMAGES_FOLDER_ID`, sets public sharing, returns `{success, url, fileId}` |
| `analizarImagenesMemoria(imagesData)` | Gemini Vision per image → returns `{success, results:[{concepto, explicacion}], errors[]}` |
| `guardarBatchMemoria(rows)` | Calls `loadConfig_()` + `crearHojasManuales()`, uploads images to Drive, appends rows to Memoria_Manual |

---

## Memoria.html — Image Card Rendering

Cards now support both Drive image URLs and legacy emoji:

```javascript
function isImgUrl(s) { return typeof s==='string' && (s.startsWith('http') || s.startsWith('data:')); }

function cardBackHTML(c) {
  if (c.side==='front' && isImgUrl(c.display)) {
    // Drive image card: <img> + concepto label below
    return `<img class="card-img" src="${c.display}" ...><div class="card-txt">${c.pair.back}</div>`;
  }
  if (c.side==='front') {
    // Legacy emoji card
    return `<span class="card-emoji-lg">${c.display}</span><div class="card-txt">${c.pair.back}</div>`;
  }
  // Back of card: shows concepto text
  return `<div class="card-txt" style="font-size:1rem;font-weight:900">${c.display}</div>`;
}
```

Image cards get CSS class `img-card` with `aspect-ratio: 3/4`.
Error panel on miss: shows `[imagen] → CONCEPTO` when front is a URL.

---

## Portal Gamification System

Defined entirely client-side in `Portal.html`:
- **XP** — earned per game played (varies by game, typically 50–200 XP)
- **Levels** — XP thresholds with titles: Aprendiz → Practicante → Especialista → Experto → Maestro → Leyenda SST
- **Badges** — unlocked by conditions (first game, score > threshold, all games played, etc.)
- Badge `ag` (Especialista SST) requires `p.uniq >= 7` — all 7 game modules played
- **Level-up overlay** (`#LUO`) and badge toast (`#BTO`) animate on unlock
- State persisted in `sessionStorage` (not backend) — resets on new session
- `getLeaderboardGlobal()` called on portal load for the ranking panel

---

## Scoring Convention

| Game | Scoring logic |
|------|--------------|
| Kahoot | 1000 pts per correct + speed bonus, shown per-question |
| Memoria | 100 pts per pair match, -20 per error, +time bonus |
| Mahjong | Points per pair + timer bonus |
| DragDrop | Points per correct drop, accuracy multiplier |
| Simulación | Points per hazard found, bonus for all found |
| Quiz | Points per correct answer |
| Pupiletras | 100 pts per word found + time bonus at completion |
| Crucigrama | 150 pts per word + 10 pts per letter + time bonus |

---

## Known Bugs Fixed — Do Not Repeat

| Bug | Where | Fix applied |
|-----|-------|-------------|
| `openSpreadsheet_()` called but never defined | `guardarBatchMemoria` in Code.js | Replaced with `SpreadsheetApp.openById(CONFIG.SPREADSHEET_ID)` |
| `loadConfig_()` not called before Spreadsheet access | `guardarBatchMemoria` | Added `loadConfig_()` at top of function |
| `renderGameTable` shows Drive URL as raw text for `imagen_url` column | Admin.html | Special-case renders `<img>` thumbnail when value starts with `http`/`data:` |
| `onchange` on concepto input didn't uppercase | Admin.html batch form | Added `.toUpperCase()` to `onchange` handler |
| Drive upload failure saved `'🖼️'` emoji silently to sheet | `guardarBatchMemoria` | Now returns the Drive error immediately — never saves emoji fallback |
| Drive/GAS error shown only as small toast | Admin.html `saveBatchMemoria` | Now renders full error in red `adm-alert-err` panel above the form |

> **Pattern:** Every backend function that opens the Spreadsheet must call `loadConfig_()` first so `CONFIG.SPREADSHEET_ID` is loaded from Script Properties. Never assume CONFIG has the right value without loading it.

---

## SST / SSOMA Vocabulary for Content Generation

When generating content or AI prompts use these key SST terms:
- EPP: casco, guantes, lentes, zapatos punta acero, arnés, tapones, mascarilla, chaleco
- Peligros: caída a distinto nivel, golpe por objeto, atrapamiento, exposición a ruido, químicos, incendio, eléctrico
- Normativa: Ley 29783 (Peru), D.S. 005-2012-TR, ISO 45001, OHSAS 18001
- Documentos: IPERC, PETS, ATS, MSDS, permiso de trabajo, plan de emergencia
- Señales: prohibición (rojo), advertencia (amarillo), obligación (azul), emergencia (verde)
- Roles: supervisor SST, inspector, vigía, brigadista, médico ocupacional
