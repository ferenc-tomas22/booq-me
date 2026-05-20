# Task 05 — Drizzle Schema

**Modul:** 2 — Databáza
**Čas:** ~60 min
**Obtiažnosť:** ★★★☆☆

## Čo sa naučíš

- Ako sa modeluje databáza pre rezervačný systém
- Ako sa píše Drizzle schema (TypeScript = SQL)
- Čo je **migration** a prečo nestačí len upraviť schemu
- Ako spustiť **Drizzle Studio** — vizuálny prehliadač DB

## Background

### Prečo Drizzle?

Existujú dva veľké TypeScript ORM-y:

| | Prisma | Drizzle |
|---|---|---|
| Schema | `.prisma` DSL | TypeScript |
| Generuje client | Áno (`prisma generate`) | Nie (typy sa odvodzujú) |
| Edge runtime | Komplikované | Funguje |
| Veľkosť bundle | Veľká | Malá |
| Query API | Vyšší abstrakcia | Bližšie k SQL |

**Drizzle** ti dá pocit "píšeš SQL ale s typmi". To je super pre učenie — naozaj vidíš, čo sa
deje. Aj la-fly ho používa.

### Schema vs migrácia

- **Schema** = TypeScript súbor opisujúci, ako majú vyzerať tabuľky
- **Migration** = SQL súbor, ktorý mení existujúcu DB na nový tvar schémy

Keď upravíš schemu, **DB sa sama nezmení**. Musíš vygenerovať migráciu a aplikovať ju.

Drizzle má dva módy:
- **`drizzle-kit push`** — rýchly mode pre dev: porovná schemu s DB a urobí zmeny na mieste.
  **Bez migration history.**
- **`drizzle-kit generate` + `migrate`** — production mode: vygeneruje SQL súbor, ty ho
  reviewneš, a potom aplikuješ.

Pre dev a MVP úplne stačí `push`. Pre production v Tasku 14 prejdeme na `migrate`.

## Design — aké tabuľky potrebujeme?

Rozmýšľame zo strán flow-u v booqme:

1. **services** — Samuelove služby (pánsky strih, brada, ...)
2. **bookings** — rezervácie zákazníkov
3. **blocked_slots** — kedy Samuel **nepracuje** (dovolenka, výjazd)

Multi-staff (Robert + Samuel) zatiaľ nepotrebujeme — máme len Samuela. Multi-staff by sme
pridali ako tabuľku `staff` s FK z `bookings` a `blocked_slots`.

User accounts zákazníkov tiež nepotrebujeme — guest checkout, kontakt cez email/telefón.

## Tvoja úloha

### 1. Vytvor `drizzle.config.ts` v rooti

```ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './src/db/schema.ts',
  out: './src/db/migrations',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
});
```

### 2. Vytvor schemu `src/db/schema.ts`

```ts
import { relations } from 'drizzle-orm';
import {
  boolean,
  integer,
  numeric,
  pgEnum,
  pgTable,
  text,
  timestamp,
  unique,
} from 'drizzle-orm/pg-core';

// ───── Enum: status ─────
export const statusEnum = pgEnum('status', ['ACTIVE', 'INACTIVE']);

// ───── Services ─────
export const services = pgTable('services', {
  id: text('id').primaryKey().$defaultFn(() => crypto.randomUUID()),
  name: text('name').notNull(),
  description: text('description'),
  durationMinutes: integer('duration_minutes').notNull(),
  priceEur: numeric('price_eur', { precision: 10, scale: 2 })
    .$type<number>()
    .notNull(),
  status: statusEnum('status').notNull().default('ACTIVE'),
  createdAt: timestamp('created_at').notNull().defaultNow(),
});

// ───── Bookings ─────
export const bookings = pgTable('bookings', {
  id: text('id').primaryKey().$defaultFn(() => crypto.randomUUID()),
  serviceId: text('service_id').notNull().references(() => services.id),
  // Kedy začína a končí — endTime ukladáme aby sme nemuseli rátať dlžku pri každom query.
  startTime: timestamp('start_time').notNull(),
  endTime: timestamp('end_time').notNull(),
  // Kontakt na zákazníka.
  customerName: text('customer_name').notNull(),
  customerEmail: text('customer_email').notNull(),
  customerPhone: text('customer_phone').notNull(),
  note: text('note'),
  canceled: boolean('canceled').notNull().default(false),
  createdAt: timestamp('created_at').notNull().defaultNow(),
});

// ───── Blocked slots (dovolenka, výjazd) ─────
export const blockedSlots = pgTable('blocked_slots', {
  id: text('id').primaryKey().$defaultFn(() => crypto.randomUUID()),
  startTime: timestamp('start_time').notNull(),
  endTime: timestamp('end_time').notNull(),
  reason: text('reason'),
  createdAt: timestamp('created_at').notNull().defaultNow(),
});

// ───── Relations ─────
export const bookingsRelations = relations(bookings, ({ one }) => ({
  service: one(services, {
    fields: [bookings.serviceId],
    references: [services.id],
  }),
}));

// ───── Type exports (na využitie v server actions a components) ─────
export type Service = typeof services.$inferSelect;
export type NewService = typeof services.$inferInsert;
export type Booking = typeof bookings.$inferSelect;
export type NewBooking = typeof bookings.$inferInsert;
export type BlockedSlot = typeof blockedSlots.$inferSelect;
```

> 💡 **Prečo `text('id')` a nie `serial`?**
> `serial` = auto-increment integer (1, 2, 3, ...). To je predvídateľné a ľahké guessovať
> v URL. UUID-y sú nepredvídateľné, takže nevieš guessnúť, koľko rezervácií Samuel má.
> Drobnosť, ale dobrý zvyk.

> 💡 **Prečo `numeric` pre cenu?**
> `float`/`real` má floating-point chyby (0.1 + 0.2 ≠ 0.3). Pre peniaze používaj vždy fixed
> decimal — `numeric(10, 2)`.

### 3. Aktualizuj `src/db/index.ts` aby vedel o schéme

```ts
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';
import * as schema from './schema';

if (!process.env.DATABASE_URL) {
  throw new Error('DATABASE_URL is not set');
}

const client = postgres(process.env.DATABASE_URL);
export const db = drizzle(client, { schema });
```

### 4. Pridaj package.json skripty

V `package.json` pridaj do `scripts`:

```json
{
  "scripts": {
    "db:push": "drizzle-kit push",
    "db:studio": "drizzle-kit studio",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate"
  }
}
```

### 5. Pushni schemu do Neonu

```bash
pnpm db:push
```

Drizzle ti ukáže plán (`CREATE TABLE services...`, `CREATE TABLE bookings...`) a opýta sa
**"Apply changes?"** — odpovedz `y`.

### 6. Otvor Drizzle Studio

```bash
pnpm db:studio
```

Otvor [https://local.drizzle.studio](https://local.drizzle.studio) v prehliadači. Mal by si
vidieť 3 prázdne tabuľky.

> 💡 **Drizzle Studio** je tvoj najlepší kamarát počas vývoja. Vieš si prezerať dáta, ručne
> niečo vložiť, spúšťať queries. Otvor ho a nechaj otvorený v ďalšom tabe.

### 7. Commit

```bash
git add .
git commit -m "task 05: drizzle schema with services, bookings, blocked_slots"
git push
```

## Acceptance Criteria

- [ ] `drizzle.config.ts` existuje v rooti
- [ ] `src/db/schema.ts` definuje 3 tabuľky: `services`, `bookings`, `blocked_slots`
- [ ] Schema exportuje TypeScript typy (`Service`, `Booking`, `BlockedSlot`)
- [ ] `package.json` má skripty `db:push`, `db:studio`, `db:generate`, `db:migrate`
- [ ] `pnpm db:push` prebehlo bez errorov
- [ ] V Drizzle Studio vidíš 3 prázdne tabuľky
- [ ] V Neon Console (SQL Editor) vieš spustiť `SELECT * FROM services;` a dostaneš prázdny výsledok

## Tipy a riešenia problémov

**Problém:** `pnpm db:push` hodí `Error: DATABASE_URL is not set`
**Riešenie:** `drizzle-kit` automaticky **nečíta** `.env.local`. Drizzle ti to vyrieši, ak
máš inštalované `drizzle-kit@latest` (od v0.20+ číta `.env*` súbory). Skontroluj verziu:
`pnpm list drizzle-kit`. Ak je staršia ako 0.20, updatuj: `pnpm add -D drizzle-kit@latest`.

**Problém:** `relation "services" already exists`
**Riešenie:** Pravdepodobne si už raz pushol a teraz prepushuvaš s nezmenenou schémou. To je
v poriadku — Drizzle si nájde že nič netreba meniť. Ak hádže, choď do Neon Console → SQL
Editor a spusti `DROP TABLE services CASCADE; DROP TABLE bookings CASCADE; DROP TABLE
blocked_slots CASCADE;`, potom `pnpm db:push` znova.

**Problém:** Drizzle Studio sa neotvorí v prehliadači
**Riešenie:** Skontroluj že príkaz `pnpm db:studio` stále beží v termináli. Studio je local
HTTPS server, takže prehliadač môže pýtať povolenie pre self-signed certificate — accept it.

## Pýtanie sa Claude Code

- *"Prečo dávame `endTime` do `bookings`, keď by sme ho mohli rátať ako `startTime +
  service.durationMinutes`? Aké sú trade-offy?"*
- *"Ako by som rozšíril schemu pre multi-staff (Samuel + Robert)? Akú tabuľku pridať a aké
  FK upraviť?"*
- *"Aký je rozdiel medzi `references()` a explicit foreign key constraint v Postgres? Drizzle
  to robí ako?"*

## Ďalší krok

➡️ [Task 06 — Seed data](06-seed-data.md)
