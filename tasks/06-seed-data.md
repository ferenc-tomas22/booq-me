# Task 06 — Seed Data

**Modul:** 2 — Databáza
**Čas:** ~30 min
**Obtiažnosť:** ★★☆☆☆

## Čo sa naučíš

- Čo je **seed script** a kedy ho použiť
- Ako spustiť TypeScript skript bez kompilácie (`tsx`)
- Ako robiť **idempotent** seed (vieš ho spustiť 100× a nič sa nepokazí)

## Background

**Seed data** = počiatočné dáta, ktoré naplníš do prázdnej DB. Použiteľné pre:
- Lokálny vývoj — aby si nemusel ručne vytvárať testovacie dáta
- E2E testy — vždy presne tieto dáta
- Production prvý deploy — napríklad zoznam služieb, ktoré od štartu existujú

**Idempotent** = vieš to spustiť opakovane a výsledok je rovnaký. Pre nás to znamená: pred
insertom najprv zmažme existujúce dáta.

## Tvoja úloha

### 1. Nainštaluj `tsx` (TypeScript executor)

`tsx` ti dovolí spúšťať `.ts` súbory priamo, bez kompilácie do `.js`:

```bash
pnpm add -D tsx
```

### 2. Vytvor `src/db/seed.ts`

```ts
import { db } from './index';
import { services, bookings, blockedSlots } from './schema';

const SAMUEL_SERVICES = [
  {
    name: 'Pánsky strih',
    description: 'Strih nožnicami alebo strojčekom, umytie vlasov, styling, úprava obočia.',
    durationMinutes: 45,
    priceEur: 18,
  },
  {
    name: 'Úprava brady',
    description: 'Úprava brady strojčekom, britvou, hot towel.',
    durationMinutes: 30,
    priceEur: 15,
  },
  {
    name: 'Pánsky strih + úprava brady',
    description: 'Kompletná úprava: vlasy aj brada.',
    durationMinutes: 75,
    priceEur: 30,
  },
  {
    name: 'Detský strih',
    description: 'Strih nožnicami alebo strojčekom, umytie, styling. Pre chlapcov do 12 rokov.',
    durationMinutes: 35,
    priceEur: 18,
  },
  {
    name: 'Dlhé vlasy',
    description: 'Strih dlhých vlasov, umytie, styling.',
    durationMinutes: 75,
    priceEur: 21,
  },
];

const seed = async () => {
  console.log('🧹 Mažem existujúce dáta...');
  await db.delete(bookings);
  await db.delete(blockedSlots);
  await db.delete(services);

  console.log('🌱 Vkladám služby...');
  await db.insert(services).values(SAMUEL_SERVICES);

  console.log('✅ Seed dokončený.');
  process.exit(0);
};

seed().catch((err) => {
  console.error('❌ Seed zlyhal:', err);
  process.exit(1);
});
```

### 3. Pridaj seed skript do `package.json`

```json
{
  "scripts": {
    "db:seed": "tsx --env-file=.env.local src/db/seed.ts"
  }
}
```

### 4. Spusti seed

```bash
pnpm db:seed
```

Mal by si vidieť:
```
🧹 Mažem existujúce dáta...
🌱 Vkladám služby...
✅ Seed dokončený.
```

### 5. Over v Drizzle Studio

```bash
pnpm db:studio
```

Otvor [local.drizzle.studio](https://local.drizzle.studio) → tabuľka `services` → mal by si
vidieť 5 riadkov.

### 6. Skús to spustiť ešte raz

```bash
pnpm db:seed
```

Stále len 5 riadkov v `services`. **To je idempotent.** Ak by si neurobil `db.delete()` na
začiatku, mal by si 10 → 15 → 20 riadkov.

### 7. Commit

```bash
git add .
git commit -m "task 06: seed script with Samuel's services"
git push
```

## Acceptance Criteria

- [ ] `pnpm add -D tsx` prebehlo
- [ ] `src/db/seed.ts` existuje a obsahuje 5 služieb z booqme.sk Men's Hub
- [ ] `package.json` má skript `db:seed`
- [ ] `pnpm db:seed` prebehlo bez errorov
- [ ] V Drizzle Studio vidíš 5 služieb
- [ ] Druhé spustenie seed-u nezmnoží dáta (idempotent)

## Bonus

- Pridaj 1–2 fake rezervácie do seedu pre testing (napríklad zajtra 10:00, pozajtra 14:00)
- Pridaj 1 `blocked_slot` na nasledujúci víkend (dovolenka)

## Tipy a riešenia problémov

**Problém:** `Error: DATABASE_URL is not set` v seed-e
**Riešenie:** Skontroluj že máš `--env-file=.env.local` v `package.json` skripte. Bez toho
`tsx` nečíta env vars.

**Problém:** `column "price_eur" is of type numeric but expression is of type integer`
**Riešenie:** V Drizzle som spravil `$type<number>()`, takže môžeš dať `18` (integer). Ak by
ti to hádzalo, skús `String(18)` alebo upravi schemu.

**Problém:** Seed prebehne ale Drizzle Studio neukáže dáta
**Riešenie:** Drizzle Studio cachuje. Refresh-ni browser tab (Cmd+R / F5).

## Pýtanie sa Claude Code

- *"Aký je rozdiel medzi `tsx` (executor) a `tsx` (TypeScript JSX)? Sú to dve rôzne veci?"*
- *"Prečo používame `process.exit(0)` na konci seedu? Čo sa stane bez toho?"*
- *"Ako spravím seed, ktorý pridá rezervácie, ale len ak ešte neexistujú (skutočne idempotent
  bez delete)?"*

## Ďalší krok

➡️ [Task 07 — Booking flow: vyber služby](07-booking-service-picker.md)
