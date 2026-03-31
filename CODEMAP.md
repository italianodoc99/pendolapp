# PendolApp — Mappa del codice (`index.html`)

Tutto il codice è in un unico file HTML (~5000 righe).  
Struttura: `<head>` CSS → `<body>` HTML screens → `<script>` JS

---

## Sezioni principali (numeri di riga approssimativi)

| Riga | Sezione |
|------|---------|
| 1–15 | `<head>`, meta, Firebase imports |
| 16–2510 | **CSS** (`:root`, variabili, classi) |
| 1863–2510 | **HTML screens** (dentro `<body>`) |
| 2511–5100 | **JavaScript** (dentro `<script>`) |

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
Nomi di fermate/luoghi usati nel planner. Aggiungere qui una nuova fermata per includerla nei suggerimenti del transit.

### `STOP_ALIASES` — oggetto `{ 'nome canonico': ['alias1', ...] }` (riga ~3419)
Normalizza input dell'utente verso il nome canonico. Chiave = nome come appare in `stops[]` degli operatori.

### `AC_PLACES` — array di stringhe (riga ~3796)
Lista per l'autocomplete dei campi from/to. Separata da HUBS.

### `OPERATORS` — array di oggetti (riga ~3885, dentro `renderTransitFull`)
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
| `transitForRoute(from, to, dateStr)` | 3489 | Filtra OPERATORS per tratta e data; applica `noSun`/`onlySun` |
| `buildJourneys(from, to, dateStr)` | 3611 | Combina passaggi carpooling + transiti in Journey objects |
| `planJourney()` | 3697 | Entry point planner: legge input, chiama buildJourneys, renderizza |
| `renderJourneys(jj, from, to)` | 3750 | Genera HTML lista risultati planner |
| `isSchoolPeriod(dateStr)` | 3478 | Ritorna true se la data è in periodo scolastico |
| `normalizeStop(q)` | 3469 | Applica STOP_ALIASES a una stringa |
| `renderTransitFull()` | 3884 | Render dell'intera schermata transit (contiene OPERATORS) |

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
| `goTo(screenId, navIdx)` | 3855 | Naviga tra screens, aggiorna nav bar |
| `showOv(type, arg)` | 3235 | Apre overlay (filtri, date) |
| `renderHomeDateChips()` | 3259 | Chip date sulla home |
| `acSearch(inp, listId)` | 2796 | Autocomplete Nominatim (geolocalizzazione) |
| `acSuggest(inputId, dropId)` | 3837 | Autocomplete da AC_PLACES (locale) |
| `safeStr(s)` | 3793 | Sanitizza stringa per uso in attributi HTML |

---

## Variabili globali JS (riga ~2553)

```javascript
let currentUser = null;     // Firebase Auth user object
let allRides = [];          // cache Firestore rides
let filterCambi = 0;        // filtro cambi planner (0=tutti)
let filterWait = 60;        // filtro attesa max planner (minuti)
let sortMode = 'time';      // ordinamento planner
let _selDate = null;        // data selezionata (YYYY-MM-DD o null=oggi)
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
1. Apri la sezione `renderTransitFull()` (riga ~3884)
2. Trova l'operatore giusto nel array `OPERATORS` (o aggiungine uno nuovo)
3. Aggiungi un oggetto `{ name, noSun?, onlySun?, dur, stops, deps }` all'array `lines`
4. Aggiungi le fermate nuove in `HUBS` (riga ~3351) e `AC_PLACES` (riga ~3796)
5. Aggiungi alias in `STOP_ALIASES` (riga ~3419) se il nome ha varianti comuni

### Aggiungere una fermata nuova
1. `HUBS` → aggiungi il nome esatto come stringa
2. `AC_PLACES` → stesso nome (per autocomplete da/a)
3. `STOP_ALIASES` → aggiungi chiave con varianti se il nome è ambiguo
4. Nelle `deps[].times` delle linee che passano per quella fermata → aggiungi `'Nome Fermata': 'HH:MM'`

### Modificare orari di una corsa
- Cerca il nome della linea dentro `renderTransitFull` con Ctrl+F
- Modifica i valori in `times: { ... }` del `dep` interessato
- Se cambia l'orario di partenza aggiorna anche `t: 'HH:MM'`

### Aggiungere un campo al form "Pubblica passaggio"
1. HTML: aggiungi `<input id="o-nomecampo">` dentro `sc-offer` (riga ~2026)
2. `publishRide()` (riga ~2938): leggi il valore e aggiungilo al doc Firestore
3. `saveRideEdit()` (riga ~3081): stesso trattamento per la modifica
4. `rideHTML(r)` (riga ~2676): mostralo nella card

---

## Come usare questo file nelle richieste future

Quando vuoi una modifica, dimmi:
- **Cosa cambiare** (es. "aggiungi corsa 08:15 Napoli→Piedimonte")
- **La sezione coinvolta** (es. "linea EAV verso Piedimonte in `renderTransitFull`")
- **Incolla solo quella sezione** (qualche decina di righe) + gli orari/dati necessari

Non serve mandare tutto `index.html`.
