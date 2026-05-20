# Task 01 — Project Setup

**Modul:** 1 — Setup & základy
**Čas:** ~45–60 min
**Obtiažnosť:** ★☆☆☆☆

## Čo sa naučíš

- Ako rozhodnúť o vzhľade a "feel-u" appky cez **shadcn.com/create** UI
- Prečo používame `pnpm` namiesto `npm`
- Ako naškálovať Next.js projekt **s vlastnou témou** jedným príkazom
- Ako spustiť dev server a otvoriť appku v prehliadači

## Background — táto appka je tvoja

Než začneš písať kód, musíš urobiť pár rozhodnutí o tom, **ako bude appka vyzerať**. Nemáš
sa čo báť rozhodnúť zle — všetko sa dá neskôr zmeniť. Ale chcem aby si **vedome vybral**,
nie aby si len kopíroval to čo ti niekto povie.

shadcn.com má geniálny "Theme Studio" na adrese [ui.shadcn.com/create](https://ui.shadcn.com/create),
kde si v live UI **klikneš čo chceš** — vyberieš template, štýl, farby, fonty, radius —
a appka ti vygeneruje **jednu inštaláciu**, ktorá ti to celé pripraví.

### Prečo pnpm a nie npm?

`pnpm` je tretí veľký package manager v JS svete (po `npm` a `yarn`). Robí to isté čo `npm`,
ale:

- **Rýchlejší** — má globálny cache, takže rovnaké balíky neinštaluje znova
- **Šetrí miesto** — používa hardlinky namiesto duplikovania súborov
- **Striktný** — nedovolí ti importovať balíky, ktoré nie sú v `package.json`

## Tvoja úloha

### 1. Skontroluj prerekvizity

```bash
node --version    # malo by byť v20.0.0 alebo vyššie
pnpm --version    # malo by byť v9.0.0 alebo vyššie
gh --version      # GitHub CLI
```

Ak `pnpm` nemáš:
```bash
npm install -g pnpm
```

Ak `gh` nemáš: [cli.github.com](https://cli.github.com) → install → `gh auth login`.

### 2. Naclonuj prázdne repo `booq-me`

```bash
cd ~/Desktop
gh repo clone ferenc-tomas22/booq-me
cd booq-me
ls
```

Mal by si vidieť `README.md` + priečinok `tasks/`. **V tomto priečinku vygenerujeme appku** —
takže `package.json`, `src/`, `next.config.ts` pôjdu vedľa `tasks/`.

### 3. Hraj sa s ui.shadcn.com/create

Otvor [ui.shadcn.com/create](https://ui.shadcn.com/create) v prehliadači.

Vľavo máš panel s nastaveniami. Pohraj sa ~15 minút:

| Nastavenie | Tvoja voľba | Čo to ovplyvní |
|---|---|---|
| **Style** | Nova / Default / New York / ... | Celkový vzhľad komponentov (zaokrúhlenie, padding, tieny) |
| **Base Color** | Neutral / Stone / Zinc / Gray / Slate | Tón sivých — neutrálne pozadie a borders |
| **Theme** | Neutral / hociaký color | Akcentová farba (tlačidlá, focus rings) |
| **Heading** font | Inter / Geist / ... | Font pre nadpisy |
| **Font** | Inter / Geist / ... | Font pre body text |
| **Icon Library** | Lucide / Radix / Tabler | Štýl ikon |
| **Radius** | 0 / 0.3 / 0.5 / 0.75 / 1.0 | Zaokrúhlenie rohov (0 = ostré, 1 = okrúhle) |

Klikni **Shuffle** zopár krát aby si videl rôzne kombinácie. Skús si v hlave predstaviť:
*"vyzeral by takto pánsky barber salón profesionálne?"*

> 💡 **Pre kaderníctvo:** odporúčam tmavú, mužnú paletu — napríklad Style: **Nova**, Base
> Color: **Neutral** alebo **Stone**, Theme: **Neutral** alebo **Orange** (akcent), Heading
> font: **Geist** alebo **Inter**, Radius: **0.5**. Ale rozhodni ty.

### 4. Vyber template a base

Hore klikni **New Project** tab. Vyber:

- **Template:** Next.js
- **Base:** Radix UI
- Toggles: všetky nechaj OFF (žiadny monorepo, žiadne RTL)

### 5. Skopíruj command a spusti ho v termináli

Dole uvidíš command typu:

```bash
pnpm dlx shadcn@latest init --preset b0 --template next --base radix
```

Kde `b0` je tvoj preset s tvojimi farbami/fontami.

> ⚠️ **DÔLEŽITÉ:** zvoľ **pnpm** záložku (nie npm/yarn/bun).

Skopíruj príkaz a spusti ho **v priečinku `~/Desktop/booq-me/`**:

```bash
cd ~/Desktop/booq-me
# vlož svoj príkaz tu, napríklad:
pnpm dlx shadcn@latest init --preset b0 --template next --base radix
```

CLI sa ťa opýta:

| Otázka | Odpoveď |
|--------|---------|
| What is your project name? | `.` (bodka = aktuálny priečinok) |
| Use TypeScript? | **Yes** |
| Use App Router? | **Yes** |
| Use `src/` directory? | **Yes** |
| Customize import alias? | **No** |

Inštalácia trvá ~1-2 min. Vytvorí ti:
- `package.json` so všetkými dependencies (Next.js, React, Tailwind v4, shadcn dependencies)
- `src/app/` so základnou stránkou
- `src/components/ui/` s prvými shadcn komponentmi v **tvojej téme**
- `components.json` (config shadcn)
- `globals.css` s `@theme` blokom obsahujúcim tvoje farby a fonty

### 6. Spusti dev server

```bash
pnpm dev
```

Otvor [http://localhost:3000](http://localhost:3000). Mal by si vidieť defaultnú Next.js
stránku — ale už **v tvojej téme** (font, farby, radius).

### 7. Skontroluj git stav

```bash
git status
git remote -v
```

`origin` by mal smerovať na `https://github.com/ferenc-tomas22/booq-me.git` (pretože si
naklonoval cez `gh repo clone`).

Mal by si vidieť všetky nové súbory ako "untracked" — to je správne.

### 8. Pridaj `.env.local` do `.gitignore`

Otvor `.gitignore` a over že tam je tento riadok:

```
.env*.local
```

Ak nie, **pridaj ho hneď**. Spustíme to v Tasku 04 keď budeme mať databázu — dovtedy nemáš
žiadne tajné kľúče.

### 9. Prvý commit a push

```bash
git add .
git commit -m "task 01: scaffold Next.js app via shadcn.com/create"
git push
```

Otvor GitHub repo v prehliadači — uvidíš svoju appku.

## Acceptance Criteria

- [ ] `node --version` (v20+), `pnpm --version` (v9+), `gh --version` všetko funguje
- [ ] Repo `booq-me` je naklonované do `~/Desktop/booq-me/`
- [ ] V priečinku `booq-me/` máš `package.json`, `src/`, `next.config.ts` (vedľa pôvodných `tasks/` a `README.md`)
- [ ] `pnpm dev` spustí dev server na porte 3000
- [ ] `localhost:3000` ukazuje stránku **v tvojej zvolenej téme** (farby, font, radius zodpovedajú tomu čo si vybral)
- [ ] `components.json` existuje
- [ ] `src/components/ui/` obsahuje aspoň `button.tsx`
- [ ] `.gitignore` obsahuje `.env*.local`
- [ ] Prvý commit je pushnutý na `main` branch a vidíš ho na GitHube

## Tipy a riešenia problémov

**Problém:** `command not found: pnpm`
**Riešenie:** Spusti `npm install -g pnpm` a otvor nový terminál.

**Problém:** `Error: EACCES: permission denied` pri `npm install -g`
**Riešenie:** Buď `sudo` (rýchle ale "škaredé"), alebo si nainštaluj `nvm` a budeš mať
Node.js v user-space. Pýtaj sa Claude: *"Ako nainštalujem nvm na Macu?"*

**Problém:** `pnpm dlx shadcn@latest init ...` hodí error že priečinok nie je prázdny
**Riešenie:** shadcn `init --template` vyžaduje **prázdny** priečinok. Repo má `README.md`
+ `tasks/`. Riešenie: presuň ich dočasne von, spusti init, vráť ich:
```bash
mv README.md tasks ..
pnpm dlx shadcn@latest init --preset b0 --template next --base radix
mv ../README.md ../tasks .
```
Alternatíva: vytvor podpriečinok `app/`, init tam a Vercel root-directory nastavíš `app`.

**Problém:** `git push` ti pýta heslo
**Riešenie:** GitHub od 2021 nepoužíva heslá. Spusti `gh auth login` — vyrieši to za teba.

**Problém:** Dev server hodí `EADDRINUSE: address already in use :::3000`
**Riešenie:** Iný proces ti drží port. Buď ho zastav, alebo `pnpm dev --port 3001`.

## Pýtanie sa Claude Code

Skús sa ho pýtať tieto **konkrétne** otázky (nie všeobecné "ako sa robí Next.js"):

- *"Vysvetli mi, čo robí `pnpm dlx` v porovnaní s `pnpm install`. Kedy ktoré použiť?"*
- *"Po inicializácii mi vznikol súbor `components.json`. Čo robí každý kľúč v ňom?"*
- *"Otvor `src/app/layout.tsx` a `src/app/page.tsx` a vysvetli mi vzťah — čo sa rendruje
  kedy?"*
- *"Vybral som si Style: Nova, Theme: Neutral. Kde v kóde tieto voľby žijú? Ako ich neskôr
  zmením?"*

## Ďalší krok

➡️ [Task 02 — Base layout + komponenty](02-tailwind-shadcn-layout.md)
