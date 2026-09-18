# Corsi Plus · CT Scaligero

Web app di iscrizione ai corsi Plus per gli agonisti. Nessun login: l'atleta sceglie
il proprio nome da un elenco e si iscrive alle sessioni della settimana in corso.

## Regole

- Ci si iscrive **solo per la settimana in corso**.
- Posti titolari per modulo: Forza 5, Velocita' 8, Resistenza 8, Calcettone 10 (modificabili).
- A posti esauriti ci si mette in **riserva**: fino a 10, in ordine di iscrizione.
- **Nessuno puo' cancellare o sovrascrivere l'iscrizione di un altro**: il database accetta
  solo letture e inserimenti. Solo i preparatori, con il PIN, possono rimuovere un'iscrizione.
- Titolari e riserve non sono salvati come tali: si calcolano leggendo le iscrizioni in ordine
  cronologico. Cosi' due iscrizioni simultanee non entrano mai in conflitto.

---

## 1. Creare il database su Supabase

1. Vai su [supabase.com](https://supabase.com) e crea un progetto (piano gratuito).
2. Apri **SQL Editor** e incolla tutto questo blocco, poi **Run**:

```sql
create table iscrizioni (
  id uuid primary key default gen_random_uuid(),
  week_key text not null,
  day int not null check (day between 0 and 4),
  slot int not null check (slot between 0 and 2),
  nome text not null,
  created_at timestamptz not null default now(),
  unique (week_key, day, slot, nome)
);

alter table iscrizioni enable row level security;

-- chiunque abbia il link puo' LEGGERE
create policy "lettura pubblica" on iscrizioni
  for select using (true);

-- chiunque abbia il link puo' ISCRIVERSI
create policy "inserimento pubblico" on iscrizioni
  for insert with check (true);

-- nessuna policy di update o delete:
-- con la chiave pubblica e' impossibile modificare o cancellare le iscrizioni altrui
```

3. Vai in **Project Settings → API** e copia:
   - **Project URL**
   - chiave **anon public**

## 2. Configurare l'app

Apri `index.html` e compila le prime righe dello `<script>`:

```js
const SUPABASE_URL = "https://xxxxx.supabase.co";
const SUPABASE_KEY = "eyJhbGciOi...";   // chiave anon public
const ADMIN_PIN    = "2027";            // cambialo
const ATLETI = ["Marco R.","Giulia B.", ...];   // elenco reale
```

La chiave anon e' pubblica per progettazione: protegge il database attraverso le policy
SQL del punto 1, non nascondendosi. Non inserire mai qui la chiave `service_role`.

### Rimozione iscrizioni (PIN preparatori)

Il pulsante "Area preparatori" chiede il PIN e mostra le ✕ accanto ai nomi.
Perche' la cancellazione funzioni davvero serve una policy aggiuntiva. Due strade:

- **Semplice**: aggiungi anche `create policy "cancellazione" on iscrizioni for delete using (true);`
  In questo modo pero' la cancellazione e' tecnicamente possibile a chiunque conosca
  l'indirizzo del database: il PIN protegge solo l'interfaccia.
- **Sicura**: lascia il database senza policy di delete e rimuovi le iscrizioni
  direttamente dalla **Table Editor** di Supabase quando serve.

## 3. Pubblicare su Vercel

1. Crea un repository GitHub con `index.html` e `README.md`.
2. Su [vercel.com](https://vercel.com): **Add New… → Project**, importa il repository,
   Framework Preset **Other**, nessun comando di build → **Deploy**.
3. Copia il link e mandalo agli atleti. Su iPhone: Safari → Condividi → *Aggiungi a Home*.

## 4. Collegare l'app dei preparatori

Nell'app dei preparatori apri `index.html` e incolla il link in:

```js
const PLUS_APP_URL = "https://corsi-plus-xxxx.vercel.app";
```

Nella scheda PLUS comparira' il pulsante "APRI LE ISCRIZIONI".

---

## Manutenzione

| Cosa | Dove in `index.html` |
|---|---|
| Elenco atleti | `ATLETI` |
| Posti per modulo | `POSTI` |
| Numero massimo riserve | `MAX_RISERVE` |
| Orari e rotazione moduli | `SLOTS` e `PLUS` |
| Lunedi' della settimana A | `DEFAULT_ANCHOR` |
| PIN preparatori | `ADMIN_PIN` |

Le iscrizioni restano archiviate settimana per settimana (`week_key`): lo storico
delle presenze ai Plus e' consultabile dalla Table Editor di Supabase.
