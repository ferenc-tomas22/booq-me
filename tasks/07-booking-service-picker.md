# Task 07 — Booking Flow: Vyber Služby

**Modul:** 3 — Rezervačný flow
**Čas:** ~45 min
**Obtiažnosť:** ★★★☆☆

## Čo sa naučíš

- Rozdiel medzi **Server Component** a **Client Component**
- Ako fetchovať dáta z DB priamo v Server Component (bez API route)
- Ako zdielať state medzi rôznymi krokmi flow-u (URL search params)
- Pattern progresívneho odhalovania krokov (ako booqme)

## Background

### Server vs Client Components

V Next.js App Routeri je **každý komponent default Server Component**. To znamená:
- Beží **na serveri**, nikdy v browseri
- Vie sa dostať priamo k DB, env vars, filesystemu
- Nemá `useState`, `useEffect`, click handlery — nič interaktívne
- Výsledok je čistý HTML poslaný do browsera

**Client Component** (označený `'use client'` na začiatku súboru):
- Beží v browseri (po hydrátii)
- Má `useState`, `useEffect`, eventy
- Nemôže priamo na DB (musí volať server actions alebo API)

**Náš booking flow**: rodičovský komponent je server (fetch služieb z DB), interakcie sú v
client komponentoch (klikanie na karty, prechody medzi krokmi).

### Ako zdielame state medzi krokmi?

Booqme všetko drží v lokálnom React state — keď refreshneš, stratíš pokrok. Lepšie riešenie:
**URL search params**.

```
/rezervacia                              → krok 1 (vyber služby)
/rezervacia?serviceId=abc                → krok 2 (vyber dátum + čas)
/rezervacia?serviceId=abc&date=2026-05-22&time=13:15  → krok 3 (formulár)
```

Výhody:
- Refresh stránky nestratí pokrok
- Vieš zdieľať link na konkrétny krok
- "Späť" v browseri funguje prirodzene
- Žiadny client state, žiadne `useState`

## Tvoja úloha

### 1. Prepíš `src/app/rezervacia/page.tsx` ako Server Component

```tsx
import { db } from '@/db';
import { services } from '@/db/schema';
import { eq } from 'drizzle-orm';
import { ServicePicker } from './service-picker';

type Props = {
  searchParams: Promise<{ serviceId?: string }>;
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
          <p className="text-muted-foreground">Bude tu kalendár (Task 08).</p>
        </section>
      )}
    </div>
  );
}
```

> 💡 **Prečo `Promise<{...}>` pre searchParams?**
> V Next.js 16 sú searchParams **async** (musíš ich awaitnut). Predtým boli sync. Toto je
> nová API.

### 2. Vytvor `src/app/rezervacia/service-picker.tsx` ako Client Component

```tsx
'use client';

import { useRouter, useSearchParams } from 'next/navigation';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { cn } from '@/lib/utils';
import type { Service } from '@/db/schema';

type Props = {
  services: Service[];
  selectedId?: string;
};

export const ServicePicker = ({ services, selectedId }: Props) => {
  const router = useRouter();
  const searchParams = useSearchParams();

  const handleSelect = (id: string) => {
    const params = new URLSearchParams(searchParams);
    params.set('serviceId', id);
    // Keď meníme službu, resetujeme date/time.
    params.delete('date');
    params.delete('time');
    router.push(`?${params.toString()}`, { scroll: false });
  };

  return (
    <div className="grid gap-4">
      {services.map((service) => (
        <Card
          key={service.id}
          onClick={() => handleSelect(service.id)}
          className={cn(
            'cursor-pointer transition hover:border-primary',
            selectedId === service.id && 'border-primary ring-2 ring-primary/20',
          )}
        >
          <CardHeader>
            <div className="flex justify-between items-start">
              <CardTitle>{service.name}</CardTitle>
              <div className="text-right">
                <div className="font-semibold">{service.priceEur} €</div>
                <div className="text-sm text-muted-foreground">
                  {service.durationMinutes} min
                </div>
              </div>
            </div>
          </CardHeader>
          {service.description && (
            <CardContent>
              <p className="text-sm text-muted-foreground">{service.description}</p>
            </CardContent>
          )}
        </Card>
      ))}
    </div>
  );
};
```

### 3. Otestuj

```bash
pnpm dev
```

Otvor `localhost:3000/rezervacia`. Mal by si vidieť:
- 5 kariet so službami zo seedu
- Po kliknutí na kartu sa zvýrazní (`border-primary`)
- URL sa zmení na `?serviceId=...`
- Pod ňou sa objaví druhá sekcia "Vyberte dátum a čas" (zatiaľ placeholder)
- Po kliku na inú kartu sa selection prepne, druhá sekcia stále zobrazená

### 4. Refresh test

Refreshni stránku — service je stále vybraný (vidíš to v URL). To je sila URL state.

### 5. Commit

```bash
git add .
git commit -m "task 07: booking flow step 1 - service picker with URL state"
git push
```

## Acceptance Criteria

- [ ] `/rezervacia` zobrazuje zoznam aktívnych služieb z DB
- [ ] Klik na kartu zmení URL na `?serviceId=...`
- [ ] Vybraná karta má vizuálne odlíšenie (border + ring)
- [ ] Po výbere služby sa zobrazí druhá sekcia (zatiaľ placeholder)
- [ ] Refresh stránky zachová výber služby
- [ ] Klik na inú službu prepne výber správne
- [ ] Žiadne console errory ("Warning: ..." sú OK, "Error:" nie)

## Tipy a riešenia problémov

**Problém:** `Error: Dynamic server usage: Page couldn't be rendered statically because it used 'searchParams'.`
**Riešenie:** Toto je len varovanie — Next.js ti hovorí, že stránka nemôže byť pre-renderovaná
ako static HTML. Pre rezerváciu to je OK (potrebujeme čerstvé dáta). Ak chceš varovanie schovať,
pridaj na vrch súboru `export const dynamic = 'force-dynamic';`.

**Problém:** Po kliku na kartu sa stránka scroluje na vrch
**Riešenie:** Pridaj `{ scroll: false }` do `router.push()` (mám tam to).

**Problém:** TypeScript error: `Property 'serviceId' does not exist on type 'Promise<...>'`
**Riešenie:** Awaitneš `searchParams`: `const params = await searchParams;` a potom používaš
`params.serviceId`, nie `searchParams.serviceId`.

**Problém:** Karta sa nezvýrazni po vybraní
**Riešenie:** Skontroluj že `selectedId === service.id` (porovnávaš stringy). Občas to môže
byť `selectedId == service.id` problém s typmi.

## Pýtanie sa Claude Code

- *"Vysvetli mi rozdiel `router.push` vs `router.replace` vs `<Link>`. Kedy ktoré použiť?"*
- *"Prečo musí byť `ServicePicker` client component? Čo by sa stalo, keby som dal `onClick`
  na server component?"*
- *"Ako sa správa `useSearchParams` keď používateľ stlači 'Späť' v browseri?"*

## Ďalší krok

➡️ [Task 08 — Booking flow: kalendár + čas](08-booking-date-time.md)
