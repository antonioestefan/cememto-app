# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Operations panel for a Cruz Azul cement distributor ("Distribuidora Cruz Azul — Antonio"). Tracks cement sale orders (Órdenes de Venta, OVs) through a delivery lifecycle, manages client accounts and advance payments (anticipos), and generates Excel/WhatsApp reports.

## Two versions of the app

There are two parallel implementations:

**`index.html` (root) — active production version**
- Single-file React app. No build step. React, ReactDOM, Babel Standalone, and XLSX are loaded from CDN.
- JSX is transpiled in-browser by Babel (`type="text/babel"`).
- Persists all data to **Firebase Firestore** (project: `cemento-antonio`). Collections: `clientes` and `ovs`.
- Has the full feature set: OV lifecycle, client account balance, plant reports, monthly reports, anticipo alerts, delete OV.
- **This is the file you should edit for any feature work.**

**`src/` directory — legacy Vite scaffold (not deployed)**
- `src/App.jsx` is a Vite+React version with hardcoded in-memory data (no Firestore). Used for local experimentation.
- `src/package.json` has scripts: `npm run dev` (Vite dev server), `npm run build`, `npm run preview`.
- This version is behind the root `index.html` in features and is not the deployed app.

## Running the app

**Production version (`index.html`):** Open the file directly in a browser, or deploy to Vercel. There is no build step. Firebase SDK is loaded via ES module imports from `gstatic.com`.

**Vite version (local dev only):**
```
cd src
npm install
npm run dev
```

## Deployment

Deployed to Vercel as a static site. The root `vercel.json` serves `index.html` for all routes.

## Architecture of `index.html`

All application logic lives in a single `<script type="text/babel">` block inside `index.html`. Key parts:

- **Firebase initialization** runs in a separate `<script type="module">` block and exposes the Firestore API on `window` globals (`window.__db`, `window.__setDoc`, etc.) because ES modules and Babel scripts can't share scope directly. The React app listens for a `dbReady` event before subscribing to Firestore.

- **OV lifecycle** (`STATUSES`): `ov_asignada` → `fletera_planta` → `remisionada` → `en_camino` → `entregado`. The `NEXT` map drives the advance button; status keys are the source of truth.

- **Plant detection** (`plantaDeSerie`): The `serie` field prefix determines which plant loaded the cement — `PU` = Puebla, `AG` = Aguascalientes. This is used in the "Plantas" tab for monthly tonnage reports.

- **`estadoCuenta`** (memoized): Computed per-client summary — total tons delivered, value delivered, advance paid, and remaining balance (`anticipo - valor_entregado`). A negative balance means the client owes money.

- **`upd(id, patch)`**: The single mutation function for OVs. Updates local state optimistically and then persists to Firestore via `saveToDb`.

- **All inline styles**: No CSS file or CSS-in-JS library. Styles are written as JS objects directly on elements. The shared input style object is `inS`.

- **Modals** (OV form, client form) are fixed-position bottom sheets rendered conditionally. The detail panel (`selected`) is also a fixed bottom panel.

## Data shapes

**OV fields:** `id`, `num_ov`, `clienteId`, `serie`, `folio`, `toneladas`, `fecha_carga`, `fecha_entrega`, `status`, `confirm` (`por_confirmar`|`confirmado`|`reprogramado`), `motivo`, `remision_firmada`, `notas`

**Client fields:** `id`, `nombre`, `precio_ton`, `anticipo`

All monetary values are MXN. Dates are ISO strings `YYYY-MM-DD`. Toneladas are stored as floats — always use `parseFloat()` when reading, not direct arithmetic.
