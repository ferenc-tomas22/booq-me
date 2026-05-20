# Task 07 — Booking Flow: Vyber Služby

**Modul:** 3 — Rezervačný flow
**Čas:** ~45 min
**Obtiažnosť:** ★★★☆☆

## Čo sa naučíš

- Rozdiel medzi **Server Component** a **Client Component**
- Ako fetchovať dáta z DB priamo v Server Component (bez API route)
- Ako zdieľať state medzi rôznymi krokmi flow-u (URL search params)
- Pattern progresívneho odhalovania krokov (ako booqme)
- 3 nové JavaScript/Next.js API: `cn()`, `useSearchParams`, `URLSearchParams`

## Background

### Server vs Client Components (rekapitulácia)

V Tasku 02 sme si vysvetlili základ. Header a footer sú server components — žiadna interakcia.

**Náš booking flow**: rodičovský komponent je server (fetch služieb z DB), interakcie sú
v client komponentoch (klikanie na karty, prechody medzi krokmi).

### Ako zdieľame state medzi krokmi?

Booqme všetko drží v lokálnom React state — keď refreshneš, stratíš pokrok. Lepšie riešenie:
**URL search params**.

```
/rezervacia                                              → krok 1 (vyber služby)
/rezervacia?serviceId=abc                                → krok 2 (vyber dátum + čas)
/rezervacia?serviceId=abc&date=2026-05-22&time=13:15     → krok 3 (formulár)
```

Výhody:
- Refresh stránky nestratí pokrok
- Vieš zdieľať link na konkrétny krok
- "Späť" v browseri funguje prirodzene
- Žiadny client state, žiadne `useState`

### Tri nové veci, ktoré dnes uvidíš

| Vec | Čo je | Príklad |
|---|---|---|
| **`cn(...)`** | Utility z `src/lib/utils.ts` (pridal ti to shadcn `init`). Spojí Tailwind class stringy a vyrieši konflikty. | `cn('px-4', isActive && 'bg-blue-500')` → vráti string podľa podmienok |
| **`useSearchParams()`** | Next.js hook ktorý vráti aktuálne URL search params ako čítací objekt | `searchParams.get('serviceId')` vráti string alebo null |
| **`URLSearchParams`** | Vstavané JavaScript API (nie z Reactu). Vytvorí mutable kópiu search params, ktorú vieš upraviť a serializovať späť | `new URLSearchParams({ a: '1' }).toString()` → `"a=1"` |

`useSearchParams` je **iba pre čítanie**. Keď chceš zmeniť URL, vyrobíš novú instanciu
`URLSearchParams`, upravíš, a pushneš cez `router.push()`. Uvidíš to nižšie.

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
> V Next.js 16 sú `searchParams` aj `params` **async** (musíš ich awaitnuť). Predtým boli
> sync. Toto je nová API.

### 2. Vytvor `src/app/rezervacia/service-picker.tsx` ako Client Component

```tsx
'use client';

import { useRouter, useSearchParams } from 'next/navigation';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { cn } from '@/lib/utils';
import { formatPriceFromCents, formatDuration } from '@/lib/format';
import type { Service } from '@/db/schema';

type Props = {
  services: Service[];
  selectedId?: string;
};

export const ServicePicker = ({ services, selectedId }: Props) => {
  const router = useRouter();
  const searchParams = useSearchParams();

  const handleSelect = (id: string) => {
    // useSearchParams() vráti read-only objekt. Aby sme ho mohli zmeniť, urobíme z neho
    // mutable kópiu cez `new URLSearchParams(...)`.
    const params = new URLSearchParams(searchParams);
    params.set('serviceId', id);
    // Keď meníme službu, resetujeme date/time — staré hodnoty by neboli zmysluplné.
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
                <div className="font-semibold">{formatPriceFromCents(service.priceCents)}</div>
                <div className="text-sm text-muted-foreground">
                  {formatDuration(service.durationMinutes)}
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

> 💡 **`cn(...)`** — čo presne robí? Spája Tailwind class stringy a vyrieši konflikty.
> Príklad: `cn('p-4', isActive && 'bg-blue', 'p-6')` → výsledok je `"bg-blue p-6"` (lebo
> `p-6` overridne `p-4`). Bez `cn` by si mal v HTML `"p-4 bg-blue p-6"` a Tailwind by mohol
> aplikovať obe → nepredvídateľné.

### 3. Otestuj

```bash
pnpm dev
```

Otvor `localhost:3000/rezervacia`. Mal by si vidieť:
- 5 kariet so službami zo seedu
- Cena ako `"18,00 €"` (slovenský formát)
- Po kliknutí na kartu sa zvýrazní (border + ring)
- URL sa zmení na `?serviceId=...`
- Pod ňou sa objaví druhá sekcia (zatiaľ placeholder)
- Po kliku na inú kartu sa selection prepne, druhá sekcia stále zobrazená

### 4. Refresh test

Refreshni stránku — service je stále vybraný (vidíš to v URL). To je sila URL state.

### 5. Skús späť v browseri

Klikni "Späť" — URL sa vráti, selection zmizne. To je tiež zadarmo (browser history aware).

### 6. Commit

```bash
git add .
git commit -m "task 07: booking flow step 1 - service picker with URL state"
git push
```

## Acceptance Criteria

- [ ] `/rezervacia` zobrazuje zoznam aktívnych služieb z DB
- [ ] Cena sa zobrazí ako `"18,00 €"` (formatter z Task 03)
- [ ] Klik na kartu zmení URL na `?serviceId=...`
- [ ] Vybraná karta má vizuálne odlíšenie (border + ring)
- [ ] Po výbere služby sa zobrazí druhá sekcia (zatiaľ placeholder)
- [ ] Refresh stránky zachová výber služby
- [ ] Browser "Späť" tlačidlo zruší selection
- [ ] Klik na inú službu prepne výber správne
- [ ] Žiadne console errory

## Tipy a riešenia problémov

**Problém:** `Error: Dynamic server usage: Page couldn't be rendered statically because it used 'searchParams'.`
**Riešenie:** Toto je len varovanie — Next.js ti hovorí, že stránka nemôže byť pre-renderovaná
ako static HTML. Pre rezerváciu to je OK (potrebujeme čerstvé dáta). Ak chceš varovanie
schovať, pridaj na vrch súboru `export const dynamic = 'force-dynamic';`.

**Problém:** Po kliku na kartu sa stránka scroluje na vrch
**Riešenie:** Pridaj `{ scroll: false }` do `router.push()` (mám tam to).

**Problém:** TypeScript error: `Property 'serviceId' does not exist on type 'Promise<...>'`
**Riešenie:** Awaitneš `searchParams`: `const params = await searchParams;` a potom
používaš `params.serviceId`, nie `searchParams.serviceId`.

**Problém:** Karta sa nezvýrazni po vybraní
**Riešenie:** Skontroluj že `selectedId === service.id` (porovnávaš stringy). Použí
`console.log(selectedId, service.id)` v komponente.

## Pýtanie sa Claude Code

- *"Vysvetli mi rozdiel `router.push` vs `router.replace` vs `<Link>`. Kedy ktoré použiť?"*
- *"Prečo musí byť `ServicePicker` client component? Čo by sa stalo, keby som dal `onClick`
  na server component?"*
- *"Ako sa správa `useSearchParams` keď používateľ stlači 'Späť' v browseri? Re-renderuje
  sa komponent?"*

## Ďalší krok

➡️ [Task 08 — Booking flow: kalendár + čas](08-booking-date-time.md)
