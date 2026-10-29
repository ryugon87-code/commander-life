# Commander Life — note per lo sviluppo

Segnapunti Commander (MTG) in **un solo file**, pixel-art, PWA. Niente build, niente
dipendenze: si apre direttamente in un browser.

## File e deploy
- `index.html` (GitHub Pages) e `commander-counter.html` (copia locale sul tablet)
  sono lo **stesso file**: ogni modifica va fatta su entrambi, tenendoli identici.
- `art/cards.js` definisce `window.__CARDS` (catalogo carte). `art/*.jpg` sono le
  immagini locali usate in modalità offline.
- `sw.js`: service worker (solo online). **A ogni modifica aumentare `CACHE`**
  (`commander-life-vNN`), altrimenti il browser serve la versione vecchia.
- Due modalità a runtime: **locale** `file:///sdcard/CommanderCounter/commander-counter.html`
  (offline, art da `art/*.jpg`) e **online** su GitHub Pages.
  `LOCAL = location.protocol === "file:"`.
- GitHub Pages pubblica da `main`: le modifiche si vedono online solo dopo il merge
  in `main`, non solo sul branch.

## Dispositivo di destinazione — Lenovo IdeaTab S6000-H
L'app DEVE girare qui. Ogni modifica va verificata contro questi limiti.
- Android 4.2.2, **Firefox 68 (Fennec)**: niente di più recente è installabile.
- Schermo **1280×800** orizzontale, 10.1". RAM **1 GB** (troppe immagini/canvas = crash).

### CSS da NON usare (FF68 scarta la regola in silenzio → layout rotto senza errori)
- `clamp()`, `min()`, `max()` (FF75) → valori fissi in `vw`/`vh`/`px`, media query,
  oppure calcolare in JS e passare px via custom property.
- unità `cqw`/`cqh`/`cqmin` e `container-type` / `@container` (FF110)
- `aspect-ratio` (FF89) → padding-top % o altezza esplicita
- `:is()` / `:where()` (FF78), `:has()` (FF121), `:focus-visible` (FF85)
- `dvh`/`svh`/`lvh` (FF101), `@layer` (FF97), CSS nesting (FF117)
- `color-mix()` (FF113), `backdrop-filter` (FF103), `accent-color` (FF92)

Sicuri: custom properties, flexbox con `gap`, grid, `inset`, `vw/vh`, `transform`,
`calc()` con var, `filter`, `env()`.

### JavaScript da NON usare (errore di sintassi = schermo intero bianco)
- `?.` (FF74), `??` (FF72), `||=` `&&=` `??=` (FF79)
- campi/metodi privati `#x` (FF90), `static` nei campi di classe (FF75)
- `.at()` (FF90), `.replaceAll()` (FF77), `structuredClone` (FF94), `Promise.any` (FF79)
- separatori numerici `1_000` (FF70), `ResizeObserver` (FF69) → evento `resize` su window

Scrivere ES5: `var`, `function`, concatenazione di stringhe, `Math.min/max`,
`element.style.setProperty`.

### API e comportamenti
- **Wake Lock API assente**: ok come miglioramento progressivo, ma non deve servire
  (lo schermo acceso sul tablet è gestito via adb).
- **Niente emoji come glifi** (il sistema non le ha) → SVG o testo.
- `file://` **non può fare fetch/XHR** di file locali: il catalogo si carica con
  `<script src="art/cards.js">`. Non sostituire con fetch. In `file://` le immagini
  delle carte vengono da `art/*.jpg` (`cardSrc` usa `c.file` quando `LOCAL`).
- `getUserMedia` (fotocamera) solo su https → in locale non c'è.
- `navigator.vibrate` chiede il permesso a ogni origine → default OFF.
- Tocco: il browser emette touch + mouse emulato → un tap può contare doppio.
  Usare `preventDefault` su touchstart/touchend e una guardia su mousedown
  (gli handler `click` normali invece scattano una volta sola, vanno bene).
- **Non** usare `?v=random` come cache-buster: su `file://` azzera localStorage/IndexedDB.

## Verifica
Non esiste FF68 in questo ambiente: i test headless (Chromium/Playwright) non
garantiscono il comportamento reale sul tablet. La prova definitiva è sul dispositivo.
Prima di dire "fatto", fare almeno l'audit statico:
`grep -nE "clamp\(|cqw|cqh|container-type|@container|aspect-ratio|:is\(|:where\(|:has\(|color-mix|backdrop-filter" index.html`
e il grep della sintassi JS vietata sopra.
