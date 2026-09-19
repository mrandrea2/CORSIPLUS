# Corsi Plus · CT Scaligero

Web app di iscrizione ai corsi Plus per gli agonisti. Nessun login: l'atleta sceglie
il proprio nome da un elenco e si iscrive alle sessioni della settimana in corso.

## Regole

- Ci si iscrive **solo per la settimana in corso**.
- Posti titolari per modulo: Forza 5, Velocita' 8, Resistenza 8, Calcettone 10 (modificabili).
- A posti esauriti ci si mette in **riserva**: fino a 10, in ordine di iscrizione.
- **Nessuno puo' sovrascrivere l'iscrizione di un altro.**
- **Annullare un'iscrizione sbagliata**: l'atleta puo' farlo da solo entro 15 minuti
  (pulsante *annulla* accanto al proprio nome). Dopo, serve un preparatore.
- **Preparatori**: pulsante "Area preparatori", PIN, e compaiono le ✕ accanto a ogni nome.
  Tutto dall'app, senza mai entrare in Supabase.
- Titolari e riserve non sono salvati come tali: si calcolano leggendo le iscrizioni in ordine
  cronologico, quindi due iscrizioni simultanee non entrano mai in conflitto.

---

## 1. Creare il database su Supabase

1. Vai su [supabase.com](https://supabase.com) e crea un progetto (piano gratuito).
2. Apri **SQL Editor**, incolla tutto questo blocco e premi **Run**.
   Cambia il PIN nella riga indicata prima di eseguire.

```sql
-- TABELLA
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

-- chiunque abbia il link puo' LEGGERE e ISCRIVERSI
create policy "lettura pubblica"     on iscrizioni for select using (true);
create policy "inserimento pubblico" on iscrizioni for insert with check (true);

-- NESSUNA policy di update o delete:
-- con la chiave pubblica e' impossibile modificare o cancellare le iscrizioni.
-- Le due funzioni qui sotto sono l'unica via di cancellazione.

-- FUNZIONE 1 — l'atleta annulla la PROPRIA iscrizione entro 15 minuti
create or replace function annulla_iscrizione(p_id uuid, p_nome text)
returns boolean language plpgsql security definer as $$
declare n int;
begin
  delete from iscrizioni
   where id = p_id
     and nome = p_nome
     and created_at > now() - interval '15 minutes';
  get diagnostics n = row_count;
  return n > 0;
end; $$;

-- FUNZIONE 2 — il preparatore rimuove qualsiasi iscrizione con il PIN
create or replace function elimina_iscrizione(p_id uuid, p_pin text)
returns boolean language plpgsql security definer as $$
begin
  if p_pin <> '2027' then          -- <<< CAMBIA IL PIN QUI
    raise exception 'PIN errato';
  end if;
  delete from iscrizioni where id = p_id;
  return true;
end; $$;

grant execute on function annulla_iscrizione(uuid, text) to anon;
grant execute on function elimina_iscrizione(uuid, text) to anon;
```

3. Vai in **Project Settings → API** e copia **Project URL** e chiave **anon public**.

## 2. Configurare l'app

Apri `index.html` e compila le prime righe dello `<script>`:

```js
const SUPABASE_URL = "https://xxxxx.supabase.co";
const SUPABASE_KEY = "eyJhbGciOi...";   // chiave anon public
const ADMIN_PIN    = "2027";            // deve essere UGUALE a quello scritto nella funzione SQL
```

Il PIN sta in due posti e devono coincidere: nell'app e dentro `elimina_iscrizione`.
Il controllo vero avviene nel database, quindi il PIN protegge davvero, non solo l'interfaccia.
Usane uno di almeno 6 cifre. Non inserire mai qui la chiave `service_role`.

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
| Minuti per l'auto-annullamento | `MINUTI_ANNULLO` |
| Orari e rotazione moduli | `SLOTS` e `PLUS` |
| Lunedi' della settimana A | `DEFAULT_ANCHOR` |
| PIN preparatori | `ADMIN_PIN` (+ funzione SQL) |

Per aggiungere atleti basta scrivere un nome in piu' nell'elenco `ATLETI` e salvare su GitHub:
Vercel ripubblica da solo in pochi secondi.

Le iscrizioni restano archiviate settimana per settimana (`week_key`): lo storico delle
presenze ai Plus e' sempre consultabile dalla Table Editor di Supabase.
