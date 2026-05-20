# Task 11 — Admin: Zoznam Rezervácií

**Modul:** 4 — Admin
**Čas:** ~45–60 min
**Obtiažnosť:** ★★★☆☆

## Čo sa naučíš

- Ako urobiť **DB joins** v Drizzle (booking + service v jednom query)
- Pattern **server component s filtrami v URL** (`?filter=today`)
- Ako použiť shadcn **Table** komponent
- Ako spraviť **cancel booking** action s revalidáciou

## Background

Admin chce vidieť:
- **Dnes** — čo má dnes pre Samuela urobiť (od **teraz** do konca dňa)
- **Tento týždeň** — výhľad
- **Všetky budúce** — celkový prehľad

Plus tlačidlo "Zrušiť" pri každej rezervácii.

Filter zachovávame v URL (`/admin?filter=today`) — rovnako ako booking flow, aby refresh
nestratil kontext.

> 💡 **Časy v admine** — všetky datetime v admine zobrazujeme v `Europe/Bratislava` cez
> helper z Tasku 08. Žiadne `format(date, 'HH:mm')` — to by ukázalo lokálny čas servera
> (UTC na Vercele).

## Tvoja úloha

### 1. Nainštaluj shadcn Table

```bash
pnpm dlx shadcn@latest add table badge alert-dialog
```

### 2. Vytvor server action `src/app/admin/actions.ts`

```ts
'use server';

import { eq } from 'drizzle-orm';
import { revalidatePath } from 'next/cache';
import { headers } from 'next/headers';
import { db } from '@/db';
import { bookings } from '@/db/schema';
import { auth } from '@/lib/auth';
import { ok, fail, type ActionResult } from '@/lib/action-result';

const requireAdmin = async () => {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) throw new Error('Unauthorized');
  return session.user;
};

export const cancelBooking = async (bookingId: string): Promise<ActionResult> => {
  try {
    await requireAdmin();
    await db
      .update(bookings)
      .set({ canceled: true })
      .where(eq(bookings.id, bookingId));
    revalidatePath('/admin');
    revalidatePath('/rezervacia'); // aby sa slot uvoľnil v page-level cache
    return ok(undefined);
  } catch (err) {
    return fail((err as Error).message);
  }
};
```

> 💡 **`requireAdmin`** je **guard** — kontrola na začiatku action. Bez nej by ktokoľvek
> mohol zavolať server action cez priame HTTP a zrušiť cudziu rezerváciu. Server actions
> **vždy** validujte auth na začiatku.

### 3. Vytvor `src/app/admin/page.tsx` — zoznam rezervácií

```tsx
import Link from 'next/link';
import { and, eq, gte, lte } from 'drizzle-orm';
import { endOfDay, addDays } from 'date-fns';
import { db } from '@/db';
import { bookings, services } from '@/db/schema';
import {
  Table, TableBody, TableCell, TableHead, TableHeader, TableRow,
} from '@/components/ui/table';
import { Badge } from '@/components/ui/badge';
import { formatBratislavaDateTime } from '@/lib/timezone';
import { formatPriceFromCents, formatDuration } from '@/lib/format';
import { CancelButton } from './cancel-button';

type Props = { searchParams: Promise<{ filter?: 'today' | 'week' | 'all' }> };

export default async function AdminBookingsPage({ searchParams }: Props) {
  const params = await searchParams;
  const filter = params.filter ?? 'today';

  // "Today" zahrnuje aj rezervacie ktore prave prebiehaju (endTime >= now), nie len budúce.
  // Samuel chce vidiet aj zákazníka, ktorého strihá práve teraz.
  const now = new Date();
  let toDate = endOfDay(now);

  if (filter === 'week') toDate = endOfDay(addDays(now, 7));
  if (filter === 'all') toDate = addDays(now, 365);

  const rows = await db
    .select({ booking: bookings, service: services })
    .from(bookings)
    .innerJoin(services, eq(bookings.serviceId, services.id))
    .where(and(gte(bookings.endTime, now), lte(bookings.startTime, toDate)))
    .orderBy(bookings.startTime);

  return (
    <div className="container mx-auto px-4 space-y-6">
      <div className="flex items-center justify-between">
        <h1 className="text-3xl font-bold">Rezervácie</h1>
        <div className="flex gap-2">
          <FilterLink current={filter} value="today" label="Dnes" />
          <FilterLink current={filter} value="week" label="Týždeň" />
          <FilterLink current={filter} value="all" label="Všetky" />
        </div>
      </div>

      {rows.length === 0 ? (
        <p className="text-muted-foreground text-center py-12">
          Žiadne nadchádzajúce rezervácie pre tento filter.
        </p>
      ) : (
        <Table>
          <TableHeader>
            <TableRow>
              <TableHead>Dátum a čas</TableHead>
              <TableHead>Služba</TableHead>
              <TableHead>Zákazník</TableHead>
              <TableHead>Kontakt</TableHead>
              <TableHead>Stav</TableHead>
              <TableHead></TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            {rows.map(({ booking, service }) => (
              <TableRow key={booking.id}>
                <TableCell className="font-medium">
                  {formatBratislavaDateTime(booking.startTime)}
                </TableCell>
                <TableCell>
                  {service.name}
                  <div className="text-xs text-muted-foreground">
                    {formatDuration(service.durationMinutes)} · {formatPriceFromCents(service.priceCents)}
                  </div>
                </TableCell>
                <TableCell>{booking.customerName}</TableCell>
                <TableCell>
                  <div className="text-sm">{booking.customerEmail}</div>
                  <div className="text-sm text-muted-foreground">{booking.customerPhone}</div>
                </TableCell>
                <TableCell>
                  {booking.canceled ? (
                    <Badge variant="destructive">Zrušená</Badge>
                  ) : (
                    <Badge>Aktívna</Badge>
                  )}
                </TableCell>
                <TableCell>
                  {!booking.canceled && <CancelButton bookingId={booking.id} />}
                </TableCell>
              </TableRow>
            ))}
          </TableBody>
        </Table>
      )}
    </div>
  );
}

const FilterLink = ({ current, value, label }: {
  current: string;
  value: string;
  label: string;
}) => (
  <Link
    href={`/admin?filter=${value}`}
    className={`px-3 py-1.5 text-sm rounded-md border ${
      current === value ? 'bg-primary text-primary-foreground' : ''
    }`}
  >
    {label}
  </Link>
);
```

### 4. Vytvor `src/app/admin/cancel-button.tsx`

```tsx
'use client';

import { useTransition } from 'react';
import { useRouter } from 'next/navigation';
import { toast } from 'sonner';
import { Button } from '@/components/ui/button';
import {
  AlertDialog, AlertDialogAction, AlertDialogCancel, AlertDialogContent,
  AlertDialogDescription, AlertDialogFooter, AlertDialogHeader, AlertDialogTitle,
  AlertDialogTrigger,
} from '@/components/ui/alert-dialog';
import { cancelBooking } from './actions';

export const CancelButton = ({ bookingId }: { bookingId: string }) => {
  const router = useRouter();
  const [isPending, startTransition] = useTransition();

  const handleConfirm = () => {
    startTransition(async () => {
      const result = await cancelBooking(bookingId);
      if (!result.ok) {
        toast.error(result.error);
        return;
      }
      toast.success('Rezervácia zrušená');
      router.refresh();
    });
  };

  return (
    <AlertDialog>
      <AlertDialogTrigger asChild>
        <Button variant="outline" size="sm" disabled={isPending}>
          Zrušiť
        </Button>
      </AlertDialogTrigger>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>Zrušiť rezerváciu?</AlertDialogTitle>
          <AlertDialogDescription>
            Túto akciu nie je možné vrátiť. Termín sa uvoľní pre nových zákazníkov pri ďalšom
            výbere dátumu v rezervačnom flow.
          </AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel>Späť</AlertDialogCancel>
          <AlertDialogAction onClick={handleConfirm}>Zrušiť rezerváciu</AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
};
```

> 💡 **Pozor na cache pre kalendár**: `revalidatePath('/rezervacia')` v server action
> invaliduje **page-level cache** stránky `/rezervacia`. Ale samotný kalendár vo flow
> fetchuje voľné sloty **per-date cez server action** z client componentu — to nie je
> page cache, takže sa neaktualizuje automaticky.
>
> Pre používateľa to znamená: ak má otvorenú stránku s vybraným dátumom a slot bol zrušený,
> uvidí ho ako "zaplnený" kým neprehadzne dátum (alebo neobnoví stránku). Pre MVP OK; pre
> lepší UX by sme pridali `revalidateTag` alebo SWR.

### 5. Otestuj

```bash
pnpm dev
```

- Prihlás sa do `/admin/login`
- Mal by si vidieť tabuľku rezervácií (alebo prázdny stav)
- Ak nemáš žiadne rezervácie, vytvor pár cez `/rezervacia` flow
- Skús filtre: Dnes / Týždeň / Všetky
- Skús zmeniť URL ručne na `?filter=all` — funguje
- Klik "Zrušiť" → confirmation dialog → potvrď → toast + tabuľka aktualizovaná
- Vrať sa do `/rezervacia` na ten istý slot (vyber tú istú službu a dátum) → slot by mal
  byť opäť voľný (page bola revalidatovaná)

### 6. Commit

```bash
git add .
git commit -m "task 11: admin bookings list with filters and cancel action"
git push
```

## Acceptance Criteria

- [ ] `/admin` zobrazuje tabuľku nadchádzajúcich rezervácií s filtrami (Dnes / Týždeň / Všetky)
- [ ] Filter `today` ukazuje len rezervácie od **teraz**, nie od polnoci
- [ ] Filter sa zachová v URL (`?filter=week`)
- [ ] Časy sú zobrazené v Bratislavskej zone (overiteľné: porovnaj s tým čo si pri rezervácii zadal)
- [ ] Každá rezervácia ma stav badge (Aktívna / Zrušená)
- [ ] Klik "Zrušiť" otvorí confirmation dialog
- [ ] Po zrušení sa toast objaví a tabuľka sa obnoví
- [ ] Otvorenie `/rezervacia` znova ponuká uvoľnený slot (po refreshi stránky)
- [ ] `cancelBooking` má auth guard (skús ho zavolať z incognito okna bez session — error)

## Bonus

- Pridaj search input (filter podľa mena/emailu)
- Pridaj sort by date (asc/desc)
- Klik na riadok otvorí detail rezervácie v dialog-u

## Tipy a riešenia problémov

**Problém:** `Table` komponent nie je responzívny (na mobile sa tlačí)
**Riešenie:** Obal table do `<div className="overflow-x-auto">`.

**Problém:** Po cancel sa tabuľka neaktualizuje
**Riešenie:** `router.refresh()` zopakuje server render. Bez neho zostane v cache.

**Problém:** Filter `today` ukáže prázdny zoznam, hoci viem že na zajtra mám booking
**Riešenie:** `today` filter má `toDate = endOfDay(now)` — to je dnešný koniec dňa. Zajtra
nepatrí do "dnes". Pozri filter `week` alebo `all`.

## Pýtanie sa Claude Code

- *"Vysvetli mi rozdiel medzi `revalidatePath('/admin')` v server action a `router.refresh()`
  na klientovi. Prečo používame oba?"*
- *"Ako spravím expandable row v shadcn Table (klik na riadok ukáže detaily)?"*
- *"Môj cancel button funguje, ale po zrušení vidím `Žiadne nadchádzajúce rezervácie` —
  prečo? Ako to opraviť?"*

## Ďalší krok

➡️ [Task 12 — Admin: služby + blokovanie termínov](12-admin-services-and-blocks.md)
