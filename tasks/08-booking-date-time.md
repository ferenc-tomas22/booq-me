# Task 08 — Booking Flow: Kalendár + Čas

**Modul:** 3 — Rezervačný flow
**Čas:** ~90–120 min
**Obtiažnosť:** ★★★★☆

## Čo sa naučíš

- Čo je **Server Action** a ako sa volá z client komponentu
- Ako pracovať s **dátumami a časovými zónami** (date-fns + date-fns-tz)
- Ako počítať **voľné time sloty** z working hours mínus rezervácie a blocked slots
- Ako použiť shadcn `Calendar` komponent
- Pattern `useTransition` pre pending state

> ⚠️ **Tento task je hustý.** Tri nové koncepty (Server Actions, useTransition,
> useEffect dependency gotcha) + time math + Calendar. Plánuj si ho na **dve sedenia**:
> najprv 1+2 (utility + business logic), potom 3+4 (server action + Calendar). Nevadí.

## Background

### Server Actions

Pred Next.js 14 keď si chcel mutovať dáta zo strany clienta, musel si:
1. Napísať API route (`/api/...`)
2. Z clienta volať `fetch('/api/...')`
3. Manuálne parseovať request, response, JSON, errory

**Server Actions** to skracujú: napíšeš funkciu s `'use server'` direktívou a importuješ
ju v client komponente. Next.js to pod kapotou prevedie na HTTP request.

```ts
// app/actions.ts
'use server';
export async function getAvailableSlots(date: Date, serviceId: string) {
  // Beží na serveri, vidí DB, env vars
  return [...];
}
```

```tsx
// app/picker.tsx
'use client';
import { getAvailableSlots } from './actions';

const handleDateChange = async (date: Date) => {
  const slots = await getAvailableSlots(date, serviceId);
  // ...
};
```

### Časové zóny — kritické

Toto je miesto, kde sa to zvyčajne pokazí v produkcii. Predstav si:

- **Tvoj laptop**: timezone `Europe/Bratislava` (CET = UTC+1 zima, CEST = UTC+2 leto)
- **Vercel server**: vždy `UTC`
- **Postgres**: ukladá `timestamp with timezone` ako UTC

Keď v dev móde napíšeš `new Date('2026-05-22T13:15:00')`, JavaScript to interpretuje ako
**lokálny čas** servera. Na tvojom laptope = 13:15 v Bratislave = 11:15 UTC. Na Vercele
= 13:15 UTC = 15:15 v Bratislave. **Posun o 2 hodiny.**

Preto: vždy buduj `Date` explicitne v `Europe/Bratislava` timezone, ukladaj UTC do DB,
formátuj späť v `Europe/Bratislava` na displaye.

Knižnica `date-fns-tz` ti to robí ľahké:
- `fromZonedTime(localDate, 'Europe/Bratislava')` → vráti `Date` v UTC (na uloženie do DB)
- `toZonedTime(utcDate, 'Europe/Bratislava')` → vráti `Date` posunutý do TZ (na zobrazenie)
- `formatInTimeZone(date, 'Europe/Bratislava', 'HH:mm')` → "13:15" string

### Ako rátame voľné sloty?

Logika:

1. Vezmi všetky 15-minútové sloty v daný deň medzi working hours (napr. 9:00–19:00
   **v Bratislavskej TZ**)
2. Pre každý slot skontroluj:
   - Či by sa služba zmestila (slot + duration ≤ koniec working hours)
   - Či sa neprekrýva s existujúcou rezerváciou
   - Či sa neprekrýva s blocked slotom
   - Či je v budúcnosti (nie minulý čas)
3. Vráť zoznam voľných slotov ako `"HH:mm"` stringy

## Tvoja úloha

### 1. Nainštaluj date-fns a shadcn Calendar

```bash
pnpm add date-fns date-fns-tz
pnpm dlx shadcn@latest add calendar
```

`shadcn calendar` automaticky pridá aj `react-day-picker` ako dependency.

### 2. Vytvor časový helper `src/lib/timezone.ts`

```ts
import { formatInTimeZone, fromZonedTime, toZonedTime } from 'date-fns-tz';

export const TZ = 'Europe/Bratislava';

/**
 * Z "2026-05-22" + "13:15" vytvori Date v UTC (na ulozenie do DB).
 * Cas je interpretovany v Europe/Bratislava.
 */
export const buildBratislavaDateTime = (dateStr: string, timeStr: string): Date => {
  // ISO string bez TZ → fromZonedTime to bere ako Bratislavsky cas → vrati Date v UTC.
  return fromZonedTime(`${dateStr}T${timeStr}:00`, TZ);
};

/**
 * Z Date (UTC) vrati "HH:mm" v Bratislavskej zone.
 */
export const formatBratislavaTime = (date: Date): string => {
  return formatInTimeZone(date, TZ, 'HH:mm');
};

/**
 * Z Date (UTC) vrati "dd.MM.yyyy HH:mm" v Bratislavskej zone.
 */
export const formatBratislavaDateTime = (date: Date): string => {
  return formatInTimeZone(date, TZ, 'dd.MM.yyyy HH:mm');
};

/**
 * Vrati zaciatok dna (00:00) v Bratislavskej zone, ako UTC Date.
 */
export const startOfDayBratislava = (dateStr: string): Date => {
  return fromZonedTime(`${dateStr}T00:00:00`, TZ);
};

/**
 * Vrati koniec dna (23:59:59.999) v Bratislavskej zone, ako UTC Date.
 */
export const endOfDayBratislava = (dateStr: string): Date => {
  return fromZonedTime(`${dateStr}T23:59:59.999`, TZ);
};

/**
 * Vrati den v tyzdni (0=nedela, 1=pondelok, ..., 6=sobota) pre dany Date v Bratislavskej zone.
 */
export const getDayOfWeekBratislava = (date: Date): number => {
  return toZonedTime(date, TZ).getDay();
};
```

### 3. Vytvor business logic `src/lib/booking-logic.ts`

```ts
import { addMinutes } from 'date-fns';
import {
  buildBratislavaDateTime,
  formatBratislavaTime,
  getDayOfWeekBratislava,
} from './timezone';

// Working hours sú zatiaľ hardcoded. V Tasku 12 ich Samuel bude vedieť meniť.
const WORKING_HOURS: Record<number, { start: string; end: string } | null> = {
  // 0 = nedeľa, 1 = pondelok, ..., 6 = sobota
  1: { start: '09:00', end: '19:00' },
  2: { start: '09:00', end: '19:00' },
  3: { start: '09:00', end: '19:00' },
  4: { start: '09:00', end: '19:00' },
  5: { start: '09:00', end: '19:00' },
  6: { start: '09:00', end: '14:00' },
  0: null, // nedeľa zatvorené
};

const SLOT_STEP_MINUTES = 15;

type BusySlot = {
  startTime: Date;
  endTime: Date;
};

/**
 * Vrati zoznam volnych slotov ("HH:mm") pre dany den.
 * Cely vypocet je v Bratislavskej zone, vsetky Date objekty (input aj busy) su UTC.
 */
export const computeAvailableSlots = (
  dateStr: string, // "YYYY-MM-DD"
  serviceDurationMinutes: number,
  busySlots: BusySlot[],
): string[] => {
  // Zisti den v tyzdni — vyrobime zaciatok dna v Bratislave a pozrieme.
  const dayStartUtc = buildBratislavaDateTime(dateStr, '00:00');
  const dayOfWeek = getDayOfWeekBratislava(dayStartUtc);
  const hours = WORKING_HOURS[dayOfWeek];

  if (!hours) return [];

  const openTimeUtc = buildBratislavaDateTime(dateStr, hours.start);
  const closeTimeUtc = buildBratislavaDateTime(dateStr, hours.end);
  const now = new Date();

  const available: string[] = [];
  let slot = openTimeUtc;

  while (slot < closeTimeUtc) {
    const slotEnd = addMinutes(slot, serviceDurationMinutes);

    if (slotEnd > closeTimeUtc) break;

    const overlaps = busySlots.some(
      (busy) => slot < busy.endTime && busy.startTime < slotEnd,
    );

    if (!overlaps && slot > now) {
      available.push(formatBratislavaTime(slot));
    }

    slot = addMinutes(slot, SLOT_STEP_MINUTES);
  }

  return available;
};
```

> 💡 **Prečo `slot < closeTimeUtc` a nie `isBefore`?**
> JavaScript `Date` objekty sa porovnávajú cez `<` `>` `<=` `>=` priamo (interne sú to
> numbery v milisekundách). `isBefore` z date-fns robí to isté ale s overheadom volania.

### 4. Vytvor Server Action `src/app/rezervacia/actions.ts`

```ts
'use server';

import { and, eq, gte, lt, lte, or } from 'drizzle-orm';
import { db } from '@/db';
import { bookings, blockedSlots, services } from '@/db/schema';
import { computeAvailableSlots } from '@/lib/booking-logic';
import { startOfDayBratislava, endOfDayBratislava } from '@/lib/timezone';

export const getAvailableSlots = async (
  serviceId: string,
  dateStr: string, // "YYYY-MM-DD"
): Promise<string[]> => {
  const [service] = await db.select().from(services).where(eq(services.id, serviceId));
  if (!service) return [];

  const dayStart = startOfDayBratislava(dateStr);
  const dayEnd = endOfDayBratislava(dateStr);

  // Existujuce aktivne rezervacie ktore zacinaju v dany den.
  const existingBookings = await db
    .select({ startTime: bookings.startTime, endTime: bookings.endTime })
    .from(bookings)
    .where(
      and(
        eq(bookings.canceled, false),
        gte(bookings.startTime, dayStart),
        lte(bookings.startTime, dayEnd),
      ),
    );

  // Blocked sloty ktore sa dotykaju daneho dna (zacinaju ALEBO koncia v ramci dna,
  // alebo cely obklopuju den).
  const blocks = await db
    .select({ startTime: blockedSlots.startTime, endTime: blockedSlots.endTime })
    .from(blockedSlots)
    .where(
      or(
        and(gte(blockedSlots.startTime, dayStart), lte(blockedSlots.startTime, dayEnd)),
        and(gte(blockedSlots.endTime, dayStart), lte(blockedSlots.endTime, dayEnd)),
        and(lt(blockedSlots.startTime, dayStart), gte(blockedSlots.endTime, dayEnd)),
      ),
    );

  return computeAvailableSlots(dateStr, service.durationMinutes, [...existingBookings, ...blocks]);
};
```

### 5. Vytvor `src/app/rezervacia/date-time-picker.tsx`

```tsx
'use client';

import { useEffect, useState, useTransition } from 'react';
import { useRouter, useSearchParams } from 'next/navigation';
import { format } from 'date-fns';
import { Calendar } from '@/components/ui/calendar';
import { Button } from '@/components/ui/button';
import { getAvailableSlots } from './actions';

type Props = {
  serviceId: string;
  selectedDate?: string;
  selectedTime?: string;
};

export const DateTimePicker = ({ serviceId, selectedDate, selectedTime }: Props) => {
  const router = useRouter();
  const searchParams = useSearchParams();
  const [isPending, startTransition] = useTransition();
  const [slots, setSlots] = useState<string[]>([]);

  // Pozor: `date` z `selectedDate` je novy Date object pri kazdom renderi (nova referencia).
  // Ak ho dame do useEffect dependency array, useEffect sa lupne donekonecna.
  // Riesenie: pouzijeme `selectedDate` STRING ako dependency, nie Date object.
  useEffect(() => {
    if (!selectedDate) {
      setSlots([]);
      return;
    }
    startTransition(async () => {
      const result = await getAvailableSlots(serviceId, selectedDate);
      setSlots(result);
    });
  }, [selectedDate, serviceId]);

  const date = selectedDate ? new Date(`${selectedDate}T12:00:00`) : undefined;

  const updateUrl = (key: 'date' | 'time', value: string) => {
    const params = new URLSearchParams(searchParams);
    params.set(key, value);
    if (key === 'date') params.delete('time');
    router.push(`?${params.toString()}`, { scroll: false });
  };

  return (
    <div className="grid md:grid-cols-2 gap-8">
      <Calendar
        mode="single"
        selected={date}
        onSelect={(d) => d && updateUrl('date', format(d, 'yyyy-MM-dd'))}
        disabled={(d) => d < new Date(new Date().setHours(0, 0, 0, 0))}
      />
      <div>
        <h3 className="font-medium mb-3">Voľné časy</h3>
        {!selectedDate && <p className="text-sm text-muted-foreground">Najprv vyber dátum.</p>}
        {selectedDate && isPending && <p className="text-sm text-muted-foreground">Načítavam...</p>}
        {selectedDate && !isPending && slots.length === 0 && (
          <p className="text-sm text-muted-foreground">V tento deň nie sú voľné termíny.</p>
        )}
        {selectedDate && !isPending && slots.length > 0 && (
          <div className="grid grid-cols-3 gap-2">
            {slots.map((time) => (
              <Button
                key={time}
                variant={selectedTime === time ? 'default' : 'outline'}
                size="sm"
                onClick={() => updateUrl('time', time)}
              >
                {time}
              </Button>
            ))}
          </div>
        )}
      </div>
    </div>
  );
};
```

> 💡 **`useEffect` dependency gotcha:**
> Keby sme dali `date` (Date object) do dependency array, React by ho porovnával cez `===`
> a `new Date(...)` v každom renderi je **nová referencia** → effect by sa volal donekonečna.
> Preto používame `selectedDate` ako string — strings sa porovnávajú podľa hodnoty.

> 💡 **Prečo `T12:00:00` pri konštrukcii `date`?**
> Ak by sme dali `new Date('2026-05-22')`, JavaScript by ho interpretoval ako **UTC** polnoc.
> Pre browser v `Europe/Bratislava` (UTC+1/+2) by to bolo "1:00/2:00 AM" — `getDay()` by
> mohol vrátiť predošlý deň. `T12:00:00` (poludnie) je vždy istý deň, bez ohľadu na TZ.

### 6. Aktualizuj `src/app/rezervacia/page.tsx`

```tsx
import { db } from '@/db';
import { services } from '@/db/schema';
import { eq } from 'drizzle-orm';
import { ServicePicker } from './service-picker';
import { DateTimePicker } from './date-time-picker';

type Props = {
  searchParams: Promise<{ serviceId?: string; date?: string; time?: string }>;
};

export default async function RezervaciaPage({ searchParams }: Props) {
  const params = await searchParams;
  const allServices = await db
    .select()
    .from(services)
    .where(eq(services.status, 'ACTIVE'));

  return (
    <div className="max-w-3xl mx-auto py-8 space-y-12">
      <h1 className="text-4xl font-bold text-center">Rezervácia termínu</h1>

      <section>
        <h2 className="text-2xl font-semibold mb-4">1. Vyberte si službu</h2>
        <ServicePicker services={allServices} selectedId={params.serviceId} />
      </section>

      {params.serviceId && (
        <section>
          <h2 className="text-2xl font-semibold mb-4">2. Vyberte dátum a čas</h2>
          <DateTimePicker
            serviceId={params.serviceId}
            selectedDate={params.date}
            selectedTime={params.time}
          />
        </section>
      )}

      {params.serviceId && params.date && params.time && (
        <section>
          <h2 className="text-2xl font-semibold mb-4">3. Vaše údaje</h2>
          <p className="text-muted-foreground">Formulár (Task 09).</p>
        </section>
      )}
    </div>
  );
}
```

### 7. Otestuj

```bash
pnpm dev
```

- Otvor `/rezervacia`
- Vyber službu → objaví sa kalendár
- Vyber dátum → objavia sa voľné časy
- Vyber čas → URL je `?serviceId=...&date=...&time=...`
- Skús dnes — staršie časy ako "teraz" by sa nemali zobraziť
- Skús nedeľu — žiadne sloty (zatvorené)
- Vlož ručne booking do DB cez Drizzle Studio na zajtra 10:00 — slot 10:00 a okolité slot by mali zmiznúť (lebo by sa s ním prekrývali)

### 8. Commit

```bash
git add .
git commit -m "task 08: booking flow step 2 - calendar with available slots, Europe/Bratislava TZ"
git push
```

## Acceptance Criteria

- [ ] Po vybere služby sa zobrazí kalendár + zoznam voľných časov
- [ ] Časy sa rátajú v **Europe/Bratislava** timezone (overiteľné: nastav OS TZ na inú,
  výsledok by mal byť rovnaký)
- [ ] Voľné časy rešpektujú working hours (Po–Pi 9–19, So 9–14, Ne zatvorené)
- [ ] Existujúce booking + blocked slot blokujú prislušné časy
- [ ] Časy v minulosti sa nezobrazujú pre dnešok
- [ ] Po vybere času sa URL aktualizuje a zobrazí sa Sekcia 3 (placeholder)
- [ ] Vybraný čas má vizuálne odlíšenie (button `variant="default"`)
- [ ] `useTransition` zobrazuje "Načítavam..." kým server action beží

## Bonus

- Rozdeľ sloty na "Doobeda" (do 12:00) a "Poobede" (od 12:00) ako booqme
- V kalendári označ červeným bodkom dni s voľnými slotmi (zatiaľ sú všetky dni klikateľné)
- Pridaj `locale={sk}` z `date-fns/locale` do `<Calendar>` — slovenské názvy mesiacov

## Tipy a riešenia problémov

**Problém:** Calendar zobrazí anglické názvy mesiacov
**Riešenie:** Pridaj `import { sk } from 'date-fns/locale'` a `<Calendar locale={sk} />`.
react-day-picker rešpektuje `locale` prop.

**Problém:** Server action vracia prázdny array, hoci by mali byť sloty
**Riešenie:** Daj `console.log` do server action — výstup uvidíš v termináli kde beží
`pnpm dev`, NIE v browser console. Server actions bežia na serveri.

**Problém:** `useEffect` sa volá donekonečna
**Riešenie:** Dependency array! `useEffect(..., [selectedDate])` (string) — NIE `[date]`
(Date object).

**Problém:** Slot je v zlej hodine (napr. máš 11:15 namiesto 13:15)
**Riešenie:** Timezone bug. Skontroluj že všade používaš `formatBratislavaTime`,
`buildBratislavaDateTime`, nie raw `Date()` / `getHours()`.

## Pýtanie sa Claude Code

- *"Vysvetli mi `useTransition` v Reacte. Čo robí `startTransition` a prečo ho používame
  s server action?"*
- *"Mám booking v DB s `startTime = 2026-05-22T11:15:00Z` (UTC). Helper to formátuje na
  `13:15` v Bratislave. Vysvetli mi krok po kroku, ako `formatInTimeZone` vie urobiť tento
  posun."*
- *"Môj algoritmus na rátanie slotov vracia zlé výsledky pre 75-minútovú službu — slot 18:00
  je voľný hoci by nemal byť (skončil by sa 19:15). Ukáž mi, kde je chyba: [paste kódu]"*

## Ďalší krok

➡️ [Task 09 — Booking flow: formulár + uloženie](09-booking-customer-form.md)
