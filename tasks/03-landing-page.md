# Task 03 — Public Landing Page

**Modul:** 1 — Setup & základy
**Čas:** ~60 min
**Obtiažnosť:** ★★☆☆☆

## Čo sa naučíš

- Ako navrhnúť čitateľný landing page (hero, sekcie, CTA)
- Ako používať shadcn `Card` komponent
- Ako pracovať s ikonami cez **lucide-react**
- Semantic HTML — `section`, `article`, `address`

## Background

**Landing page** = prvá stránka, ktorú vidí návštevník. Musí mu za 5 sekúnd povedať:
1. **Čo to je** (kaderníctvo)
2. **Pre koho** (páni, deti)
3. **Čo má spraviť** (rezervovať si termín)

V tomto tasku všetko **hardcodujeme** — služby napíšeme priamo do JSX. V tasku 07 to nahradíme
dátami z databázy.

## Tvoja úloha

### 1. Pridaj ikony

```bash
pnpm add lucide-react
```

[Lucide](https://lucide.dev) je sada vyše 1000 SVG ikon ako React komponenty. Použiješ ich
ako `<Scissors className="h-5 w-5" />`.

### 2. Prepíš `src/app/page.tsx`

Stránka má mať 3 sekcie:

**A) Hero**
- Veľký nadpis: "Pánsky barber v centre Bratislavy"
- Podnadpis: "Strihy, brady, detské strihy. Rezervácia online za 60 sekúnd."
- Veľké CTA tlačidlo: "Rezervovať termín" → linkuje na `/rezervacia`

**B) Služby**
- Mriežka 5 kariet (Card komponent zo shadcn)
- Každá karta: ikona + názov + dĺžka + cena + popis
- Zoznam služieb (skopíruj z booqme.sk Men's Hub):

```ts
const services = [
  {
    name: 'Pánsky strih',
    duration: 45,
    price: 18,
    description: 'Strih nožnicami alebo strojčekom, umytie vlasov, styling, úprava obočia.',
  },
  {
    name: 'Úprava brady',
    duration: 30,
    price: 15,
    description: 'Úprava brady strojčekom, britvou, hot towel.',
  },
  {
    name: 'Pánsky strih + úprava brady',
    duration: 75,
    price: 30,
    description: 'Kompletná úprava: vlasy aj brada.',
  },
  {
    name: 'Detský strih',
    duration: 35,
    price: 18,
    description: 'Strih nožnicami alebo strojčekom, umytie, styling. Pre chlapcov do 12 rokov.',
  },
  {
    name: 'Dlhé vlasy',
    duration: 75,
    price: 21,
    description: 'Strih dlhých vlasov, umytie, styling.',
  },
];
```

**C) Kontakt**
- Adresa salónu (vymysli si — napr. *"Hlavná 12, 811 01 Bratislava"*)
- Telefón
- Otváracie hodiny (Po–Pi 9:00–19:00, So 9:00–14:00)
- Použí `<address>` element

### 3. Štýluj cez Tailwind

Niekoľko princípov:

- **Vertikálne medzery medzi sekciami**: `py-16` (64px hore aj dole)
- **Hero**: `text-center`, veľký nadpis `text-5xl md:text-6xl font-bold`
- **Grid pre služby**: `grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6`
- **Cards**: použí shadcn `Card`, `CardHeader`, `CardTitle`, `CardDescription`, `CardContent`
- **CTA tlačidlo**: shadcn `Button` so `size="lg"`

### 4. Vytvor stub stránku `/rezervacia`

Aby ti CTA z hero linku niekde smeroval, vytvor `src/app/rezervacia/page.tsx`:

```tsx
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

### 5. Pozri si výsledok

```bash
pnpm dev
```

Otvor `localhost:3000`. Skús:
- Či sa stránka peknú vyzerá na desktope
- Otvor DevTools → toggle device toolbar (iPhone) → či sa nič nerozbije na mobile
- Klikni na "Rezervovať termín" — dostaneš sa na `/rezervacia` stub

### 6. Commit

```bash
git add .
git commit -m "task 03: landing page with hero, services grid, contact info"
git push
```

## Acceptance Criteria

- [ ] `pnpm add lucide-react` prebehlo, ikony fungujú
- [ ] `src/app/page.tsx` obsahuje **hero** sekciu s nadpisom, podnadpisom a CTA tlačidlom
- [ ] **Služby** sekcia má 5 kariet s ikonou, názvom, dĺžkou (`45 min`), cenou (`18 €`) a popisom
- [ ] **Kontakt** sekcia má adresu (v `<address>` element), telefón a otváracie hodiny
- [ ] Klik na CTA z hera ťa zoberie na `/rezervacia` (stub stránka)
- [ ] Layout je responzívny — vyskúšaj iPhone (375px), iPad (768px), desktop (1280px)
- [ ] Žiadne errory v console

## Bonus

- Pridaj sekciu "Hodnotenia" (3–5 mock recenzií s menom, hodnotením `★★★★★` a textom)
- Pridaj Google Maps embed do kontaktu
- Pridaj smooth scroll pre anchor linky (`html { scroll-behavior: smooth; }` v `globals.css`)

## Tipy a riešenia problémov

**Problém:** Mriežka kariet sa zobrazuje ako 1 stĺpec aj na desktope
**Riešenie:** Skontroluj že máš `md:grid-cols-2 lg:grid-cols-3` — bez breakpoint prefixov by
to zostalo `grid-cols-1` všade.

**Problém:** Ikony lucide-react nevyzerajú ako mali — sú obrovské
**Riešenie:** Default veľkosť SVG ikony je veľká. Pridaj `className="h-5 w-5"` (alebo `h-6 w-6`).

**Problém:** Hero text je zalomený na zlých miestach na mobile
**Riešenie:** Použí `<br className="hidden md:block" />` pre zalomenie len na desktope, alebo
necháj prirodzené zalomenie a hraj sa s `max-w-2xl mx-auto`.

## Pýtanie sa Claude Code

- *"Ukáž mi, ako napísať Hero sekciu pre kaderníctvo s Tailwindom, ktorá vyzerá moderne a
  professional. Nepoužívaj komponenty knižníc, len Tailwind triedy."*
- *"Aký je rozdiel medzi `<section>`, `<article>` a `<div>`? Kedy ktorý použiť?"*
- *"Daj mi tip, ako spraviť 'glass morphism' efekt pre Hero (priesvitné pozadie s blur)."*

## Ďalší krok

➡️ [Task 04 — Neon DB + env vars](04-neon-database.md)
