# Task 02 — Tailwind v4 + shadcn/ui + Base Layout

**Modul:** 1 — Setup & základy
**Čas:** ~45 min
**Obtiažnosť:** ★★☆☆☆

## Čo sa naučíš

- Ako funguje **shadcn/ui** — knižnica komponentov, ktorá kopíruje kód do tvojho repa
- Ako nastaviť custom font cez `next/font`
- Ako spraviť responzívny base layout (header, main, footer)
- Rozdiel medzi **server component** a **client component**

## Background

### Čo je shadcn/ui?

Klasické komponentové knižnice (Material UI, Chakra, Ant Design) ti dajú komponenty ako
**npm balíky**, ktoré importuješ. Problém: keď chceš niečo upraviť, musíš bojovať s ich API,
overrideovať CSS a robiť hacky.

**shadcn/ui** to robí naopak — komponenty ti **skopíruje priamo do tvojho repa** ako
TypeScript súbory. Sú postavené na **Radix UI** (headless = bez vizuálu) + **Tailwind** (vizuál).
Vďaka tomu si ich vieš donekonečna upravovať bez bojov.

### Čo je `next/font`?

Načítanie web-fontov je tricky — chceš ich rýchlo, bez "flash of unstyled text" (FOUT), a
ideálne self-hostnuté. `next/font` ti to vyrieši: optimalizuje font, generuje CSS, a v build
time downloadne font priamo do tvojho build outputu.

## Tvoja úloha

### 1. Nastav shadcn/ui

V root tvojej Next.js appky (`booq-me-app/`):

```bash
pnpm dlx shadcn@latest init
```

CLI sa ťa opýta:

| Otázka | Odpoveď |
|--------|---------|
| Which style would you like to use? | **New York** |
| Which color would you like to use as base color? | **Zinc** |
| Would you like to use CSS variables for theming? | **Yes** |

CLI ti:
- vytvorí `components.json` (config shadcn)
- aktualizuje `src/app/globals.css` (Tailwind imports + CSS variables)
- vytvorí `src/lib/utils.ts` (utility `cn()` pre conditional classes)

### 2. Nainštaluj prvé komponenty, ktoré budeš potrebovať

```bash
pnpm dlx shadcn@latest add button card input label
```

Tieto sa pridajú do `src/components/ui/`. Pozri sa do týchto súborov — sú to obyčajné React
komponenty s Tailwind triedami. Žiadna mágia.

### 3. Nastav Inter font

Otvor `src/app/layout.tsx`. Naimportuj `Inter` z `next/font/google`:

```tsx
import { Inter } from 'next/font/google';

const inter = Inter({ subsets: ['latin'], variable: '--font-sans' });
```

A pridaj `inter.variable` na `<html>` tag:

```tsx
<html lang="sk" className={inter.variable}>
```

> 💡 **Prečo `lang="sk"`?** Pretože appka je v slovenčine. Browser a screen readery to potrebujú
> vedieť — napríklad pre hyphenation a výslovnosť.

### 4. Naprav metadata

V `src/app/layout.tsx` zmeň `metadata`:

```tsx
export const metadata: Metadata = {
  title: 'Samuel Agošton — Pánsky barber',
  description: 'Rezervuj si termín u Samuela. Pánske strihy, úprava brady, detské strihy.',
};
```

### 5. Spravme base layout

Vytvor `src/components/site-header.tsx`:

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

Vytvor `src/components/site-footer.tsx`:

```tsx
export const SiteFooter = () => (
  <footer className="border-t mt-16">
    <div className="container mx-auto px-4 py-8 text-sm text-muted-foreground">
      © {new Date().getFullYear()} Samuel Agošton. Všetky práva vyhradené.
    </div>
  </footer>
);
```

> 💡 **Prečo `arrow function` namiesto `function`?** Štýlová preferencia. La-fly to má vynútené
> cez ESLint. Oba prístupy fungujú rovnako.

### 6. Zapoj header a footer do layoutu

V `src/app/layout.tsx`:

```tsx
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

Klikni na "Rezervácia" — dostaneš **404**. To je správne, túto stránku zatiaľ nemáš.

### 8. Commit a push

```bash
git add .
git commit -m "task 02: tailwind, shadcn/ui setup, base layout with header/footer"
git push
```

## Acceptance Criteria

- [ ] `components.json` existuje v rooti
- [ ] `src/components/ui/` obsahuje `button.tsx`, `card.tsx`, `input.tsx`, `label.tsx`
- [ ] V `layout.tsx` je `Inter` font importovaný cez `next/font/google`
- [ ] `<html lang="sk">` a custom `metadata` (title + description)
- [ ] Header (`SiteHeader`) a footer (`SiteFooter`) sú zobrazené na každej stránke
- [ ] `pnpm dev` beží bez errorov
- [ ] Layout je responzívny (na mobile sa nič nerozbije — skús DevTools → Responsive mode)

## Tipy a riešenia problémov

**Problém:** Po `pnpm dlx shadcn@latest init` mi to hodí error o Tailwinde
**Riešenie:** Skontroluj že `pnpm dev` aspoň raz prešiel pred init-om. shadcn potrebuje vidieť
fungujúci Tailwind setup.

**Problém:** Font sa nezobrazuje (vidím defaultný serif font)
**Riešenie:** Skontroluj že `inter.variable` je na `<html>` tag-u (NIE na `<body>`) a že máš
`className="font-sans"` na `<body>`. Tailwind v4 si načíta `--font-sans` CSS premennú a aplikuje.

**Problém:** `container mx-auto` nerobí to čo má (nie je centrované)
**Riešenie:** Tailwind v4 zmenil defaulty pre `container`. Skontroluj že v `globals.css` máš
`@import "tailwindcss";` na začiatku.

## Pýtanie sa Claude Code

- *"Vysvetli mi rozdiel medzi server components a client components v Next.js. Header a footer
  som spravil ako server components — je to OK?"*
- *"Ako môžem ku komponentu shadcn `Button` pridať vlastný variant (napríklad `gold`)?"*
- *"Tailwind v4 vraj funguje inak ako v3 — čo sa zmenilo?"*

## Ďalší krok

➡️ [Task 03 — Public landing page](03-landing-page.md)
