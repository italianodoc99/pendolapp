# PendolApp — Mappa del codice (`index.html`)

Tutto il codice è in un unico file HTML (~5300 righe).  
Struttura: `<head>` CSS → `<body>` HTML screens → `<script>` JS

---

## Sezioni principali (numeri di riga approssimativi)

| Riga | Sezione |
|------|---------|
| 1–15 | `<head>`, meta, Firebase imports |
| 16–2510 | **CSS** (`:root`, variabili, classi) |
| 1863–2510 | **HTML screens** (dentro `<body>`) |
| 2511–5300 | **JavaScript** (dentro `<script>`) |

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

### `HUBS` — array di stringhe (riga ~3351)
Lista **statica** di fermate/luoghi usabili come nodi di cambio nel planner.  
Viene unita agli **hub dinamici** calcolati da `getDynamicHubs()` (vedi sotto): non è più necessario aggiungerci ogni fermata nuova — basta che la fermata compaia in `stops[]` di almeno 2 linee in `OPERATORS` e diventa hub automaticamente.  
Mantenere `HUBS` per fermate usate dai passaggi carpooling (che non sono in OPERATORS).

### `STOP_ALIASES` — oggetto `{ 'nome canonico': ['alias1', ...] }` (riga ~3419)
Normalizza input dell'utente verso il nome canonico. Chiave = nome come appare in `stops[]` degli operatori.

### `AC_PLACES` — array di stringhe (riga ~3960)
Lista per l'autocomplete dei campi from/to. Separata da HUBS.

### `OPERATORS` — array di oggetti (riga ~4050, dentro `renderTransitFull`)
Definisce tutte le linee di trasporto pubblico. **Struttura di un operatore:**

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
          t: '07:50',          // orario di partenza (dalla prima fermata o da s:)
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

---

## Funzioni JS chiave

### Planner e transit

| Funzione | Riga | Descrizione |
|----------|------|-------------|
| `getDynamicHubs()` | ~3614 | Calcola hub dinamici da OPERATORS (fermate in 2+ linee) + HUBS statici. Risultato in cache `_dynamicHubs`; invalidare con `window._invalidateDynamicHubs()` se OPERATORS cambia a runtime |
| `transitForRoute(from, to, dateStr)` | ~3489 | Filtra OPERATORS per tratta e data; applica `noSun`/`onlySun`. Interno: `matchInText` usa 3 step (exact → substring → word-split). Il word-split usa soglia `> 5` caratteri per evitare falsi positivi (es. "monte" in "Piedimonte") |
| `buildJourneys(from, to, dateStr)` | ~3640 | Combina passaggi carpooling + transiti in Journey objects. Internamente usa `cachedTransit()` per memoizzare le chiamate a `transitForRoute`. **5 fasi:** diretti ride / diretti transit / ride→transit / transit→transit / 2 cambi (ride→t→t e t→t→t) |
| `planJourney()` | ~3860 | Entry point planner: legge input, chiama buildJourneys, applica filtri, calcola `slowWarning`, renderizza |
| `renderJourneys(jj, from, to)` | ~3910 | Genera HTML lista risultati planner. Badge `⚠️ percorso lento` se `j.slowWarning === true` |
| `isSchoolPeriod(dateStr)` | ~3478 | Ritorna true se la data è in periodo scolastico |
| `normalizeStop(q)` | ~3469 | Applica STOP_ALIASES a una stringa |
| `renderTransitFull()` | ~4050 | Render dell'intera schermata transit (contiene OPERATORS) |

### Rides (carpooling)

| Funzione | Riga | Descrizione |
|----------|------|-------------|
| `listenRides()` | 2661 | Firestore realtime listener su `rides` collection |
| `rideHTML(r)` | 2676 | Template HTML per singolo passaggio |
| `publishRide()` | 2938 | Crea nuovo documento in `rides` |
| `showMyRides()` | 2995 | Screen driver: lista passaggi + richieste pending |
| `deleteRide(id)` | 3052 | Elimina da Firestore |
| `editRide(id)` | 3062 | Pre-popola form offri per modifica |
| `saveRideEdit(id)` | 3081 | Aggiorna documento esistente |
| `openDetail(id)` | 2752 | Apre sc-detail con mappa Leaflet |

### Prenotazioni

| Funzione | Riga | Descrizione |
|----------|------|-------------|
| `bookRide(rideId, driverId, driverName)` | 3106 | Crea doc in `bookings` (status: pending) |
| `showMyBookings()` | 3130 | Screen rider: lista prenotazioni proprie |
| `confirmBooking(bookingId)` | 3164 | Driver: imposta status: confirmed |
| `cancelBooking(bookingId)` | 3173 | Driver o rider: imposta status: cancelled |

### Recensioni

| Funzione | Riga | Descrizione |
|----------|------|-------------|
| `openReviewModal(bookingId, driverId, driverName)` | 3191 | Apre modal stelle |
| `pickStar(n)` | 3200 | Seleziona n stelle nel modal |
| `submitReview()` | 3208 | Crea doc `reviews`, aggiorna media rating in `users` |

### Auth e profilo

| Funzione | Riga | Descrizione |
|----------|------|-------------|
| `doAuth()` | 2619 | Login/register email+password |
| `doGoogleAuth()` | 2648 | Login Google |
| `doLogout()` | 2658 | Sign out |
| `renderProfileBadges(profile)` | 3315 | Badge verifiche social nel profilo |

### UI e navigazione

| Funzione | Riga | Descrizione |
|----------|------|-------------|
| `goTo(screenId, navIdx)` | ~4020 | Naviga tra screens, aggiorna nav bar |
| `showOv(type, arg)` | 3235 | Apre overlay (filtri, date) |
| `renderHomeDateChips()` | 3259 | Chip date sulla home |
| `acSearch(inp, listId)` | 2796 | Autocomplete Nominatim (geolocalizzazione) |
| `acSuggest(inputId, dropId)` | ~4000 | Autocomplete da AC_PLACES (locale) |
| `safeStr(s)` | ~3957 | Sanitizza stringa per uso in attributi HTML |

---

## Come funziona il routing (buildJourneys)

`buildJourneys` esegue 5 fasi in sequenza e raccoglie tutti i percorsi possibili:

```
Fase 1 — Corse dirette (ride carpooling)         → cambi: 0
Fase 2 — Transiti diretti (una sola linea)       → cambi: 0
Fase 3 — ride → [hub] → transit                  → cambi: 1
Fase 4 — transit → [hub] → transit               → cambi: 1  ← aggiunta
Fase 5a — ride → [hub1] → transit → [hub2] → transit  → cambi: 2  ← aggiunta
Fase 5b — transit → [hub1] → transit → [hub2] → transit → cambi: 2  ← aggiunta
```

**Performance:** `cachedTransit(f, t)` memoizza ogni coppia (from, to) per l'intera durata di `buildJourneys`. Ogni coppia viene calcolata una sola volta anche se hub1/hub2 si ripetono.

**Hub dinamici:** `getDynamicHubs()` costruisce la lista di nodi una volta sola (poi cached in `_dynamicHubs`) leggendo `window._OPERATORS`. Una fermata diventa hub se appare in `stops[]` di almeno 2 linee diverse.

**slowWarning:** dopo `buildJourneys`, `planJourney` calcola il miglior tempo diretto (`bestDirectMins`). Se un percorso a 2 cambi supera `bestDirectMins * 2`, riceve `slowWarning: true` e mostra il badge `⚠️ percorso lento`.

---

## Variabili globali JS (riga ~2553)

```javascript
let currentUser = null;     // Firebase Auth user object
let allRides = [];          // cache Firestore rides
let filterCambi = 0;        // filtro cambi planner (0=tutti)
let filterWait = 60;        // filtro attesa max planner (minuti)
let sortMode = 'time';      // ordinamento planner
let _selDate = null;        // data selezionata (YYYY-MM-DD o null=oggi)
let _dynamicHubs = null;    // cache hub calcolati da getDynamicHubs()
```

---

## Firestore collections

| Collection | Documenti | Note |
|-----------|-----------|------|
| `rides` | `{ from, to, depTime, price, seats, driverId, driverName, transit, createdAt }` | Realtime listener |
| `bookings` | `{ rideId, driverId, riderId, riderName, status, createdAt, reviewed? }` | status: pending/confirmed/cancelled |
| `reviews` | `{ bookingId, driverId, riderId, rating, createdAt }` | |
| `users` | `{ displayName, photoUrl, rating, reviewCount, type, ... }` | |
| `chats` | `{ participants[], lastMsg, updatedAt }` | |
| `messages` | sub-collection di `chats` | |

---

## CSS variabili principali (riga 16)

```css
--blue: #2563eb       --blue-l: #eff6ff      --blue-d: #1e40af
--red: #e53e3e        --red-l: #fff5f5
--txt: #111827        --txt2: #6b7280        --txt3: #9ca3af
--card: #ffffff       --bg: #f3f4f6
--r: 14px             --rsm: 8px             --font: 'Inter', sans-serif
```

---

## Ricette rapide

### Aggiungere una linea di trasporto
1. Apri `renderTransitFull()` (riga ~4050)
2. Trova l'operatore nel array `OPERATORS` (o aggiungine uno nuovo)
3. Aggiungi `{ name, noSun?, onlySun?, dur, stops, deps }` all'array `lines`
4. Le fermate in `stops[]` diventano hub automatici se compaiono in 2+ linee — **non serve aggiungerle a `HUBS`**
5. Aggiungerle ad `AC_PLACES` (riga ~3960) per l'autocomplete from/to
6. Aggiungere alias in `STOP_ALIASES` (riga ~3419) se il nome ha varianti comuni
7. Chiamare `window._invalidateDynamicHubs()` se si aggiungono operatori a runtime

### Aggiungere una fermata nuova
1. `AC_PLACES` (riga ~3960) → aggiungi per autocomplete from/to
2. `STOP_ALIASES` (riga ~3419) → aggiungi varianti del nome se ambiguo
3. Nelle `deps[].times` delle linee che passano per quella fermata → `'Nome Fermata': 'HH:MM'`
4. **`HUBS` è necessario solo** se la fermata è usata da passaggi carpooling ma NON è in OPERATORS

### Modificare orari di una corsa
- Cerca il nome della linea dentro `renderTransitFull` con Ctrl+F
- Modifica i valori in `times: { ... }` del `dep` interessato
- Se cambia l'orario di partenza aggiorna anche `t: 'HH:MM'`

### Aggiungere un campo al form "Pubblica passaggio"
1. HTML: aggiungi `<input id="o-nomecampo">` dentro `sc-offer` (riga ~2026)
2. `publishRide()` (riga ~2938): leggi il valore e aggiungilo al doc Firestore
3. `saveRideEdit()` (riga ~3081): stesso trattamento per la modifica
4. `rideHTML(r)` (riga ~2676): mostralo nella card

### Debug del routing (percorsi non trovati)
1. Verifica che le fermate siano in `stops[]` delle linee giuste in OPERATORS
2. Verifica che i nomi in `times: {}` siano normalizzabili via `STOP_ALIASES`
3. Controlla che `deps[].times` contenga entrambe le fermate (from e to): se ne manca una la corsa viene scartata
4. Per hub dinamici: una fermata deve comparire in `stops[]` di almeno 2 linee per diventare hub

---

## Come usare questo file nelle richieste future

Quando vuoi una modifica, dimmi:
- **Cosa cambiare** (es. "aggiungi corsa 08:15 Napoli→Piedimonte")
- **La sezione coinvolta** (es. "linea EAV verso Piedimonte in `renderTransitFull`")
- **Incolla solo quella sezione** (qualche decina di righe) + gli orari/dati necessari

Non serve mandare tutto `index.html`.
