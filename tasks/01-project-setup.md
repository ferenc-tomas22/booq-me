# Task 01 — Project Setup

**Modul:** 1 — Setup & základy
**Čas:** ~30 min
**Obtiažnosť:** ★☆☆☆☆

## Čo sa naučíš

- Ako naškálovať novú Next.js appku cez CLI
- Prečo používame `pnpm` namiesto `npm`
- Ako napojiť lokálne repo na GitHub remote
- Ako spustiť dev server a otvoriť appku v prehliadači

## Background — čo je to vlastne `create-next-app`?

`create-next-app` je oficiálny CLI nástroj od Vercel-u, ktorý ti vygeneruje **kostru** Next.js
projektu — všetky konfiguračné súbory, závislosti, `package.json`, prvú stránku, atď.
Bez neho by si musel ručne nainštalovať React, TypeScript, ESLint, Tailwind a nastaviť ich na
seba — to by trvalo hodiny.

### Prečo pnpm a nie npm?

`pnpm` je tretí veľký package manager v JS svete (po `npm` a `yarn`). Robí to isté čo `npm`,
ale:

- **Rýchlejší** — má globálny cache, takže rovnaké balíky neinštaluje znova
- **Šetrí miesto** — používa hardlinky namiesto duplikovania súborov
- **Striktný** — nedovolí ti importovať balíky, ktoré nie sú v `package.json` (`npm` to
  povolí kvôli "hoistingu")

La-fly, ktorú sme videli, používa `pnpm`. Použijeme ho aj my.

## Tvoja úloha

1. **Skontroluj že máš všetko nainštalované:**

   ```bash
   node --version    # malo by byť v20.0.0 alebo vyššie
   pnpm --version    # malo by byť v9.0.0 alebo vyššie
   gh --version      # GitHub CLI
   ```

   Ak `pnpm` nemáš:
   ```bash
   npm install -g pnpm
   ```

2. **Naškáluj nový Next.js projekt** vo svojom Desktope (alebo kdekoľvek inde):

   ```bash
   cd ~/Desktop
   pnpm create next-app@latest booq-me-app
   ```

   CLI sa ťa opýta niekoľko otázok. Odpovedz nasledovne:

   | Otázka | Odpoveď |
   |--------|---------|
   | Would you like to use TypeScript? | **Yes** |
   | Would you like to use ESLint? | **Yes** |
   | Would you like to use Tailwind CSS? | **Yes** |
   | Would you like your code inside a `src/` directory? | **Yes** |
   | Would you like to use App Router? | **Yes** |
   | Would you like to use Turbopack? | **Yes** |
   | Would you like to customize the import alias? | **No** (necháme defaultné `@/*`) |

   > 💡 **Prečo `src/`?** Bez `src/` by ti `app/`, `lib/`, `components/` priečinky ležali
   > rovno v rooti vedľa `package.json`. So `src/` máš všetko pekne pohromade.

3. **Vstúp do priečinka a spusti dev server:**

   ```bash
   cd booq-me-app
   pnpm dev
   ```

   Otvor [http://localhost:3000](http://localhost:3000) v prehliadači. Mal by si vidieť
   defaultnú Next.js privítaciu stránku.

4. **Zastav dev server** (`Ctrl+C` v termináli).

5. **Napoj lokálne repo na náš GitHub repo `booq-me`:**

   Najprv pridaj GitHub remote (URL si vezmi zo svojho repa, pre teba je to
   `https://github.com/ferenc-tomas22/booq-me.git`):

   ```bash
   git remote add origin https://github.com/ferenc-tomas22/booq-me.git
   ```

   > ⚠️ Pozor — `create-next-app` ti už urobil `git init`. Ak by si chcel skopírovať tasky
   > z **tohto** repa (kde máš `tasks/` a `README.md`), potrebuješ ich zmergovať s tvojou novou
   > Next.js appkou. To je trošku komplikované. Jednoduchší prístup: prekopíruj si subory
   > `tasks/` + `README.md` ručne do tvojho nového projektu (alebo neprekopiruj a nechaj ich
   > v separate repe — ako ti vyhovuje).

6. **Skontroluj že všetko sedí:**

   ```bash
   git status            # mal by vidieť všetky súbory ako modified/untracked
   git remote -v         # mal by vidieť origin → github.com/ferenc-tomas22/booq-me.git
   pnpm dev              # appka stále musí fungovať
   ```

7. **Prvý commit a push:**

   ```bash
   git add .
   git commit -m "task 01: initial Next.js scaffold"
   git branch -M main
   git push -u origin main
   ```

   Otvor GitHub repo v prehliadači — uvidíš svoje súbory.

## Acceptance Criteria

- [ ] V termináli funguje `node --version` (v20+) a `pnpm --version` (v9+)
- [ ] Príkaz `pnpm dev` v priečinku `booq-me-app/` spustí dev server na porte 3000
- [ ] `http://localhost:3000` zobrazuje Next.js stránku (alebo už ňou upravenú, ak si experimentoval)
- [ ] `git remote -v` ukazuje `origin` smerujúci na tvoj GitHub repo
- [ ] Prvý commit je pushnutý na `main` branch
- [ ] Na GitHube vidíš svoje súbory

## Tipy a riešenia problémov

**Problém:** `command not found: pnpm`
**Riešenie:** Spusti `npm install -g pnpm` a otvor nový terminál.

**Problém:** `Error: EACCES: permission denied` pri `npm install -g`
**Riešenie:** Buď použiješ `sudo` (rýchle ale "škaredé"), alebo si naištaluješ `nvm` a budeš
mať Node.js v user-space. Pýtaj sa Claude: *"Ako nainštalujem nvm na Macu?"*

**Problém:** `git push` ti pýta heslo a nepustí ťa dnu
**Riešenie:** GitHub od 2021 nepoužíva heslá pre git operácie. Buď máš nastavený **SSH key**,
alebo **personal access token**, alebo `gh CLI`. Najjednoduchšie: `gh auth login` ti to
vyrieši v 30 sekundách.

**Problém:** Dev server hodí error `Error: listen EADDRINUSE: address already in use :::3000`
**Riešenie:** Niečo iné už beží na porte 3000. Buď to zastav, alebo spusti dev server na
inom porte: `pnpm dev --port 3001`.

## Pýtanie sa Claude Code

Dobré prompty pre tento task:

- *"Vysvetli mi, čo je rozdiel medzi App Routerom a Pages Routerom v Next.js."*
- *"Po `pnpm create next-app` mi vznikol súbor `next.config.ts`. Čo robí každý kľúč v ňom?"*
- *"Vysvetli mi obsah súboru `src/app/layout.tsx` a `src/app/page.tsx` — čo sa stane keď otvorím
  `localhost:3000`?"*

## Ďalší krok

➡️ [Task 02 — Tailwind v4 + shadcn/ui + base layout](02-tailwind-shadcn-layout.md)
