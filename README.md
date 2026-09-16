# Prompt Studio

Applicazione web single-file per gestire una libreria di prompt e template riutilizzabili con variabili dinamiche. Tutto gira nel browser: nessun backend, nessuna build, nessuna dipendenza installata localmente.

Basta aprire `index.html` in un browser moderno (o servirlo con un qualsiasi web server statico).

---

## Indice

- [Panoramica](#panoramica)
- [Caratteristiche](#caratteristiche)
- [Avvio](#avvio)
- [Stack tecnico](#stack-tecnico)
- [Struttura del progetto](#struttura-del-progetto)
- [Modello dati](#modello-dati)
- [Sintassi delle variabili](#sintassi-delle-variabili)
- [Interfaccia](#interfaccia)
- [Flusso di utilizzo](#flusso-di-utilizzo)
- [Scorciatoie da tastiera](#scorciatoie-da-tastiera)
- [Command palette](#command-palette)
- [Import / Export](#import--export)
- [Persistenza](#persistenza)
- [Architettura del codice](#architettura-del-codice)
- [Accessibilità e responsive](#accessibilità-e-responsive)
- [Note e limiti](#note-e-limiti)

---

## Panoramica

Prompt Studio è un editor di prompt organizzato in tre concetti:

1. **Libreria** — l'insieme di tutti i template salvati.
2. **Template** — un prompt con metadati (titolo, categoria, descrizione) e un corpo testuale che può contenere variabili.
3. **Variabili** — segnaposto nel testo del template che vengono compilati al momento dell'uso.

Il flusso tipico è: si sceglie un template dalla sidebar, si compilano le variabili nella vista **Compila**, si copia il prompt finale generato. La vista **Modifica** serve invece a creare o modificare i template.

---

## Caratteristiche

- Libreria di prompt con ricerca testuale e filtro per categoria.
- Variabili inline con supporto a **valori liberi** e **preset** (menù di opzioni).
- Compilazione dal vivo del prompt con conteggio caratteri e stima token.
- Copia negli appunti con un click.
- Import/export dell'intera libreria in JSON.
- Persistenza automatica su `localStorage`.
- Tema chiaro/scuro con rilevamento della preferenza iniziale.
- Command palette (`Ctrl/⌘ + K`) per eseguire azioni e aprire prompt.
- Sidebar ridimensionabile con larghezza memorizzata.
- Layout responsive con drawer laterale su mobile.
- Zero dipendenze runtime locali (CDN per Vue e Tailwind).

---

## Avvio

### Aprire direttamente il file

Aprire `index.html` con doppio click funziona, ma alcune API (es. clipboard) richiedono un contesto sicuro (`https` o `localhost`).

### Servire in locale (consigliato)

```bash
cd prompt-keep
python3 -m http.server 8080
```

Poi visitare `http://localhost:8080`.

---

## Stack tecnico

| Componente | Uso |
|---|---|
| [Vue 3](https://vuejs.org/) (build globale da CDN) | Reattività, stato, rendering dichiarativo |
| [Tailwind CSS](https://tailwindcss.com/) (Play CDN) | Utility class e tema scuro |
| Google Fonts | Font `Geist` e `Geist Mono` |
| Web Storage API | Persistenza di template, tema e larghezza sidebar |
| Clipboard API + fallback `execCommand` | Copia del prompt |

Nessun bundler, nessun `package.json`, nessun processo di build.

---

## Struttura del progetto

```text
prompt-keep/
└── index.html   # Applicazione completa: markup, stili, template Vue e logica
```

Tutto è contenuto in un unico file, suddiviso in sezioni:

- `<head>` — configurazione Tailwind, font, variabili CSS del tema, stili dei componenti.
- `<body>` — sprite SVG delle icone, markup dell'app (gestito da Vue), `<script>` con la logica.

---

## Modello dati

Ogni template è un oggetto con la seguente forma:

```json
{
  "id": "1750000000000",
  "title": "Code reviewer senior",
  "category": "Sviluppo",
  "description": "Analisi del codice per sicurezza, performance e leggibilità.",
  "content": "Agisci come {linguaggio:JavaScript,Python} e analizza:\n{codice}",
  "variablePresets": {}
}
```

| Campo | Tipo | Descrizione |
|---|---|---|
| `id` | string | Identificatore univoco (timestamp in millisecondi per i nuovi template). |
| `title` | string | Titolo mostrato nella sidebar e nell'header. |
| `category` | string | Categoria per il filtro. Se vuota viene trattata come `Generale`. |
| `description` | string | Riga descrittiva opzionale, mostrata nel riquadro del template. |
| `content` | string | Corpo del prompt, può contenere variabili. |
| `variablePresets` | object | Campo legacy per i preset (`{ "nome": ["a","b"] }`). Oggi i preset si definiscono inline; se presente viene usato come fallback. |

La libreria è semplicemente un array di questi oggetti.

---

## Sintassi delle variabili

Le variabili si scrivono tra parentesi graffe nel corpo del template.

### Variabile a testo libero

```text
Riassumi il seguente testo: {testo}
```

Nella vista **Compila** la variabile appare come token cliccabile; al click si apre un popover con una textarea per inserire il valore.

### Variabile con preset

```text
Traduci in {lingua:Italiano,Inglese,Spagnolo,Francese}
```

La parte dopo `:` è una lista separata da virgole. Nella vista **Compila** il popover mostra le opzioni come elenco selezionabile invece di una textarea.

### Regole

- Il nome deve corrispondere a `[a-zA-Z0-9_]+`.
- Se lo stesso nome compare più volte, è considerata **una sola variabile** e il valore viene riutilizzato ovunque.
- Se per una variabile esistono più definizioni con preset diversi, l'ultima opzione non vuota trovata viene usata come elenco preset (il parser sovrascrive `presets[name]`).
- I prefissi dei preset vengono rimossi dal testo compilato: `{lingua:...}` diventa semplicemente il valore scelto.
- Una variabile non compilata resta visibile nel testo come `{nome}`.

Regex usata dal parser:

```js
/\{([a-zA-Z0-9_]+)(?::([^}]*))?\}/g
```

---

## Interfaccia

L'interfaccia è composta da quattro aree principali.

### 1. Title bar

- Logo e nome dell'app.
- Pulsante **Comandi** (apre la command palette).
- Toggle tema chiaro/scuro.
- Import/Export JSON.
- Pulsante **Nuovo prompt**.
- Su mobile: pulsante per aprire la sidebar.

### 2. Sidebar (libreria)

- Campo di ricerca (filtra per titolo e contenuto).
- Chip delle categorie (`Tutti` + categorie presenti).
- Elenco dei template filtrati, con titolo, categoria e anteprima.
- Maniglia di ridimensionamento sul bordo destro (solo desktop).
- Su schermi < 768px diventa un drawer a scomparsa con overlay.

### 3. Workbench (area di lavoro)

Ha due modalità, controllate da un selettore segmentato:

**Compila**
- Riquadro "Prompt compilato" con il testo e i token delle variabili.
- I token non compilati sono tratteggiati; quelli compilati diventano pieni (colore accento).
- Conteggio `compilate/totali`, pulsante **Svuota** e statistiche `caratteri · ~token`.
- Pulsante **Copia prompt**.

**Modifica**
- Sezione "Dettagli": titolo, categoria, descrizione.
- Sezione "Testo del template": textarea monospace con anteprima delle variabili rilevate (libere o con preset).
- Indicatore "non salvato" e pulsanti **Annulla** / **Salva**.

Se nessun template è selezionato, il workbench mostra uno stato vuoto con CTA per crearne uno.

### 4. Status bar

Mostra numero di prompt, titolo attivo, stato delle modifiche e promemoria delle scorciatoie.

### Overlay

- **Command palette**, **dialog di conferma**, **toast** e **popover delle variabili** sono overlay gestiti con transizioni dedicate.

---

## Flusso di utilizzo

### Creare un nuovo prompt

1. **Nuovo prompt** (title bar, status bar o tasto `n`).
2. Il template viene creato con valori predefiniti e si apre direttamente in modifica.
3. Compilare titolo, categoria e descrizione.
4. Scrivere il corpo usando variabili, ad es. `Agisci come {ruolo} e analizza {argomento}`.
5. **Salva**.

### Compilare un prompt

1. Selezionare un template dalla sidebar (o dalla command palette).
2. Nella vista **Compila**, cliccare un token variabile.
3. Inserire un valore libero oppure scegliere un preset.
4. Ripetere per tutte le variabili.
5. **Copia prompt** (o `Ctrl/⌘ + Invio`).

### Modificare un prompt

1. Selezionare il template e passare a **Modifica**.
2. Modificare i campi.
3. **Salva** (le modifiche sono persistenti) oppure **Annulla**.
4. Se ci sono modifiche non salvate e si prova a cambiare tab o template, appare un dialog di conferma.

### Eliminare un prompt

- **Modifica → Elimina** oppure dalla command palette.
- La cancellazione richiede sempre conferma. Se si elimina il template attivo viene selezionato il primo disponibile.

---

## Scorciatoie da tastiera

| Scorciatoia | Azione |
|---|---|
| `Ctrl`/`⌘` + `K` | Apri/chiudi la command palette |
| `Ctrl`/`⌘` + `Invio` | Copia il prompt compilato (vista Compila) |
| `Esc` | Chiude popover variabile, palette, dialog o drawer |
| `/` | Porta il focus sulla ricerca della sidebar |
| `n` | Crea un nuovo prompt |
| `e` | Passa alla modalità Modifica |

Le scorciatoie a singolo tasto (`/`, `n`, `e`) sono ignorate quando il focus è su un campo di input, una textarea o un elemento editabile, per non interferire con la digitazione.

---

## Command palette

Aperta con `Ctrl/⌘ + K`, offre un elenco filtrabile di:

- **Azioni** — nuovo prompt, compila/modifica/copia/elimina il prompt corrente.
- **Prompt** — tutti i template della libreria, con la categoria come suggerimento.
- **Interfaccia** — cambio tema, export, import.

La navigazione avviene con le frecce `↑`/`↓`, l'esecuzione con `Invio` (o click), la chiusura con `Esc`. Il filtro è case-insensitive sull'etichetta.

---

## Import / Export

### Export

Genera un file `prompt_backup_YYYY-MM-DD.json` contenente l'intera libreria serializzata con indentazione a 2 spazi. Il download è gestito tramite un data-URL e un elemento `<a>` temporaneo.

### Import

Legge un file `.json` selezionato tramite input file. Il contenuto deve essere un **array** di template. In caso contrario viene mostrato l'errore "File JSON non valido" e la libreria resta invariata.

> Nota: l'import **sostituisce** l'intera libreria corrente, non la unisce.

---

## Persistenza

Tutti i dati sono salvati nel `localStorage` del browser sotto tre chiavi:

| Chiave | Contenuto |
|---|---|
| `prompt_studio_templates` | Array JSON di tutti i template |
| `prompt_studio_theme` | `"dark"` o `"light"` |
| `prompt_studio_sidebar` | Larghezza della sidebar in pixel |

Dettagli:

- I template vengono salvati automaticamente ad ogni modifica grazie a un `watch` profondo.
- Al primo avvio, se non esiste alcun salvataggio, vengono caricati tre template dimostrativi (`DEFAULT_TEMPLATES`).
- Se il salvataggio risulta corrotto o non è un array, si ricade sui template predefiniti.
- Il tema iniziale è scuro, salvo preferenza salvata.
- La larghezza della sidebar è vincolata tra 240 e 460 px.

---

## Architettura del codice

L'app è un singolo componente Vue creato con `createApp({ setup() { ... } })`.

### Stato reattivo principale

| Ref / Computed | Ruolo |
|---|---|
| `templates` | Array dei template |
| `activeTemplateId` / `activeTemplate` | Template selezionato |
| `activeTab` | `'fill'` o `'edit'` |
| `searchQuery` / `selectedCategory` | Filtri della sidebar |
| `variableValues` | Mappa `nome → valore` per la compilazione |
| `editDraft` / `editBaseline` / `editDirty` | Bozza di modifica e rilevamento modifiche |
| `categories` / `filteredTemplates` | Liste derivate per la UI |
| `compiledSegments` / `compiledPrompt` | Prompt compilato segmentato |
| `charCount` / `tokenCount` / `filledCount` | Statistiche |
| `isDark`, `sidebarWidth`, `sidebarOpen`, `isNarrow`, `isResizing` | Stato UI |
| `toast`, `confirmState`, `palette`, `varEditor` | Stato degli overlay |

### Funzioni chiave

- `parseVariables(text)` — estrae nomi variabili e preset tramite regex.
- `compiledSegments` — costruisce l'array di segmenti `text`/`var` per il rendering e per il conteggio.
- `startEdit` / `saveEdit` / `requestTab` — gestione del ciclo di modifica.
- `selectTemplate` / `createNewTemplate` / `executeDelete` — CRUD dei template.
- `copyPrompt` — copia con Clipboard API e fallback.
- `exportTemplates` / `importTemplates` — serializzazione e lettura JSON.
- `openVarEditor` / `setVarValue` / `clearVar` — popover di compilazione con posizionamento dinamico (apertura verso l'alto se manca spazio in basso).
- `startResize` — ridimensionamento sidebar via eventi pointer.
- `onGlobalKey` — gestione centralizzata delle scorciatoie.

### Ciclo di vita

- `onMounted`: carica template/tema/sidebar da `localStorage`, imposta la media query responsive, registra i listener globali (`keydown`, `scroll` in cattura, `resize`).
- `onUnmounted`: rimuove listener e timer.
- `watch(templates, ..., { deep: true })`: salva automaticamente la libreria.

### Stima dei token

Il numero di token è una stima grezza calcolata come `ceil(caratteri / 4)`, utile solo come indicatore indicativo.

---

## Accessibilità e responsive

- Attributi `aria-label` sui pulsanti icona, `role="status"` sul toast e `role="dialog"` sul popover.
- Focus visibile personalizzato (`:focus-visible`).
- Rispetto di `prefers-reduced-motion`: le transizioni vengono disattivate.
- Sotto i 768px la sidebar diventa un drawer con overlay; la maniglia di ridimensionamento è nascosta.
- Il layout si adatta progressivamente nascondendo etichette e promemoria delle scorciatoie su schermi piccoli.

---

## Note e limiti

- L'app è **solo client-side**: la libreria non è sincronizzata tra dispositivi o browser.
- Cancellare i dati del browser (o il `localStorage`) comporta la perdita dei template: fare export regolari.
- L'import non fa merge: sovrascrive tutta la libreria.
- I preset `variablePresets` restano supportati solo come fallback legacy; il modo consigliato è la sintassi inline `{nome:a,b,c}`.
- La copia negli appunti può fallire fuori da contesti sicuri; in tal caso viene usato il fallback `document.execCommand('copy')`.
- Il tema segue una preferenza interna (default scuro), non il `prefers-color-scheme` di sistema.
