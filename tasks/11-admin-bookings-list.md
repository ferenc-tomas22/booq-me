# Task 11 — Admin: Zoznam Rezervácií

**Modul:** 4 — Admin
**Čas:** ~45 min
**Obtiažnosť:** ★★★☆☆

## Čo sa naučíš

- Ako urobiť **DB joins** v Drizzle (booking + service v jednom query)
- Pattern **server component s filtrami v URL** (`?filter=today`)
- Ako použiť shadcn **Table** komponent
- Ako spraviť **cancel booking** action s revalidáciou

## Background

Admin chce vidieť:
- **Dnes** — čo má dnes pre Samuela urobiť
- **Tento týždeň** — výhľad
- **Všetky budúce** — celkový prehľad

Plus tlačidlo "Zrušiť" pri každej rezervácii.

Filter zachovávame v URL (`/admin?filter=today`) — rovnako ako booking flow, aby refresh
nestratil kontext.

## Tvoja úloha

### 1. Nainštaluj shadcn Table

```bash
pnpm dlx shadcn@latest add table badge alert-dialog
```

### 2. Vytvor server action `src/app/admin/actions.ts`

```ts
'use server';

import { db } from '@/db';
import { bookings } from '@/db/schema';
import { eq } from 'drizzle-orm';
import { revalidatePath } from 'next/cache';
import { ok, fail, type ActionResult } from '@/lib/action-result';
import { auth } from '@/lib/auth';
import { headers } from 'next/headers';

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
    revalidatePath('/rezervacia'); // aby sa slot uvoľnil pre nových
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
import { db } from '@/db';
import { bookings, services } from '@/db/schema';
import { and, eq, gte, lte } from 'drizzle-orm';
import { startOfDay, endOfDay, addDays, format } from 'date-fns';
import {
  Table, TableBody, TableCell, TableHead, TableHeader, TableRow,
} from '@/components/ui/table';
import { Badge } from '@/components/ui/badge';
import Link from 'next/link';
import { CancelButton } from './cancel-button';

type Props = { searchParams: Promise<{ filter?: 'today' | 'week' | 'all' }> };

export default async function AdminBookingsPage({ searchParams }: Props) {
  const params = await searchParams;
  const filter = params.filter ?? 'today';

  const now = new Date();
  let fromDate = startOfDay(now);
  let toDate = endOfDay(now);

  if (filter === 'week') toDate = endOfDay(addDays(now, 7));
  if (filter === 'all') toDate = addDays(now, 365);

  const rows = await db
    .select({ booking: bookings, service: services })
    .from(bookings)
    .innerJoin(services, eq(bookings.serviceId, services.id))
    .where(and(gte(bookings.startTime, fromDate), lte(bookings.startTime, toDate)))
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
          Žiadne rezervácie pre tento filter.
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
                  {format(booking.startTime, 'dd.MM.yyyy HH:mm')}
                </TableCell>
                <TableCell>
                  {service.name}
                  <div className="text-xs text-muted-foreground">
                    {service.durationMinutes} min · {service.priceEur} €
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
            Túto akciu nie je možné vrátiť. Slot sa uvoľní pre ďalších zákazníkov.
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

### 5. Otestuj

```bash
pnpm dev
```

- Prihlás sa do `/admin/login`
- Mal by si vidieť tabuľku rezervácií (alebo prázdny stav)
- Ak nemáš žiadne rezervácie, vytvor pár cez `/rezervacia` flow
- Skús filtre: Dnes / Týždeň / Všetky
- Klik "Zrušiť" → confirmation dialog → potvrď → toast + tabuľka aktualizovaná
- Skontroluj v `/rezervacia` že zrušený slot je opäť voľný

### 6. Commit

```bash
git add .
git commit -m "task 11: admin bookings list with filters and cancel action"
git push
```

## Acceptance Criteria

- [ ] `/admin` zobrazuje tabuľku rezervácií s filtrami (Dnes / Týždeň / Všetky)
- [ ] Filter sa zachová v URL (`?filter=week`)
- [ ] Každá rezervácia ma stav badge (Aktívna / Zrušená)
- [ ] Klik "Zrušiť" otvorí confirmation dialog
- [ ] Po zrušení sa toast objaví a tabuľka sa obnoví
- [ ] Zrušená rezervácia uvoľní svoj slot v `/rezervacia` flow
- [ ] `cancelBooking` má auth guard (skús ho zavolať z incognito okna bez session — error)

## Bonus

- Pridaj search input (filter podľa mena/emailu)
- Pridaj sort by date (asc/desc)
- Klik na riadok otvorí detail rezervácie v dialog-u

## Tipy a riešenia problémov

**Problém:** `Table` komponent nie je responzívny (na mobile sa tlačí)
**Riešenie:** Obal table do `<div className="overflow-x-auto">`. Alebo na mobile zobraz Card layout.

**Problém:** Po cancel sa tabuľka neaktualizuje
**Riešenie:** `router.refresh()` zopakuje server render. Bez neho zostane v cache. Alternatíva:
`revalidatePath('/admin')` v server action (mám tam to) — Next.js to refreshne automaticky pri ďalšom navigation, ale `router.refresh()` to spraví hneď.

**Problém:** `BigInt` error pri join-e
**Riešenie:** Nepoužívaš `BigInt`, takže by to nemalo vznikať. Ak hej, skontroluj že
`priceEur` má `$type<number>()` v schéme.

## Pýtanie sa Claude Code

- *"Vysvetli mi rozdiel medzi `revalidatePath('/admin')` v server action a `router.refresh()`
  na klientovi. Kedy ktoré použiť?"*
- *"Môj filter `today` ukáže aj rezervácie v minulosti (ráno). Ako to zmeniť aby ukazoval len
  budúce rezervácie?"*
- *"Ako spravím expandable row v shadcn Table (klik na riadok ukáže detaily)?"*

## Ďalší krok

➡️ [Task 12 — Admin: služby + blokovanie termínov](12-admin-services-and-blocks.md)
