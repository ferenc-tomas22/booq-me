# Task 02 — Base Layout + Komponenty

**Modul:** 1 — Setup & základy
**Čas:** ~45 min
**Obtiažnosť:** ★★☆☆☆

## Čo sa naučíš

- Ako funguje **shadcn/ui** — knižnica, ktorá kopíruje kód do tvojho repa
- Ako pridať ďalšie shadcn komponenty (`button`, `card`, `input`, `label`)
- Ako spraviť responzívny base layout (header, main, footer)
- Rozdiel medzi **server component** a **client component**

## Background

### Čo je shadcn/ui — vlastne?

Klasické komponentové knižnice (Material UI, Chakra, Ant Design) ti dajú komponenty ako
**npm balíky**, ktoré importuješ. Problém: keď chceš niečo upraviť, musíš bojovať s ich API.

**shadcn/ui** to robí naopak — komponenty ti **skopíruje priamo do tvojho repa** ako
TypeScript súbory. Sú postavené na **Radix UI** (headless = bez vizuálu) + **Tailwind**
(vizuál). Vďaka tomu si ich vieš donekonečna upravovať bez bojov.

V Tasku 01 ti `init` vytvoril aspoň `Button` komponent. Teraz pridáme ďalšie.

### Server vs Client komponenty

V Next.js App Routeri je **každý komponent default Server Component**:
- Beží **na serveri**, nikdy v browseri
- Vie sa priamo dostať k DB, env vars, filesystemu
- Nemá `useState`, `useEffect`, `onClick` — nič interaktívne
- Výsledok je čistý HTML poslaný do browsera

**Client Component** (označený `'use client'` na začiatku súboru):
- Beží v browseri (po hydrátii)
- Má `useState`, `useEffect`, eventy
- Nemôže priamo na DB

Header a footer budú **server components** (žiadna interakcia). Neskôr v Tasku 07 spravíme
prvý client component.

## Tvoja úloha

### 1. Pridaj ďalšie shadcn komponenty

Cez shadcn CLI pridáme komponenty, ktoré budeme potrebovať v ďalších taskoch:

```bash
pnpm dlx shadcn@latest add card input label
```

shadcn ich vloží do `src/components/ui/`. Pozri sa do týchto súborov — sú to obyčajné
React komponenty s Tailwind triedami. Žiadna mágia.

> 💡 **Skús zmeniť čokoľvek v `button.tsx`** — napríklad pridaj `text-uppercase` class. V
> dev mode uvidíš zmenu okamžite. To je sila shadcn — komponenty sú **tvoje**.

### 2. Nastav metadata appky

Otvor `src/app/layout.tsx`. Zmeň `metadata`:

```tsx
export const metadata: Metadata = {
  title: 'Samuel Agošton — Pánsky barber',
  description: 'Rezervuj si termín u Samuela. Pánske strihy, úprava brady, detské strihy.',
};
```

A zmeň jazyk dokumentu:

```tsx
<html lang="sk">
```

> 💡 **Prečo `lang="sk"`?** Pretože appka je v slovenčine. Browser a screen readery to
> potrebujú vedieť — napríklad pre hyphenation, výslovnosť, a SEO.

### 3. Skontroluj že fonty fungujú

V `globals.css` (vytvoril ho `init` v Tasku 01) by si mal vidieť `@theme` blok ktorý
obsahuje tvoj zvolený font:

```css
@theme {
  --font-sans: ...;
  --font-heading: ...;
}
```

Ak ten blok nevidíš (rôzne verzie shadcn ho niekedy vynechajú), pridaj ho ručne. Sprav
fontom čo si vybral na ui.shadcn.com/create.

V `layout.tsx` by `<body>` mal mať `className="font-sans"` (alebo podobne). Skontroluj.

### 4. Vytvor `src/components/site-header.tsx`

```tsx
import Link from 'next/link';

export const SiteHeader = () => (
  <header className="border-b">
    <div className="container mx-auto flex h-16 items-center justify-between px-4">
      <Link href="/" className="text-xl font-bold">
        Samuel Agošton
      </Link>
      <nav className="flex gap-6 text-sm">
        <Link href="/" className="hover:underline">Domov</Link>
        <Link href="/rezervacia" className="hover:underline">Rezervácia</Link>
      </nav>
    </div>
  </header>
);
```

### 5. Vytvor `src/components/site-footer.tsx`

```tsx
export const SiteFooter = () => (
  <footer className="border-t mt-16">
    <div className="container mx-auto px-4 py-8 text-sm text-muted-foreground">
      © {new Date().getFullYear()} Samuel Agošton. Všetky práva vyhradené.
    </div>
  </footer>
);
```

> 💡 **Prečo arrow function namiesto `function`?** Štýlová preferencia — väčšina Next.js
> kódu je arrow functions. Obe varianty fungujú rovnako, drž sa jednej v celej appke.

### 6. Zapoj header a footer do layoutu

V `src/app/layout.tsx`:

```tsx
import { SiteHeader } from '@/components/site-header';
import { SiteFooter } from '@/components/site-footer';

// ... vnútri RootLayout, v `body`:
<body className="font-sans antialiased">
  <SiteHeader />
  <main className="container mx-auto px-4 py-8">
    {children}
  </main>
  <SiteFooter />
</body>
```

### 7. Otestuj

```bash
pnpm dev
```

Otvor `localhost:3000`. Mal by si vidieť:
- Hore: navigáciu s "Samuel Agošton" + "Domov" + "Rezervácia"
- V strede: defaultný Next.js content
- Dole: copyright

Klikni "Rezervácia" — dostaneš **404**. To je správne, túto stránku zatiaľ nemáš.

### 8. Vyskúšaj responsivitu

V Chrome DevTools (F12) → toggle device toolbar → vyber iPhone 12 → reload. Layout by mal:
- Header zostáva readable, nič nepretečie
- Padding na bokoch nemizne

### 9. Commit a push

```bash
git add .
git commit -m "task 02: shadcn components, base layout with header/footer, metadata"
git push
```

## Acceptance Criteria

- [ ] `src/components/ui/` obsahuje aspoň `button.tsx`, `card.tsx`, `input.tsx`, `label.tsx`
- [ ] V `layout.tsx` je `<html lang="sk">` a tvoja `metadata` (title + description)
- [ ] `globals.css` obsahuje `@theme` blok s `--font-sans` (a/alebo `--font-heading`)
- [ ] Header (`SiteHeader`) a footer (`SiteFooter`) sa zobrazujú na každej stránke
- [ ] `pnpm dev` beží bez errorov
- [ ] Layout je responzívny — vyskúšaj iPhone 12 (390px), iPad (768px), desktop (1280px)
- [ ] Žiadne `Error: ...` v console (warningy sú OK)

## Tipy a riešenia problémov

**Problém:** Font sa nezobrazuje (vidím defaultný serif)
**Riešenie:** Skontroluj `globals.css` že má `@theme { --font-sans: ... }`. Tailwind v4 si
ho načíta automaticky. Bez toho bloku Tailwind nevie aký font použiť pre `font-sans` class.

**Problém:** `container mx-auto` nerobí čo má (nie je centrované)
**Riešenie:** Tailwind v4 zmenil defaulty pre `container`. Skontroluj že máš `@import
"tailwindcss";` v `globals.css` (mal by tam byť po inite).

**Problém:** Hover na link nerobí underline
**Riešenie:** `hover:underline` funguje len ak je dev server bežiaci a Tailwind ho vidí.
Niekedy treba dev server restartnúť po pridaní novej class.

**Problém:** `Cannot find module '@/components/site-header'`
**Riešenie:** Skontroluj `tsconfig.json` že má `"paths": { "@/*": ["./src/*"] }`. shadcn
init to nastaví, ale ak si si menil štruktúru, môže byť rozbité.

## Pýtanie sa Claude Code

- *"Vysvetli mi rozdiel medzi server components a client components v Next.js. Header a
  footer som spravil ako server components — je to OK?"*
- *"Ako môžem ku komponentu shadcn `Button` pridať vlastný variant (napríklad `gold`)? Ukáž
  mi krok po kroku."*
- *"V `globals.css` mám `@theme` blok s mojimi farbami a fontmi. Ako presne ich Tailwind v4
  premieňa na CSS triedy?"*

## Ďalší krok

➡️ [Task 03 — Public landing page](03-landing-page.md)
