# Task 09 — Booking Flow: Formulár + Uloženie

**Modul:** 3 — Rezervačný flow
**Čas:** ~60–90 min
**Obtiažnosť:** ★★★☆☆

## Čo sa naučíš

- Ako validovať form input cez **Zod**
- Ako používať **react-hook-form** so Zod resolverom
- Pattern **ActionResult** (`{ ok: true, data }` / `{ ok: false, error }`)
- Ako zabrániť **race condition** pri vytváraní rezervácie (dvaja na ten istý slot)
- Ako redirectnúť po úspešnom submite

## Background

### Prečo validovať na serveri AJ na klientovi?

- **Klient** = pekná UX (red border, hláška "Email musí mať @")
- **Server** = bezpečnosť. Klientovi sa nedá nikdy úveriť. Niekto môže poslať POST request
  mimo formulár (cez curl, Postman) a obísť všetky JS validácie.

**Zod** ti dovolí napísať schemu **raz** a použiť ju aj na klientovi aj na serveri.

### ActionResult pattern

Server actions môžu **hodiť exception** alebo **vrátiť error**. Druhá varianta je čistejšia:

```ts
type ActionResult<T> =
  | { ok: true; data: T }
  | { ok: false; error: string };
```

Klient skontroluje `result.ok` a zobrazí dáta alebo error message. Žiadne `try/catch` chaos.

### Race condition pri rezervácii

Predstav si: dvaja zákazníci kliknú "Vytvoriť rezerváciu" pre rovnaký termín v rovnaký
moment. Bez ochrany by sa obaja zapísali do DB → dvojitá rezervácia → krízová situácia
pre Samuela.

Ochrana: **pred insertom** sa DB query opýta "je tento termín stále voľný?". Nie je to
100% bezpečné (presný interleaving by stále mohol zlyhať), ale pre MVP úplne stačí. Pre
100% bezpečnosť by sme použili `SELECT ... FOR UPDATE` v transakcii.

## Tvoja úloha

### 1. Pridaj dependencies

```bash
pnpm add zod react-hook-form @hookform/resolvers
pnpm dlx shadcn@latest add form textarea sonner
```

`shadcn form` ti pridá `Form`, `FormField`, `FormItem`, `FormControl`, `FormMessage`
komponenty — wrappy nad `react-hook-form`.

`shadcn sonner` ti dá toast notifikácie.

### 2. Pridaj `<Toaster />` do root layoutu

V `src/app/layout.tsx`:

```tsx
import { Toaster } from '@/components/ui/sonner';

// ... v body, na konci
<Toaster richColors />
```

### 3. Vytvor Zod schemu `src/lib/validation.ts`

```ts
import { z } from 'zod';

export const bookingFormSchema = z.object({
  customerName: z
    .string()
    .min(2, 'Meno musí mať aspoň 2 znaky')
    .max(100, 'Meno môže mať max 100 znakov'),
  customerEmail: z.string().email('Neplatný email'),
  customerPhone: z
    .string()
    .min(9, 'Telefón musí mať aspoň 9 znakov')
    .regex(/^[+\d\s]+$/, 'Telefón môže obsahovať len číslice, medzery a +'),
  // Poznamka je optional na klientovi, ale ulozime ju ako null do DB (nie undefined).
  note: z
    .string()
    .max(500, 'Poznámka môže mať max 500 znakov')
    .optional()
    .transform((v) => v?.trim() || null),
});

export type BookingFormData = z.infer<typeof bookingFormSchema>;
```

### 4. Vytvor ActionResult helper `src/lib/action-result.ts`

```ts
export type ActionResult<T = void> =
  | { ok: true; data: T }
  | { ok: false; error: string };

export const ok = <T>(data: T): ActionResult<T> => ({ ok: true, data });
export const fail = <T = never>(error: string): ActionResult<T> => ({ ok: false, error });
```

### 5. Pridaj `createBooking` server action

Otvor `src/app/rezervacia/actions.ts` a pridaj:

```ts
import { addMinutes } from 'date-fns';
import { revalidatePath } from 'next/cache';
import { and, eq, lt, gt } from 'drizzle-orm';
import { bookingFormSchema } from '@/lib/validation';
import { ok, fail, type ActionResult } from '@/lib/action-result';
import { buildBratislavaDateTime } from '@/lib/timezone';

// Vstup pre createBooking — note pripustame ako string aj null aj undefined,
// Zod transform si to znormalizuje (trim, prazdne -> null).
type CreateBookingInput = {
  serviceId: string;
  date: string; // YYYY-MM-DD
  time: string; // HH:mm
  customerName: string;
  customerEmail: string;
  customerPhone: string;
  note?: string | null;
};

export const createBooking = async (
  input: CreateBookingInput,
): Promise<ActionResult<{ bookingId: string }>> => {
  // 1. Validate input cez Zod (na serveri!)
  const parsed = bookingFormSchema.safeParse(input);
  if (!parsed.success) {
    return fail(parsed.error.issues[0]?.message ?? 'Neplatné údaje');
  }

  // 2. Skontroluj že služba existuje
  const [service] = await db.select().from(services).where(eq(services.id, input.serviceId));
  if (!service) return fail('Služba neexistuje');

  // 3. Zostav startTime a endTime v Europe/Bratislava timezone, ulozenie v UTC.
  const startTime = buildBratislavaDateTime(input.date, input.time);
  const endTime = addMinutes(startTime, service.durationMinutes);

  // 4. Race-condition guard: pozrime priamo do DB, ci sa nikde neprekryva s aktivnym bookingom.
  //    Dva intervaly sa prekryvaju, ak: a.start < b.end AND b.start < a.end.
  const [conflict] = await db
    .select({ id: bookings.id })
    .from(bookings)
    .where(
      and(
        eq(bookings.canceled, false),
        lt(bookings.startTime, endTime),
        gt(bookings.endTime, startTime),
      ),
    )
    .limit(1);

  if (conflict) {
    return fail('Tento termín už nie je voľný. Vyberte iný.');
  }

  // 5. Vlož booking
  const [created] = await db
    .insert(bookings)
    .values({
      serviceId: input.serviceId,
      startTime,
      endTime,
      customerName: parsed.data.customerName,
      customerEmail: parsed.data.customerEmail,
      customerPhone: parsed.data.customerPhone,
      note: parsed.data.note,
    })
    .returning({ id: bookings.id });

  if (!created) return fail('Nepodarilo sa vytvoriť rezerváciu');

  // 6. Invalidate kalendar cache aby dalsi user nevidel tento slot
  revalidatePath('/rezervacia');

  return ok({ bookingId: created.id });
};
```

> 💡 **Race condition guard cez DB query** (krok 4) — toto je správnejšie než znova
> rátať všetky sloty pre celý deň. Kontrolujeme presný presah s konkrétnym intervalom
> `[startTime, endTime)`. Rýchle, presné.

### 6. Vytvor `src/app/rezervacia/customer-form.tsx`

```tsx
'use client';

import { useTransition } from 'react';
import { useRouter } from 'next/navigation';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { toast } from 'sonner';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Textarea } from '@/components/ui/textarea';
import {
  Form, FormControl, FormField, FormItem, FormLabel, FormMessage,
} from '@/components/ui/form';
import { bookingFormSchema, type BookingFormData } from '@/lib/validation';
import { createBooking } from './actions';

type Props = {
  serviceId: string;
  date: string;
  time: string;
};

export const CustomerForm = ({ serviceId, date, time }: Props) => {
  const router = useRouter();
  const [isPending, startTransition] = useTransition();

  const form = useForm<BookingFormData>({
    resolver: zodResolver(bookingFormSchema),
    defaultValues: { customerName: '', customerEmail: '', customerPhone: '', note: '' },
  });

  const onSubmit = (values: BookingFormData) => {
    startTransition(async () => {
      // Note: values.note moze byt null (zod transform) — createBooking to akceptuje.
      const result = await createBooking({ ...values, serviceId, date, time });
      if (!result.ok) {
        toast.error(result.error);
        return;
      }
      router.push(`/rezervacia/potvrdenie/${result.data.bookingId}`);
    });
  };

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4 max-w-lg">
        <FormField
          control={form.control}
          name="customerName"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Meno a priezvisko</FormLabel>
              <FormControl><Input {...field} /></FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <FormField
          control={form.control}
          name="customerEmail"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl><Input type="email" {...field} /></FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <FormField
          control={form.control}
          name="customerPhone"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Telefón</FormLabel>
              <FormControl><Input type="tel" placeholder="+421 9XX XXX XXX" {...field} /></FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <FormField
          control={form.control}
          name="note"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Poznámka (voliteľné)</FormLabel>
              <FormControl><Textarea {...field} value={field.value ?? ''} /></FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <Button type="submit" disabled={isPending} size="lg">
          {isPending ? 'Vytváram...' : 'Vytvoriť rezerváciu'}
        </Button>
      </form>
    </Form>
  );
};
```

### 7. Vytvor confirmation page `src/app/rezervacia/potvrdenie/[id]/page.tsx`

```tsx
import { db } from '@/db';
import { bookings, services } from '@/db/schema';
import { eq } from 'drizzle-orm';
import { notFound } from 'next/navigation';
import { formatBratislavaDateTime } from '@/lib/timezone';
import { formatPriceFromCents } from '@/lib/format';

type Props = { params: Promise<{ id: string }> };

export default async function PotvrdeniePage({ params }: Props) {
  const { id } = await params;
  const [row] = await db
    .select({ booking: bookings, service: services })
    .from(bookings)
    .innerJoin(services, eq(bookings.serviceId, services.id))
    .where(eq(bookings.id, id));

  if (!row) notFound();

  return (
    <div className="max-w-md mx-auto py-16 text-center space-y-6">
      <div className="text-6xl">✅</div>
      <h1 className="text-3xl font-bold">Rezervácia potvrdená!</h1>
      <div className="space-y-2 text-left p-6 border rounded-lg">
        <p><strong>Služba:</strong> {row.service.name}</p>
        <p><strong>Termín:</strong> {formatBratislavaDateTime(row.booking.startTime)}</p>
        <p><strong>Cena:</strong> {formatPriceFromCents(row.service.priceCents)}</p>
        <p><strong>Meno:</strong> {row.booking.customerName}</p>
      </div>
      <p className="text-sm text-muted-foreground">
        Potvrdenie sme ti poslali na <strong>{row.booking.customerEmail}</strong>.
      </p>
    </div>
  );
}
```

> 💡 **Bezpečnosť**: URL `/rezervacia/potvrdenie/<uuid>` je guess-able cez UUID (málo
> pravdepodobné, ale možné). Pre MVP OK; pre paranoid mód by sme pridali token v URL alebo
> ukázali len kým session match-uje.

### 8. Zapoj formulár v `src/app/rezervacia/page.tsx`

V sekcii 3 nahraď placeholder:

```tsx
{params.serviceId && params.date && params.time && (
  <section>
    <h2 className="text-2xl font-semibold mb-4">3. Vaše údaje</h2>
    <CustomerForm
      serviceId={params.serviceId}
      date={params.date}
      time={params.time}
    />
  </section>
)}
```

A pridaj `import { CustomerForm } from './customer-form';`.

### 9. Otestuj

```bash
pnpm dev
```

- Prejdi celý flow: služba → dátum → čas → vyplň formulár → submit
- Po submite ťa malo presmerovať na `/rezervacia/potvrdenie/<id>`
- V Drizzle Studio (`pnpm db:studio`) skontroluj že rezervácia je v `bookings`. `startTime`
  by mal byť UTC (napr. `2026-05-22 11:15:00+00` pre 13:15 v Bratislave v lete)
- Skús submit s prázdnym menom — uvidíš error message pri inpute
- Skús submit s neplatným emailom — uvidíš error
- Otvor druhý browser tab, prejdi flow do rovnakého slotu, submit — dostaneš toast
  "Tento termín už nie je voľný"

### 10. Commit

```bash
git add .
git commit -m "task 09: booking flow step 3 - customer form with zod, race-condition guard, TZ-safe"
git push
```

## Acceptance Criteria

- [ ] Formulár má 4 polia: meno, email, telefón, poznámka
- [ ] Validačné errory sa zobrazujú pri inputoch (cez `FormMessage`)
- [ ] Submit volá `createBooking` server action
- [ ] Po úspechu redirect na `/rezervacia/potvrdenie/<id>`
- [ ] Potvrdovacia stránka zobrazí detaily rezervácie (čas v Bratislavskej zone, cena ako `"18,00 €"`)
- [ ] Race-condition guard cez DB query: druhý pokus o ten istý slot dostane toast error
- [ ] V DB sa `startTime`/`endTime` ukladá ako UTC (skontroluj v Drizzle Studio — má `+00` suffix)
- [ ] Po vytvorení rezervácie je daný slot odstránený zo zoznamu voľných slotov v `/rezervacia`

## Tipy a riešenia problémov

**Problém:** `Error: zodResolver is not a function`
**Riešenie:** Import: `import { zodResolver } from '@hookform/resolvers/zod';` (zo subcesty
`/zod`, nie z root).

**Problém:** Formulár sa nesubmituje
**Riešenie:** Skontroluj že máš `<form onSubmit={form.handleSubmit(onSubmit)}>` a že tlačidlo
má `type="submit"`.

**Problém:** Toast sa neobjaví
**Riešenie:** `<Toaster />` musí byť v `layout.tsx`. Bez neho toasty nikam nevyskakujú.

**Problém:** Po submite vidím v DB `startTime` v zlej zone
**Riešenie:** Skontroluj že používaš `buildBratislavaDateTime(date, time)` a NIE `new Date(...)`.
Schema musí mať `timestamp({ withTimezone: true })`.

**Problém:** TypeScript hlási error pri `note` poli
**Riešenie:** `CreateBookingInput.note` má byť typ `string | null | undefined` (akceptuje
všetky tri). Zod transform v schéme znormalizuje na `null`, DB stĺpec je nullable.

## Pýtanie sa Claude Code

- *"Vysvetli mi, prečo robíme validáciu cez Zod aj na klientovi aj na serveri. Nie je to
  duplikácia?"*
- *"Čo robí `revalidatePath('/rezervacia')` v server action? Čo by sa stalo, keby som ho
  odstránil?"*
- *"Môj race-condition guard cez `lt(start, endTime) AND gt(end, startTime)` — vysvetli mi
  logiku presahu intervalov. Prečo to funguje?"*

## Ďalší krok

➡️ [Task 10 — Better Auth setup](10-better-auth-setup.md)
