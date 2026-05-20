# booq-me — Rezervačný systém pre kaderníctvo

Reálny end-to-end projekt: postav webovú aplikáciu s rezervačným systémom pre kaderníctvo
**Samuela Agoštona**, postupne, krok za krokom, od prázdneho priečinka až po nasadenú
aplikáciu na vlastnej doméne.

Inšpirácia: [booqme.sk/sk/rezervacia/mens-hub](https://booqme.sk/sk/rezervacia/mens-hub).

---

## Čo postavíš

**Public časť** — to, čo vidí zákazník:
- Landing page s informáciami o salóne, službami, kontaktom
- Rezervačný flow: vyber službu → vyber dátum a čas → zadaj kontaktné údaje → potvrdenie
- Po rezervácii dostane zákazník potvrdzovací email

**Admin časť** — to, čo vidí Samuel:
- Login (email + heslo)
- Prehľad rezervácií (dnes / nadchádzajúce)
- Správa služieb (pridať/upraviť/zmazať)
- Blokovanie termínov (dovolenka, sviatky)
- Email notifikácia pri novej rezervácii

**Deploy:** appka beží na [Vercel](https://vercel.com) zdarma, dáta v [Neon Postgres](https://neon.tech) zdarma.

---

## Stack

| Vrstva | Technológia | Prečo |
|--------|-------------|-------|
| Framework | **Next.js 16** (App Router) | Moderný React framework so server components |
| Jazyk | **TypeScript** | Typová kontrola = menej bugov |
| Package manager | **pnpm** | Rýchlejší a úspornejší ako npm |
| Styling | **Tailwind CSS v4** + **shadcn/ui** | Utility CSS + krásne pripravené komponenty (vyberáš si tému!) |
| Databáza | **Neon Postgres** | Serverless Postgres zdarma |
| ORM | **Drizzle** | TypeScript-first, schema-as-code |
| Auth | **Better Auth** | Moderný auth knižnica pre Next.js |
| Email | **Resend** + React Email | Jednoduché API pre transakčné emaily |
| Validácia | **Zod** | Schema-based validácia inputov |
| Forms | **react-hook-form** | Performantné formuláre |
| Dátumy / TZ | **date-fns** + **date-fns-tz** | Bezpečné narábanie s časovými zónami |
| Deploy | **Vercel** | Zero-config deploy Next.js apiek |

---

## Predpoklady

Predtým než začneš, mal by si:

- ✅ Mať dokončené `learn-to-code` curriculum (HTML, CSS, JS, React, Next.js basics)
- ✅ Vedieť používať Git (commit, branch, push)
- ✅ Mať nainštalované:
  - **Node.js 20+** ([nodejs.org](https://nodejs.org))
  - **pnpm** (`npm install -g pnpm`)
  - **VS Code** alebo **Cursor**
  - **Claude Code** ([dokumentácia](https://docs.anthropic.com/en/docs/claude-code))
  - **gh CLI** ([cli.github.com](https://cli.github.com)) — pre prácu s GitHub
- ✅ Mať účty:
  - [GitHub](https://github.com) ✓ (už máš)
  - [Neon](https://neon.tech) (vytvoríš v Tasku 04)
  - [Resend](https://resend.com) (vytvoríš v Tasku 13)
  - [Vercel](https://vercel.com) (vytvoríš v Tasku 14)

---

## Ako pracovať s týmto repom

1. **Čítaj tasky v poradí** — každý ďalší stavia na predošlom.
2. **Otvor si `tasks/01-project-setup.md`** a urob čo je v ňom napísané.
3. **Po každom tasku commitni** svoju prácu (`git add . && git commit -m "task NN: ..."`).
4. **Keď sa zasekneš na 15 minút** — pozri do sekcie "Tipy a riešenia problémov" v danom tasku.
5. **Keď ani to nepomôže** — pýtaj sa Claude Code. V každom tasku máš odporúčané prompty.
6. **Nikdy len nekopíruj kód od Claude.** Vysvetli si ho v hlave. Zmeň premennú. Skús
   niečo pokaziť a pozri sa, čo sa stane. Tak sa učíš.

---

## Roadmap

### Modul 1 — Setup & základy

| # | Task | Čas | Čo postavíš |
|---|------|-----|-------------|
| 01 | [Project setup](tasks/01-project-setup.md) | 60–90 min | Next.js appku s **tvojou témou** (shadcn.com/create) napojenou na GitHub |
| 02 | [Tailwind + shadcn/ui + base layout](tasks/02-tailwind-shadcn-layout.md) | 45 min | Hlavičku, pätu, font, základný layout |
| 03 | [Public landing page](tasks/03-landing-page.md) | 60 min | Hero, "Naše služby", kontakt + cena formatter |

### Modul 2 — Databáza

| # | Task | Čas | Čo postavíš |
|---|------|-----|-------------|
| 04 | [Neon DB + env vars](tasks/04-neon-database.md) | 30 min | Pripojenú prázdnu DB s env premennými |
| 05 | [Drizzle schema](tasks/05-drizzle-schema.md) | 60–90 min | Tabuľky `services`, `bookings`, `blocked_slots` (cena v centoch) |
| 06 | [Seed data](tasks/06-seed-data.md) | 30 min | Naplnenú DB so 5 službami |

### Modul 3 — Rezervačný flow (public)

| # | Task | Čas | Čo postavíš |
|---|------|-----|-------------|
| 07 | [Booking — vyber služby](tasks/07-booking-service-picker.md) | 45 min | Krok 1 flow-u: zoznam služieb z DB |
| 08 | [Booking — kalendár + čas](tasks/08-booking-date-time.md) | 90–120 min | Krok 2: kalendár s voľnými slotmi (Europe/Bratislava timezone) |
| 09 | [Booking — formulár + uloženie](tasks/09-booking-customer-form.md) | 60–90 min | Krok 3: validácia, race-condition guard, uloženie do DB |

### Modul 4 — Admin

| # | Task | Čas | Čo postavíš |
|---|------|-----|-------------|
| 10 | [Better Auth setup](tasks/10-better-auth-setup.md) | 60–90 min | Login pre Samuela, chránené `/admin` routy |
| 11 | [Admin — zoznam rezervácií](tasks/11-admin-bookings-list.md) | 45–60 min | Tabuľka rezervácií (od teraz + filtre) |
| 12 | [Admin — služby + blokovanie](tasks/12-admin-services-and-blocks.md) | 60–90 min | CRUD služieb + blokovanie dovolenky |

### Modul 5 — Emaily a deploy

| # | Task | Čas | Čo postavíš |
|---|------|-----|-------------|
| 13 | [Email notifikácie cez Resend](tasks/13-resend-emails.md) | 45–60 min | Potvrdzovací email zákazníkovi + notifikácia Samuelovi |
| 14 | [Deploy na Vercel](tasks/14-deploy-to-vercel.md) | 75–90 min | Appka beží na verejnej URL + admin password reset |

**Celkový čas:** ~14–18 hodín čistej práce. S premýšľaním, googlením a debugovaním reálne
**30–50 hodín** (a to je dobre — učenie nemá byť rýchle).

---

## Mimo scope kurzu — pridáš si neskôr

Toto MVP zámerne necieli na "production-ready komerčný produkt" — cieli na **funkčný
end-to-end zážitok** ktorý vieš postaviť za pár dní. Tieto veci by sa pridali keď máš
stabilný feature set:

| Vec | Prečo neskôr |
|---|---|
| **Multi-staff** (Samuel + Robert) | Single-staff je o krok jednoduchší flow; rozšírenie je čistý DB refactor |
| **Online platba** (Stripe) | Vyžaduje webhooky, refundy, fraud handling — vlastný projekt |
| **SMS notifikácie** | Twilio integration + náklady na SMS; email pokryje 90 % use case |
| **Recenzie zákazníkov** | Pridáš po prvých 50 rezerváciách keď máš zákazníkov |
| **Unit + E2E testy** | Pridáš keď máš stabilný feature set; teraz by ti len brzdili iterácie |
| **Accessibility audit** | shadcn/ui má a11y základ; full audit pred verejným launchom |
| **Git branching/PR workflow** | Solo dev s jednou environmentou; tím + staging by si pýtal PR-y |
| **CI/CD pipeline** | Vercel auto-deploy stačí; vlastný CI keď budeš mať testy |
| **Rate limiting** | Open booking je riziko spamu; pridaj cez Upstash alebo Vercel Edge Config po launchi |

---

## Tipy na úspech

- **Commituj často.** Nie raz za task, ale po každom logickom kroku. Keď to pokazíš, vieš
  sa jednoducho vrátiť.
- **Čítaj error messages.** Next.js, TypeScript a Postgres ti hovoria veľmi konkrétne, čo
  sa pokazilo. Skús to najprv prečítať a pochopiť, až potom googli.
- **Otvor browser DevTools.** Network tab, Console tab, React DevTools — to sú tvoje oči
  do toho, čo sa naozaj deje.
- **Drizzle Studio (`pnpm db:studio`)** ti ukáže obsah DB v krásnej UI. Používaj ho hojne.
- **Nepreskakuj tasky.** Ak vynecháš Task 04, v Tasku 05 budeš mať schema bez DB. Funguje
  to ako reťaz.
- **Pýtaj sa Claude Code, ale s rozmyslom.** "Hej Claude, napíš mi toto" je zlý prompt.
  Lepšie: "Mám tento error `[paste]`, snažil som sa `[čo som skúsil]`, čo môže byť zle?"

---

## Užitočné odkazy

- [Next.js docs](https://nextjs.org/docs) — oficiálna dokumentácia
- [Tailwind v4 docs](https://tailwindcss.com/docs)
- [shadcn/ui](https://ui.shadcn.com/) — komponenty + [Theme Studio](https://ui.shadcn.com/create)
- [Drizzle docs](https://orm.drizzle.team/docs/overview)
- [Better Auth docs](https://www.better-auth.com/docs)
- [Neon docs](https://neon.tech/docs)
- [Resend docs](https://resend.com/docs)
- [Vercel docs](https://vercel.com/docs)

---

## Licencia

MIT — robte si s tým čo chcete. Pôvodný účel: učenie sa.
