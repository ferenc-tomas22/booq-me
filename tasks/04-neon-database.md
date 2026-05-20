# Task 04 — Neon Postgres + Environment Variables

**Modul:** 2 — Databáza
**Čas:** ~30 min
**Obtiažnosť:** ★★☆☆☆

## Čo sa naučíš

- Čo je **serverless databáza** a prečo Neon
- Ako si vytvoríš projekt v Neone
- Čo je **connection string** a prečo NIKDY nesmie ísť do gitu
- Ako fungujú **environment variables** v Next.js (`.env.local`, `process.env`)

## Background

### Čo je Neon?

[Neon](https://neon.tech) je **serverless Postgres** — klasický PostgreSQL, ale s týmito vychytávkami:

- **Auto-suspend**: keď databáza nedostáva queries, automaticky sa "vypne" (0€ za compute)
- **Branchovateľné**: ako git! Vieš si urobiť branch DB pre testing
- **Free tier**: 0.5 GB storage + 1 compute hour denne — pre náš MVP úplne dosť

### Prečo nie SQLite?

SQLite je super pre lokálny vývoj, ale na Vercele nemáš filesystem (každý request beží
v inej "Lambda" funkcii). Potrebuješ **shared databázu** — niečo, k čomu sa pripojí každý
request rovnako. Neon je presne to.

### Čo je connection string?

URL v tvare:

```
postgres://user:password@host:port/dbname?sslmode=require
```

Je to ako URL pre webovú stránku, ale namiesto `https://` máme `postgres://`. Obsahuje **heslo
k tvojej databáze**, takže:

- ❌ **NIKDY** ho necommit do gitu
- ❌ **NIKDY** ho neuploaduj na Pastebin
- ❌ **NIKDY** ho nepriamy do JSX kódu

Patrí do **environment variables** — premenných, ktoré žijú mimo kódu.

## Tvoja úloha

### 1. Vytvor si účet v Neone

1. Choď na [neon.tech](https://neon.tech)
2. Klikni "Sign Up" → prihlás sa cez GitHub (najrýchlejšie)
3. Po prihlásení ťa Neon prevedie wizardom

### 2. Vytvor projekt

V Neon Console:

- **Project name**: `booq-me`
- **Postgres version**: latest (17)
- **Region**: `Europe (Frankfurt)` — najbližšie k Slovensku
- **Database name**: `booq_me` (default `neondb` je v pohode tiež)

Klikni "Create project".

### 3. Ulož si connection string

Po vytvorení ti Neon ukáže obrazovku s connection details. **Skopíruj si connection string**
(vyzerá ako `postgres://neondb_owner:abc123@ep-...neon.tech/neondb?sslmode=require`).

> 💡 **Branche**: Neon ti hneď vytvoril dve branche — `main` (production) a v Tasku 14 si pridáme
> ešte `dev`. Teraz použijeme `main`.

### 4. Vytvor `.env.local` v Next.js projekte

V root tvojej Next.js appky (`booq-me-app/`) vytvor súbor `.env.local`:

```bash
DATABASE_URL="postgres://neondb_owner:abc123@ep-...neon.tech/neondb?sslmode=require"
```

(Vlož svoj skutočný connection string z Neonu.)

> ⚠️ **DÔLEŽITÉ**: súbor sa volá `.env.local`, NIE `.env`. Next.js číta obe, ale `.env.local`
> je v defaultnom `.gitignore` — `.env` nie je.

### 5. Over `.gitignore`

Otvor `.gitignore` v rooti. Mal by tam byť riadok:

```
.env*.local
```

Ak nie, pridaj ho. **Toto je kritické** — ak commitneš connection string do gitu, hocikto si
môže pripojiť na tvoju DB.

> 🔍 **Skontroluj že to funguje:**
> ```bash
> git status
> ```
> Súbor `.env.local` sa NESMIE zobraziť v zozname.

### 6. Pridaj `.env.local.example`

Ostatní devs (alebo budúci ty) potrebujú vedieť, **aké env vars majú existovať**, ale nie ich
hodnoty. Vytvor `.env.local.example`:

```bash
DATABASE_URL="postgres://user:password@host/dbname?sslmode=require"
```

Tento súbor commitni do gitu — slúži ako šablóna.

### 7. Nainštaluj postgres driver

Drizzle (ktorý použijeme v Tasku 05) potrebuje pod kapotou driver:

```bash
pnpm add postgres
pnpm add -D drizzle-kit
pnpm add drizzle-orm
```

> 💡 **Prečo `postgres` (postgres.js)?** Je to moderný, rýchly Postgres driver pre Node.js,
> ktorý funguje skvele aj v serverless prostredí (Vercel). La-fly ho používa tiež.

### 8. Otestuj pripojenie

Vytvor súbor `src/db/index.ts`:

```ts
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';

if (!process.env.DATABASE_URL) {
  throw new Error('DATABASE_URL is not set');
}

const client = postgres(process.env.DATABASE_URL);
export const db = drizzle(client);
```

A quick test script `src/db/test-connection.ts`:

```ts
import { db } from './index';
import { sql } from 'drizzle-orm';

const test = async () => {
  const result = await db.execute(sql`SELECT NOW() as time`);
  console.log('✅ DB connected. Server time:', result);
  process.exit(0);
};

void test();
```

Spusti:

```bash
pnpm dlx tsx src/db/test-connection.ts
```

Ak vidíš `✅ DB connected. Server time: ...` — máš pripojenie.

Ak vidíš `Error: DATABASE_URL is not set` — `.env.local` nie je načítaný. `tsx` automaticky
nečíta `.env.local`. Riešenie:

```bash
pnpm dlx tsx --env-file=.env.local src/db/test-connection.ts
```

> 💡 **Prečo to v `next dev` funguje aj bez `--env-file`?** Next.js si automaticky načíta
> `.env.local`. Skripty mimo Next.js to musia robiť explicitne.

### 9. Vymaž test súbor

```bash
rm src/db/test-connection.ts
```

`src/db/index.ts` necháj — budeme ho používať od Tasku 05.

### 10. Commit

```bash
git add .
git commit -m "task 04: neon db setup, drizzle dependencies, .env scaffolding"
git push
```

## Acceptance Criteria

- [ ] Máš účet v Neone a projekt `booq-me`
- [ ] `.env.local` obsahuje `DATABASE_URL`
- [ ] `.env.local` **NIE JE** v gite (`git status` ho nevypisuje)
- [ ] `.env.local.example` JE v gite ako šablóna
- [ ] `pnpm add postgres drizzle-orm`, `pnpm add -D drizzle-kit` prebehlo
- [ ] `src/db/index.ts` existuje a exportuje `db`
- [ ] Test pripojenia prebehol s `✅ DB connected`

## Tipy a riešenia problémov

**Problém:** `Error: getaddrinfo ENOTFOUND ep-...neon.tech`
**Riešenie:** Skontroluj connection string — pravdepodobne máš preklep alebo chýba `https://`
nikde. Skopíruj ho znova z Neon Console.

**Problém:** `Error: connect ECONNREFUSED`
**Riešenie:** Neon má **suspend mode** — keď databáza dlho neaktívna, vypne sa. Prvý connect
ju "zobudí" za ~1 sekundu. Skús ešte raz.

**Problém:** `Error: password authentication failed for user`
**Riešenie:** Heslo v connection stringu je zlé. V Neon Console choď do `Settings → Reset
password` a vygeneruj nový string.

**Problém:** Vidím `.env.local` v `git status`!
**Riešenie:** Pridaj `.env*.local` do `.gitignore` HNEĎ. Ak si ho už commitol, treba ho vyňať
z gitu cez `git rm --cached .env.local`, commitnúť, a (ak si pushol na public repo)
**zmeniť heslo k DB** v Neone.

## Pýtanie sa Claude Code

- *"Vysvetli mi, prečo Next.js načítava `.env.local` automaticky, ale `tsx` nie. Ako sa to dá zariadiť?"*
- *"Aký je rozdiel medzi `process.env.X` na serveri a v browseri?"*
- *"Čo by sa stalo, keby som ten connection string nechtiac pushol na GitHub? Ako sa to dá zachrániť?"*

## Ďalší krok

➡️ [Task 05 — Drizzle schema](05-drizzle-schema.md)
