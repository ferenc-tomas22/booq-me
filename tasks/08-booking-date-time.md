# Task 08 — Booking Flow: Kalendár + Čas

**Modul:** 3 — Rezervačný flow
**Čas:** ~90 min
**Obtiažnosť:** ★★★★☆

## Čo sa naučíš

- Čo je **Server Action** a ako sa volá z client komponentu
- Ako pracovať s **date-fns** (sčítavanie hodín, formátovanie, porovnávanie)
- Ako počítať **voľné time sloty** z working hours mínus rezervácie a blocked slots
- Ako použiť shadcn `Calendar` komponent

## Background

### Server Actions

Pred Next.js 14 keď si chcel mutovať dáta zo strany clienta, musel si:
1. Napísať API route (`/api/...`)
2. Z clienta volať `fetch('/api/...')`
3. Manuálne parseovať request, response, JSON, errory

**Server Actions** to skracujú: napíšeš funkciu s `'use server'` direktívou a importuješ ju
v client komponente. Next.js to pod kapotou prevedie na HTTP request.

```ts
'use server';
export async function getAvailableSlots(date: Date, serviceId: string) {
  // Beží na serveri, vidí DB, env vars
  return [...];
}
```

```tsx
'use client';
import { getAvailableSlots } from './actions';

const handleDateChange = async (date: Date) => {
  const slots = await getAvailableSlots(date, serviceId);
  // ...
};
```

### Ako rátame voľné sloty?

Logika:

1. Vezmi všetky 15-minútové sloty v daný deň medzi working hours (napr. 9:00–19:00)
2. Pre každý slot skontroluj:
   - Či by sa služba zmestila (slot + duration ≤ koniec working hours)
   - Či sa neprekrýva s existujúcou rezerváciou
   - Či sa neprekrýva s blocked slotom
3. Vráť zoznam voľných slotov

## Tvoja úloha

### 1. Nainštaluj date-fns a shadcn Calendar

```bash
pnpm add date-fns date-fns-tz
pnpm dlx shadcn@latest add calendar
```

`shadcn calendar` automaticky pridá aj `react-day-picker` ako dependency.

### 2. Vytvor business logic súbor `src/lib/booking-logic.ts`

```ts
import { addMinutes, isAfter, isBefore, setHours, setMinutes, setSeconds } from 'date-fns';

// Working hours sú zatiaľ hardcoded. V tasku 12 ich Samuel bude vedieť meniť.
const WORKING_HOURS = {
  // 0 = nedeľa, 1 = pondelok, ..., 6 = sobota
  1: { start: 9, end: 19 },
  2: { start: 9, end: 19 },
  3: { start: 9, end: 19 },
  4: { start: 9, end: 19 },
  5: { start: 9, end: 19 },
  6: { start: 9, end: 14 },
  0: null, // nedeľa zatvorené
};

const SLOT_STEP_MINUTES = 15;

type BusySlot = {
  startTime: Date;
  endTime: Date;
};

export const computeAvailableSlots = (
  date: Date,
  serviceDurationMinutes: number,
  busySlots: BusySlot[],
): string[] => {
  const dayOfWeek = date.getDay() as keyof typeof WORKING_HOURS;
  const hours = WORKING_HOURS[dayOfWeek];

  if (!hours) return [];

  const dayStart = setSeconds(setMinutes(setHours(date, hours.start), 0), 0);
  const dayEnd = setSeconds(setMinutes(setHours(date, hours.end), 0), 0);

  const available: string[] = [];
  let slot = dayStart;

  while (isBefore(slot, dayEnd)) {
    const slotEnd = addMinutes(slot, serviceDurationMinutes);

    if (isAfter(slotEnd, dayEnd)) break;

    const overlaps = busySlots.some(
      (busy) => isBefore(slot, busy.endTime) && isBefore(busy.startTime, slotEnd),
    );

    if (!overlaps && isAfter(slot, new Date())) {
      const hh = slot.getHours().toString().padStart(2, '0');
      const mm = slot.getMinutes().toString().padStart(2, '0');
      available.push(`${hh}:${mm}`);
    }

    slot = addMinutes(slot, SLOT_STEP_MINUTES);
  }

  return available;
};
```

### 3. Vytvor Server Action `src/app/rezervacia/actions.ts`

```ts
'use server';

import { db } from '@/db';
import { bookings, services, blockedSlots } from '@/db/schema';
import { and, eq, gte, lte, or } from 'drizzle-orm';
import { startOfDay, endOfDay } from 'date-fns';
import { computeAvailableSlots } from '@/lib/booking-logic';

export const getAvailableSlots = async (
  serviceId: string,
  dateIso: string,
): Promise<string[]> => {
  const date = new Date(dateIso);
  const dayStart = startOfDay(date);
  const dayEnd = endOfDay(date);

  const [service] = await db.select().from(services).where(eq(services.id, serviceId));
  if (!service) return [];

  // Existujúce rezervácie v daný deň.
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

  // Blocked sloty, ktoré sa dotýkajú daného dňa.
  const blocks = await db
    .select({ startTime: blockedSlots.startTime, endTime: blockedSlots.endTime })
    .from(blockedSlots)
    .where(
      or(
        and(gte(blockedSlots.startTime, dayStart), lte(blockedSlots.startTime, dayEnd)),
        and(gte(blockedSlots.endTime, dayStart), lte(blockedSlots.endTime, dayEnd)),
      ),
    );

  return computeAvailableSlots(date, service.durationMinutes, [...existingBookings, ...blocks]);
};
```

### 4. Vytvor `src/app/rezervacia/date-time-picker.tsx`

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

  const date = selectedDate ? new Date(selectedDate) : undefined;

  useEffect(() => {
    if (!date) {
      setSlots([]);
      return;
    }
    startTransition(async () => {
      const result = await getAvailableSlots(serviceId, date.toISOString());
      setSlots(result);
    });
  }, [date?.toISOString(), serviceId]);

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
        {!date && <p className="text-sm text-muted-foreground">Najprv vyber dátum.</p>}
        {date && isPending && <p className="text-sm text-muted-foreground">Načítavam...</p>}
        {date && !isPending && slots.length === 0 && (
          <p className="text-sm text-muted-foreground">V tento deň nie sú voľné termíny.</p>
        )}
        {date && !isPending && slots.length > 0 && (
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

### 5. Aktualizuj `src/app/rezervacia/page.tsx`

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

### 6. Otestuj

```bash
pnpm dev
```

- Otvor `/rezervacia`
- Vyber službu → objaví sa kalendár
- Vyber dátum → objavia sa voľné časy
- Vyber čas → URL je `?serviceId=...&date=...&time=...`
- Skús dnes — staršie časy ako "teraz" by sa nemali zobraziť
- Skús nedeľu — žiadne sloty (zatvorené)
- Vlož ručne booking do DB cez Drizzle Studio na dnešok 10:00 — slot 10:00 by mal zmiznúť

### 7. Commit

```bash
git add .
git commit -m "task 08: booking flow step 2 - calendar with available slots via server action"
git push
```

## Acceptance Criteria

- [ ] Po vybere služby sa zobrazí kalendár + zoznam voľných časov
- [ ] Voľné časy sa rátajú správne (rešpektujú working hours, existujúce bookings, blocked slots)
- [ ] V nedeľu nie sú žiadne sloty (zatvorené)
- [ ] Časy v minulosti sa nezobrazujú pre dnešok
- [ ] Po vybere času sa URL aktualizuje a zobrazí sa Sekcia 3 (placeholder)
- [ ] Vybraný čas je vizuálne odlíšený (button variant="default")
- [ ] `useTransition` zobrazuje "Načítavam..." kým server action beží

## Bonus

- Rozdeľ sloty na "Doobeda" (do 12:00) a "Poobede" (od 12:00) ako booqme
- V kalendári označ červeným bodkom dni, kedy sú voľné sloty (zatiaľ sú všetky dni klikateľné)
- Pridaj timezone handling — vždy `Europe/Bratislava` (využi `date-fns-tz`)

## Tipy a riešenia problémov

**Problém:** Calendar komponent ukazuje anglické názvy dní
**Riešenie:** Pridaj `locale` z `date-fns/locale/sk`. shadcn Calendar zoberie locale z `react-day-picker` props.

**Problém:** Server action vracia prázdny array, hoci by mali byť sloty
**Riešenie:** Daj `console.log` do server action — výstup uvidíš v termináli kde beží `pnpm dev`,
NIE v browser console. Server actions bežia na serveri.

**Problém:** `useEffect` sa loopuje (neustále volá server action)
**Riešenie:** Závislosti! Použí `date?.toISOString()` ako dependency, NIE `date`. Pretože new
`Date` object pri každom renderi má inú referenciu.

**Problém:** Klik na disabled deň v kalendári nič nerobí ale console error
**Riešenie:** `react-day-picker` to handluje. Skontroluj že `disabled` prop má správnu logiku.

## Pýtanie sa Claude Code

- *"Vysvetli mi `useTransition` v Reacte. Čo robí `startTransition` a prečo ho používame?"*
- *"Ako fungujú časové zóny v JavaScripte? Mám DB v UTC, server v Európe, klient kdekoľvek —
  ako zaistím, že 13:15 je vždy 13:15 v Bratislave?"*
- *"Môj algoritmus na rátanie slotov vracia zlé výsledky pre 75-minútovú službu — slot 18:00
  je voľný hoci by nemal byť (skončil by sa 19:15). Ukáž mi, kde je chyba: [paste kódu]"*

## Ďalší krok

➡️ [Task 09 — Booking flow: formulár + uloženie](09-booking-customer-form.md)
