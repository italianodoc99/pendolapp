# PendolApp — Mappa del codice (`index.html`)

Tutto il codice è in un unico file HTML (~5480 righe).  
Struttura: `<head>` CSS → `<body>` HTML screens → `<script>` JS

---

## Sezioni principali (numeri di riga approssimativi)

| Riga | Sezione |
|------|---------|
| 1–15 | `<head>`, meta, Firebase imports |
| 16–1860 | **CSS** (`:root`, variabili, classi) |
| 1863–2510 | **HTML screens** (dentro `<body>`) |
| 2511–5477 | **JavaScript** (dentro `<script>`) |

---

## Screens HTML

| `id` | Riga | Descrizione |
|------|------|-------------|
| `sc-home` | 1863 | Home con chip data e ricerca rapida |
| `sc-search` | 1939 | Lista risultati passaggi |
| `sc-detail` | 2017 | Dettaglio passaggio + mappa |
| `sc-offer` | 2026 | Form pubblica passaggio |
| `sc-my-rides` | 2126 | I miei passaggi (driver) |
| `sc-bookings` | 2164 | Le mie prenotazioni (rider) |
| `sc-transit` | 2199 | Planner treni/bus |
| `sc-profile` | 2241 | Profilo utente |
| `sc-chat` | 2375 | Lista chat |
| `sc-chat-detail` | 2417 | Singola chat |
| `sc-ocr` | 2437 | Scanner orari OCR |

---

## Strutture dati JS

### `HUBS` — array di stringhe (riga ~3454)
Lista **statica** di fermate/luoghi usabili come nodi di cambio nel planner.  
Viene unita agli **hub dinamici** calcolati da `getDynamicHubs()`: non è più necessario aggiungerci ogni fermata nuova — basta che la fermata compaia in `stops[]` di almeno 2 linee in `OPERATORS` e diventa hub automaticamente.  
Mantenere `HUBS` per fermate usate dai passaggi carpooling che non sono in OPERATORS.

### `STOP_ALIASES` — oggetto `{ 'nome canonico': ['alias1', ...] }` (riga ~3522)
Normalizza input dell'utente verso il nome canonico. Chiave = nome come appare in `stops[]` degli operatori.

### `AC_PLACES` — array di stringhe (riga ~4156)
Lista per l'autocomplete dei campi from/to. Separata da HUBS.

### `OPERATORS` — array di oggetti (riga ~4252, dentro `renderTransitFull`)
Definisce tutte le linee di trasporto pubblico. **Inizializzato una sola volta** all'avvio e memorizzato in `window._OPERATORS`. Le successive chiamate a `renderTransitFull()` usano questa variabile globale e rigenerano solo l'HTML, senza ricostruire l'intero dataset.

**Struttura di un operatore:**

```javascript
{
  id: 'eav',
  name: 'EAV MetroCampania',
  color: '#e63946',
  lines: [
    {
      name: 'Nome linea → Direzione',
      noSun: true,        // opzionale: salta domenica
      onlySun: true,      // opzionale: solo domenica/festivi
      dur: 125,           // durata totale in minuti
      stops: ['Stop A', 'Stop B', ...],   // array ordinato
      deps: [
        {
          t: '07:50',          // orario di partenza
          s: 'Stop A',         // opzionale: fermata di partenza se non è stops[0]
          validity: 'FE6',     // opzionale: 'FE6'=lun-sab, 'SA'=solo sab, 'Sc'=scolastico
          times: {             // orari esatti per fermata (usati nel routing)
            'Stop A': '07:50',
            'Stop B': '08:10',
            'Stop C': '08:30',
          }
        }
      ]
    }
  ]
}
```

**Codici `validity`:**
- `'FE6'` / `'A6'` → Lunedì–Sabato
- `'SA'` → Solo Sabato
- `'Sc'` → Solo periodo scolastico (vedi `isSchoolPeriod()`)
- *(assente)* → Ogni giorno (filtrato da `noSun`/`onlySun`)

> **Nota:** Se `dep.times` è presente ma manca l'orario di una delle due fermate richieste, il sistema ora usa sempre il **fallback proporzionale** basato sugli indici di `stops[]` — il check che scartava la corsa in questi casi è stato rimosso (bug fix #5).

---

## Funzioni JS chiave

### Planner e transit

| Funzione | Riga | Descrizione |
|----------|------|-------------|
| `getDynamicHubs()` | ~3721 | Calcola hub dinamici da OPERATORS (fermate in 2+ linee) + HUBS statici. Risultato in cache `_dynamicHubs`; invalidare con `window._invalidateDynamicHubs()` se OPERATORS cambia a runtime |
| `transitForRoute(from, to, dateStr)` | ~3597 | Filtra OPERATORS per tratta e data; applica `noSun`/`onlySun`. Interno: `matchInText` usa 3 step (exact → substring → word-split). Il word-split usa soglia `> 5` caratteri per evitare falsi positivi |
| `buildJourneys(from, to, dateStr)` | ~3745 | Combina passaggi carpooling + transiti in Journey objects. Usa `cachedTransit()` per memoizzare `transitForRoute`. Internamente usa `addJourney()` con un `Set` di ID per deduplicare durante la costruzione (non solo alla fine). **5 fasi:** diretti ride / diretti transit / ride→transit / transit→transit / 2 cambi |
| `getCachedJourneys(from, to, date, callback)` | ~4015 | Wrapper asincrono di `buildJourneys`: usa `requestIdleCallback` (con fallback su `setTimeout`) per non bloccare il thread UI. Risultato passato via callback |
| `planJourney()` | ~4094 | Entry point planner (debounced 400ms): legge input, chiama `getCachedJourneys` in modo asincrono, applica filtri, calcola `slowWarning`, renderizza |
| `renderJourneys(jj, from, to)` | ~4097 | Genera HTML lista risultati planner. Badge `⚠️ percorso lento` se `j.slowWarning === true` |
| `isSchoolPeriod(dateStr)` | ~3586 | Ritorna true se la data è in periodo scolastico |
| `normalizeStop(q)` | ~3574 | Applica STOP_ALIASES a una stringa. **Memoizzata** in `_normalizeCache` (Map) per evitare ricalcoli ripetuti nei loop |
| `renderTransitFull()` | ~4252 | Render dell'intera schermata transit (contiene OPERATORS). Usa `_updateTransitTimeBadges()` per aggiornamenti incrementali |
| `_updateTransitTimeBadges(container, nowM)` | ~5283 | Aggiorna solo i badge orario (now/soon/later/gone) senza ricostruire il DOM. Chiamata anche da un `setInterval` ogni 30 secondi |

### Rides (carpooling)

| Funzione | Riga | Descrizione |
|----------|------|-------------|
| `listenRides()` | ~2728 | Firestore realtime listener su `rides` collection. L'unsubscribe è salvato in `window._ridesUnsub` e viene annullato al logout |
| `rideHTML(r)` | ~2770 | Template HTML per singolo passaggio |
| `publishRide()` | ~3038 | Crea nuovo documento in `rides` |
| `showMyRides()` | ~3097 | Screen driver: lista passaggi + richieste pending |
| `deleteRide(id)` | ~3154 | Elimina da Firestore |
| `editRide(id)` | ~3164 | Pre-popola form offri per modifica |
| `saveRideEdit(id)` | ~3183 | Aggiorna documento esistente |
| `openDetail(id)` | ~2826 | Apre sc-detail con mappa Leaflet |

### Prenotazioni

| Funzione | Riga | Descrizione |
|----------|------|-------------|
| `bookRide(rideId, driverId, driverName)` | ~3209 | Crea doc in `bookings` (status: pending) |
| `showMyBookings()` | ~3233 | Screen rider: lista prenotazioni proprie |
| `confirmBooking(bookingId)` | ~3267 | Driver: imposta status: confirmed |
| `cancelBooking(bookingId)` | ~3276 | Driver o rider: imposta status: cancelled |

### Recensioni

| Funzione | Riga | Descrizione |
|----------|------|-------------|
| `openReviewModal(bookingId, driverId, driverName)` | ~3294 | Apre modal stelle |
| `pickStar(n)` | ~3303 | Seleziona n stelle nel modal |
| `submitReview()` | ~3311 | Crea doc `reviews`, aggiorna media rating in `users` |

### Auth e profilo

| Funzione | Riga | Descrizione |
|----------|------|-------------|
| `doAuth()` | ~2685 | Login/register email+password |
| `doGoogleAuth()` | ~2714 | Login Google |
| `doLogout()` | ~2724 | Sign out (triggera cleanup listener via `onAuthStateChanged`) |

### Chat

| Funzione | Riga | Descrizione |
|----------|------|-------------|
| `listenChats()` | ~5323 | Listener Firestore lista chat. L'unsubscribe è salvato in `window._chatsUnsub` e viene annullato al logout |
| `openChatDetail(chatId, oname, ini)` | ~5347 | Apre sc-chat-detail. Salva listener messaggi in `_chatUnsub` (annulla il precedente se presente) |
| `openChatWith(otherUid, otherName, rideId)` | ~5372 | Crea o apre chat con un utente |
| `sendMsg()` | ~5391 | Invia messaggio nella chat corrente |

### UI e navigazione

| Funzione | Riga | Descrizione |
|----------|------|-------------|
| `goTo(screenId, navIdx)` | ~4217 | Naviga tra screens, aggiorna nav bar. **Dismette `_chatUnsub`** quando si abbandona `sc-chat-detail` |
| `showOv(type, arg)` | ~3338 | Apre overlay (filtri, date) |
| `acSearch(inp, listId)` | ~2883 | Autocomplete Nominatim (geolocalizzazione) |
| `acSuggest(inputId, dropId)` | ~4197 | Autocomplete da AC_PLACES (locale). Output con `escapeHtml()` per sicurezza XSS |
| `acSuggestDebounced(inputId, dropId)` | ~4213 | Versione debounced (200ms) di `acSuggest`, usata dagli input `home-from`/`home-to` |
| `safeStr(s)` | ~4142 | Sanitizza stringa per uso in attributi HTML inline (es. `onclick`) |
| `escapeHtml(s)` | ~4145 | Escape HTML per prevenire XSS nei dati dinamici (messaggi, nomi utente, autocomplete) |
| `toggleNearMe(el)` | ~3004 | Attiva/disattiva filtro "vicino a me". In caso di errore geolocalizzazione mostra un **banner inline** (non un `alert`) che scompare dopo 5 secondi |

---

## Come funziona il routing (buildJourneys)

`buildJourneys` esegue 5 fasi in sequenza e raccoglie tutti i percorsi possibili tramite `addJourney()`:

```
Fase 1 — Corse dirette (ride carpooling)              → cambi: 0
Fase 2 — Transiti diretti (una sola linea)            → cambi: 0
Fase 3 — ride → [hub] → transit                       → cambi: 1
Fase 4 — transit → [hub] → transit                    → cambi: 1
Fase 5a — ride → [hub1] → transit → [hub2] → transit  → cambi: 2
Fase 5b — transit → [hub1] → transit → [hub2] → transit → cambi: 2
```

**Deduplicazione:** `addJourney(j)` aggiunge il percorso solo se il suo `id` non è già nel `Set seen`. Il filtro è applicato **durante** la costruzione in tutte le fasi, non solo alla fine.

**Performance:** `cachedTransit(f, t)` memoizza ogni coppia (from, to) per l'intera sessione. `getCachedJourneys` usa `requestIdleCallback` per non bloccare l'UI durante il calcolo.

**Hub dinamici:** `getDynamicHubs()` costruisce la lista di nodi una volta sola (cached in `_dynamicHubs`) leggendo `window._OPERATORS`. Una fermata diventa hub se appare in `stops[]` di almeno 2 linee diverse.

**slowWarning:** dopo `buildJourneys`, `planJourney` calcola il miglior tempo diretto (`bestDirectMins`). Se un percorso a 2 cambi supera `bestDirectMins * 2`, riceve `slowWarning: true` e mostra il badge `⚠️ percorso lento`.

---

## Variabili globali JS (riga ~2553)

```javascript
let currentUser = null;          // Firebase Auth user object
let allRides = [];               // cache Firestore rides
let filterCambi = 0;             // filtro cambi planner (0=tutti)
let filterWait = 60;             // filtro attesa max planner (minuti)
let sortMode = 'time';           // ordinamento planner
let _dynamicHubs = null;         // cache hub calcolati da getDynamicHubs()

// Listener unsubscribe (gestione ciclo di vita)
window._ridesUnsub = null;       // unsubscribe listener rides Firestore
window._chatsUnsub = null;       // unsubscribe listener lista chat Firestore
let _chatUnsub = null;           // unsubscribe listener messaggi chat corrente

// Cache
window._transitCache = {};       // cache transitForRoute per coppia from|to|date
window._journeyCache = null;     // cache risultato buildJourneys
window._journeyCacheKey = null;  // chiave "from|to|date" della cache journey
const _normalizeCache = new Map(); // cache memoizzata di normalizeStop()
```

---

## Firestore collections

| Collection | Documenti | Note |
|-----------|-----------|------|
| `rides` | `{ from, to, depTime, price, seats, driverId, driverName, transit, createdAt }` | Realtime listener (`_ridesUnsub`) |
| `bookings` | `{ rideId, driverId, riderId, riderName, status, createdAt, reviewed? }` | status: pending/confirmed/cancelled |
| `reviews` | `{ bookingId, driverId, riderId, rating, createdAt }` | |
| `users` | `{ displayName, photoUrl, rating, reviewCount, type, ... }` | |
| `chats` | `{ participants[], lastMsg, lastMsgTime, names{}, unread{} }` | Realtime listener (`_chatsUnsub`) |
| `messages` | sub-collection di `chats` | Realtime listener (`_chatUnsub`) |

---

## CSS variabili principali (riga 16)

```css
--green: #1D9E75      --green-l: #E1F5EE     --green-d: #0F6E56
--blue: #378ADD       --blue-l: #E6F1FB      --blue-d: #185FA5
--coral: #D85A30      --coral-l: #FAECE7     --coral-d: #993C1D
--red: #A32D2D        --red-l: #FCEBEB
--txt: #1A1C19        --txt2: #5A6057        --txt3: #8C9189
--surf: #FFF          --surf2: #F2F3F1       --bg: #F7F8F6
--r: 14px             --rsm: 8px             --font: 'DM Sans', sans-serif
```

---

## Sicurezza XSS

Tutti i dati dinamici provenienti da Firestore o dall'utente devono essere trattati prima di inserirli nell'HTML:

- **`escapeHtml(s)`** → per contenuti mostrati come testo (messaggi, nomi utente, autocomplete items)
- **`safeStr(s)`** → per valori usati in attributi `onclick` inline (rimuove `'` e `"`)
- **NON usare** mai dati non sanitizzati direttamente in `innerHTML`

---

## Ricette rapide

### Aggiungere una linea di trasporto
1. Apri `renderTransitFull()` (riga ~4252)
2. Trova l'operatore nel array `OPERATORS` (o aggiungine uno nuovo)
3. Aggiungi `{ name, noSun?, onlySun?, dur, stops, deps }` all'array `lines`
4. Le fermate in `stops[]` diventano hub automatici se compaiono in 2+ linee — **non serve aggiungerle a `HUBS`**
5. Aggiungerle ad `AC_PLACES` (riga ~4156) per l'autocomplete from/to
6. Aggiungere alias in `STOP_ALIASES` (riga ~3522) se il nome ha varianti comuni
7. Chiamare `window._invalidateDynamicHubs()` se si aggiungono operatori a runtime

### Aggiungere una fermata nuova
1. `AC_PLACES` (riga ~4156) → aggiungi per autocomplete from/to
2. `STOP_ALIASES` (riga ~3522) → aggiungi varianti del nome se ambiguo
3. Nelle `deps[].times` delle linee che passano per quella fermata → `'Nome Fermata': 'HH:MM'`
4. **`HUBS` è necessario solo** se la fermata è usata da passaggi carpooling ma NON è in OPERATORS
5. Se `times` è parziale (manca la fermata di arrivo), il sistema usa ora automaticamente il fallback proporzionale — non serve aggiungere la fermata mancante se la direzione è corretta in `stops[]`

### Modificare orari di una corsa
- Cerca il nome della linea dentro `renderTransitFull` con Ctrl+F
- Modifica i valori in `times: { ... }` del `dep` interessato
- Se cambia l'orario di partenza aggiorna anche `t: 'HH:MM'`

### Aggiungere un campo al form "Pubblica passaggio"
1. HTML: aggiungi `<input id="o-nomecampo">` dentro `sc-offer` (riga ~2026)
2. `publishRide()` (riga ~3038): leggi il valore e aggiungilo al doc Firestore
3. `saveRideEdit()` (riga ~3183): stesso trattamento per la modifica
4. `rideHTML(r)` (riga ~2770): mostralo nella card

### Debug del routing (percorsi non trovati)
1. Verifica che le fermate siano in `stops[]` delle linee giuste in OPERATORS
2. Verifica che i nomi in `times: {}` siano normalizzabili via `STOP_ALIASES`
3. Se `dep.times` ha solo alcune fermate, il sistema ora usa il fallback proporzionale — verifica che l'ordine in `stops[]` rifletta la direzione corretta
4. Per hub dinamici: una fermata deve comparire in `stops[]` di almeno 2 linee per diventare hub

### Debug dei listener Firestore
- Al logout, `onAuthStateChanged` dismette automaticamente `_ridesUnsub`, `_chatsUnsub` e `_chatUnsub`
- Quando si abbandona `sc-chat-detail` via `goTo()`, `_chatUnsub` viene dismesso
- Se un listener non si aggiorna, verifica che non sia stato annullato prematuramente

---

## Come usare questo file nelle richieste future

Quando vuoi una modifica, dimmi:
- **Cosa cambiare** (es. "aggiungi corsa 08:15 Napoli→Piedimonte")
- **La sezione coinvolta** (es. "linea EAV verso Piedimonte in `renderTransitFull`")
- **Incolla solo quella sezione** (qualche decina di righe) + gli orari/dati necessari

Non serve mandare tutto `index.html`.
