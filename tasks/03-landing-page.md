# Task 03 — Public Landing Page

**Modul:** 1 — Setup & základy
**Čas:** ~60 min
**Obtiažnosť:** ★★☆☆☆

## Čo sa naučíš

- Ako navrhnúť čitateľný landing page (hero, sekcie, CTA)
- Ako používať shadcn `Card` komponent
- Ako pracovať s ikonami cez **lucide-react** (alebo inú knižnicu, ktorú si si vybral)
- Semantic HTML — `section`, `article`, `address`
- Helper funkcia na formátovanie ceny (zo `centov` na `"18,00 €"`)

## Background

**Landing page** = prvá stránka, ktorú vidí návštevník. Musí mu za 5 sekúnd povedať:
1. **Čo to je** (kaderníctvo)
2. **Pre koho** (páni, deti)
3. **Čo má spraviť** (rezervovať si termín)

V tomto tasku všetko **hardcodujeme** — služby napíšeme priamo do JSX. V tasku 07 to
nahradíme dátami z databázy.

### Centy vs eurá — prečo ukladáme cenu ako celé číslo?

V profesionálnych appkách sa **peniaze nikdy neukladajú ako floating-point** (`18.00`).
Floating-point má neistú reprezentáciu (`0.1 + 0.2 === 0.30000000000000004`). Štandard:

- Ulož cenu ako **celé čísla v najmenšej jednotke** (centy pre EUR, fillér pre HUF, ...)
- Zobraz cez **formatter** (`18,00 €`)

Takže miesto `priceEur: 18.50` budeme používať `priceCents: 1850`. Helper na zobrazenie
napíšeme za chvíľu.

## Tvoja úloha

### 1. Pridaj ikony

```bash
pnpm add lucide-react
```

[Lucide](https://lucide.dev) je sada vyše 1000 SVG ikon ako React komponenty. Ak si si v
Tasku 01 vybral inú knižnicu (Radix, Tabler), pridaj tú namiesto Lucide.

### 2. Vytvor formatter pre cenu — `src/lib/format.ts`

```ts
export const formatPriceFromCents = (cents: number): string => {
  const euros = cents / 100;
  return new Intl.NumberFormat('sk-SK', {
    style: 'currency',
    currency: 'EUR',
  }).format(euros);
};

export const formatDuration = (minutes: number): string => {
  if (minutes < 60) return `${minutes} min`;
  const hours = Math.floor(minutes / 60);
  const rest = minutes % 60;
  return rest === 0 ? `${hours} h` : `${hours} h ${rest} min`;
};
```

> 💡 **`Intl.NumberFormat`** je vstavané v JavaScripte (a browseroch). Vie formátovať čísla,
> dátumy, meny podľa lokále. Pre Slovensko vyrobí `"18,00 €"` (čiarka ako desatinný oddelovač).

### 3. Prepíš `src/app/page.tsx`

Stránka má 3 sekcie:

**A) Hero**
- Veľký nadpis: "Pánsky barber v centre Bratislavy"
- Podnadpis: "Strihy, brady, detské strihy. Rezervácia online za 60 sekúnd."
- Veľké CTA tlačidlo: "Rezervovať termín" → linkuje na `/rezervacia`

**B) Služby**
- Mriežka 5 kariet (Card komponent zo shadcn)
- Každá karta: ikona + názov + dĺžka + cena + popis
- Zoznam služieb (skopíruj z booqme.sk Men's Hub) — **cena je v centoch**:

```ts
import { Scissors, Sparkles, Baby } from 'lucide-react';

const services = [
  {
    icon: Scissors,
    name: 'Pánsky strih',
    durationMinutes: 45,
    priceCents: 1800,
    description: 'Strih nožnicami alebo strojčekom, umytie vlasov, styling, úprava obočia.',
  },
  {
    icon: Sparkles, // brada
    name: 'Úprava brady',
    durationMinutes: 30,
    priceCents: 1500,
    description: 'Úprava brady strojčekom, britvou, hot towel.',
  },
  {
    icon: Sparkles,
    name: 'Pánsky strih + úprava brady',
    durationMinutes: 75,
    priceCents: 3000,
    description: 'Kompletná úprava: vlasy aj brada.',
  },
  {
    icon: Baby,
    name: 'Detský strih',
    durationMinutes: 35,
    priceCents: 1800,
    description: 'Strih nožnicami alebo strojčekom, umytie, styling. Pre chlapcov do 12 rokov.',
  },
  {
    icon: Scissors,
    name: 'Dlhé vlasy',
    durationMinutes: 75,
    priceCents: 2100,
    description: 'Strih dlhých vlasov, umytie, styling.',
  },
];
```

> 💡 **Lucide nemá ikonu špeciálne pre bradu** — použili sme `Sparkles` (svieži look). Skús
> si pozrieť [lucide.dev/icons](https://lucide.dev/icons) a vybrať čo sa ti páči viac.
> Alternatívy mimo Lucide: [Tabler Icons](https://tabler.io/icons) má `Moustache`,
> `Beard`.

**C) Kontakt**
- Adresa salónu (vymysli si — napr. *"Hlavná 12, 811 01 Bratislava"*)
- Telefón
- Otváracie hodiny (Po–Pi 9:00–19:00, So 9:00–14:00)
- Použí `<address>` element

### 4. Štýluj cez Tailwind

Niekoľko princípov:

- **Vertikálne medzery medzi sekciami**: `py-16` (64px hore aj dole)
- **Hero**: `text-center`, veľký nadpis `text-5xl md:text-6xl font-bold`
- **Grid pre služby**: `grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6`
- **Cards**: shadcn `Card`, `CardHeader`, `CardTitle`, `CardDescription`, `CardContent`
- **CTA tlačidlo**: shadcn `Button` so `size="lg"`

Príklad jednej karty:

```tsx
import { formatPriceFromCents, formatDuration } from '@/lib/format';

// ...vo vnútri grid:
{services.map((service) => {
  const Icon = service.icon;
  return (
    <Card key={service.name}>
      <CardHeader>
        <div className="flex justify-between items-start">
          <div className="flex items-center gap-3">
            <Icon className="h-6 w-6" />
            <CardTitle>{service.name}</CardTitle>
          </div>
          <div className="text-right">
            <div className="font-semibold">{formatPriceFromCents(service.priceCents)}</div>
            <div className="text-sm text-muted-foreground">
              {formatDuration(service.durationMinutes)}
            </div>
          </div>
        </div>
      </CardHeader>
      <CardContent>
        <p className="text-sm text-muted-foreground">{service.description}</p>
      </CardContent>
    </Card>
  );
})}
```

### 5. Vytvor stub stránku `/rezervacia`

```tsx
// src/app/rezervacia/page.tsx
export default function RezervaciaPage() {
  return (
    <div className="py-16 text-center">
      <h1 className="text-4xl font-bold">Rezervácia</h1>
      <p className="mt-4 text-muted-foreground">
        Bude tu rezervačný formulár (Task 07).
      </p>
    </div>
  );
}
```

### 6. Pozri si výsledok

```bash
pnpm dev
```

Otvor `localhost:3000`. Skús:
- Či sa stránka pekne vyzerá na desktope
- DevTools → toggle device toolbar (iPhone) → či sa nič nerozbije na mobile
- Klik na "Rezervovať termín" → dostaneš sa na `/rezervacia` stub
- Skontroluj že cena sa zobrazí ako `"18,00 €"` (slovenský formát s čiarkou)

### 7. Commit

```bash
git add .
git commit -m "task 03: landing page with hero, services grid, contact, price formatter"
git push
```

## Acceptance Criteria

- [ ] `pnpm add lucide-react` prebehlo
- [ ] `src/lib/format.ts` exportuje `formatPriceFromCents` a `formatDuration`
- [ ] `src/app/page.tsx` má **hero** sekciu s nadpisom, podnadpisom a CTA tlačidlom
- [ ] **Služby** sekcia má 5 kariet, každá s ikonou, názvom, **cenou ako `"18,00 €"`** a popisom
- [ ] **Kontakt** sekcia má adresu (v `<address>` element), telefón, otváracie hodiny
- [ ] Klik na CTA z hera ťa zoberie na `/rezervacia` (stub stránka)
- [ ] Layout je responzívny — iPhone 12, iPad, desktop
- [ ] Žiadne errory v console

## Bonus

- Pridaj sekciu "Hodnotenia" (3–5 mock recenzií s menom, hodnotením `★★★★★` a textom)
- Pridaj Google Maps embed do kontaktu (cez `<iframe>` z Google Maps share linku)
- Pridaj smooth scroll pre anchor linky (`html { scroll-behavior: smooth; }` v `globals.css`)

## Tipy a riešenia problémov

**Problém:** Mriežka kariet sa zobrazuje ako 1 stĺpec aj na desktope
**Riešenie:** Skontroluj že máš `md:grid-cols-2 lg:grid-cols-3` — bez breakpoint prefixov by
to zostalo `grid-cols-1` všade.

**Problém:** Ikony lucide-react sú obrovské
**Riešenie:** Default `<svg>` je veľký. Pridaj `className="h-5 w-5"` (alebo `h-6 w-6`).

**Problém:** Cena sa zobrazuje ako `1800` namiesto `18,00 €`
**Riešenie:** Volaš `formatPriceFromCents(service.priceCents)`? Bez formatter funkcie ti
nič nedá `18,00 €`.

**Problém:** Hero text je zalomený na zlých miestach na mobile
**Riešenie:** Skús `max-w-2xl mx-auto` na hero container — text sa zúži na čitateľnú šírku.

## Pýtanie sa Claude Code

- *"Ukáž mi, ako napísať Hero sekciu pre kaderníctvo s Tailwindom, ktorá vyzerá moderne a
  profesionálne. Nepoužívaj komponenty knižníc, len Tailwind triedy."*
- *"Aký je rozdiel medzi `<section>`, `<article>` a `<div>`? Kedy ktorý použiť?"*
- *"`Intl.NumberFormat('sk-SK', { style: 'currency', currency: 'EUR' })` — ako presne to
  funguje? Vie spraviť rovnaké pre HUF, USD, CZK?"*

## Ďalší krok

➡️ [Task 04 — Neon DB + env vars](04-neon-database.md)
