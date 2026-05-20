# Task 14 — Deploy na Vercel

**Modul:** 5 — Emaily a deploy
**Čas:** ~60 min
**Obtiažnosť:** ★★★★★

## Čo sa naučíš

- Ako sa **deployne** Next.js appka na Vercel cez git
- Ako sa nastavujú **environment variables** pre production
- Ako zmigrujeme z `drizzle-kit push` na `drizzle-kit migrate` (production-safe)
- Ako sa pripojí **custom domain** (voliteľné)
- Ako sa monitoruje appka po deployi

## Background

### Čo Vercel robí pod kapotou?

Vercel je platforma od tvorcov Next.js. Keď pushneš commit na GitHub:

1. Vercel detekuje push cez **webhook**
2. Naclonuje tvoj repo
3. Spustí `pnpm install` + `pnpm build`
4. Zoberie build output (`.next/`) a uloží na **CDN**
5. Nasadí **serverless functions** pre API routes a server actions
6. Aktivuje novú verziu — staré requesty dobehnu na starej, nové na novej

Pre teba: zero config, len pushneš a beží.

### Migrácie v production

V dev sme používali `drizzle-kit push` — to porovná schemu s DB a urobí zmeny "na mieste".
Super pre rýchle iterovanie, **nebezpečné pre production**:

- Žiadne migration history → nevieš revertnúť
- Žiadne ručné review → môže ti zmazať stĺpec s dátami

Pre production prejdeme na klasický flow:

1. `pnpm db:generate` → vytvorí SQL migration file v `src/db/migrations/`
2. Reviewneme SQL (open `0000_...sql` a prečítaj)
3. `pnpm db:migrate` → aplikuje migráciu (s migration history)
4. Pri deployi na Vercel: migration sa spustí pred buildom

## Tvoja úloha

### 1. Vygeneruj prvú migration

V Drizzle dev móde sme používali `push`. Teraz prejdime na migráciu:

```bash
# Najprv resetni Neon DB ak chceš čistý stav (voliteľné):
# V Neon Console → SQL Editor → spusti:
# DROP SCHEMA public CASCADE; CREATE SCHEMA public;

pnpm db:generate
```

Drizzle ti v `src/db/migrations/` vytvorí súbor `0000_xxx.sql` so všetkými `CREATE TABLE`
príkazmi. Otvor ho a prejdi si ho — toto sa spustí na production DB.

### 2. Pridaj `db:migrate` skript (ak ešte nemáš)

V `package.json`:

```json
{
  "scripts": {
    "db:migrate": "tsx --env-file=.env.local src/db/migrate-runner.ts"
  }
}
```

A vytvor `src/db/migrate-runner.ts`:

```ts
import { migrate } from 'drizzle-orm/postgres-js/migrator';
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';

const run = async () => {
  if (!process.env.DATABASE_URL) throw new Error('DATABASE_URL not set');

  const client = postgres(process.env.DATABASE_URL, { max: 1 });
  const db = drizzle(client);

  console.log('🚀 Running migrations...');
  await migrate(db, { migrationsFolder: './src/db/migrations' });
  console.log('✅ Migrations done.');

  await client.end();
  process.exit(0);
};

run().catch((err) => {
  console.error('❌ Migration failed:', err);
  process.exit(1);
});
```

### 3. Vytvor Neon production branch

V Neon Console:

1. Choď do svojho projektu `booq-me`
2. **Branches → Create branch**
3. Pomenuj `prod`, parent = `main`
4. Skopíruj nový **connection string** pre `prod` branch (bude potrebný v ďalšom kroku)

> 💡 **Prečo branch?** `main` použiješ pre dev (môžeš tam experimentovať, resetovať). `prod`
> je tvoj production — len Vercel zaň hovorí.

### 4. Spravi prvý deploy na Vercel

1. Choď na [vercel.com](https://vercel.com) → Sign Up cez GitHub
2. **Add New → Project**
3. Vyber `ferenc-tomas22/booq-me` z listu
4. **Framework preset**: Next.js (auto-detekuje)
5. **Root directory**: ak máš Next.js v `booq-me-app/` subfoldri, nastav to. Ak je v rooti, nechaj prázdne.
6. **Environment Variables** — pridaj všetko z `.env.local`:
   - `DATABASE_URL` → connection string z **prod branch** Neonu
   - `BETTER_AUTH_SECRET` → vygeneruj nový cez `openssl rand -base64 32`
   - `BETTER_AUTH_URL` → zatiaľ daj `https://booq-me.vercel.app` (URL prispôsobíš po prvom deployi)
   - `NEXT_PUBLIC_APP_URL` → rovnaké ako `BETTER_AUTH_URL`
   - `RESEND_API_KEY`
   - `RESEND_FROM_EMAIL` → `onboarding@resend.dev` zatiaľ
   - `ADMIN_EMAIL` → tvoj/Samuelov real email
7. **Deploy**!

Po ~2 min: appka beží na `https://booq-me-xxx.vercel.app`.

### 5. Aplikuj migrácie na production DB

Vercel **automaticky nespúšťa migrácie** — musíš to urobiť explicitne. Najjednoduchšie:
lokálne, s `DATABASE_URL` nastaveným na prod connection:

```bash
# Skopíruj prod connection string do shell premennej
export DATABASE_URL="postgres://...prod...neon.tech/..."

# Spusti migrácie
pnpm db:migrate
```

(Po skončení **unset** premennú alebo zatvor terminál — aby si si neomylom nepushuvál na prod.)

> 💡 **Lepšia varianta pre tím**: pridaj build command na Vercel:
> `pnpm db:migrate && pnpm build`. Drobnosť: pri každom deployi sa migrácie spustia. Pre náš
> case zatiaľ stačí manuálne.

### 6. Vytvor admin usera v production

```bash
export DATABASE_URL="postgres://...prod..."
pnpm db:seed
```

Toto vytvorí 5 služieb + admin usera v prod DB.

> ⚠️ **POZOR**: defaultné heslo `samuel123` je v `seed.ts` hardcoded. Po prvom prihlásení
> **HNEĎ ho zmeňte** (ideálne cez Better Auth API priamo, alebo do schemy pridáte "change
> password" formulár pre admin).

### 7. Otestuj production

Otvor `https://booq-me-xxx.vercel.app`:

- Landing page ✓
- Rezervačný flow (vyber službu → dátum → čas → vyplň → submit) ✓
- Skontroluj email že prišiel ✓
- Prihlás sa do `/admin/login` ✓
- Vidíš rezerváciu v `/admin` ✓

Ak niečo nefunguje:

- **Vercel → Project → Logs** → uvidíš runtime errory
- **Vercel → Project → Deployments → [najnovší] → View Function Logs** → server action logy

### 8. (Voliteľne) Vlastná doména

Ak máš doménu (napr. `samuel-barber.sk`):

1. Vercel → Project → Settings → Domains → Add → `samuel-barber.sk`
2. Vercel ti ukáže DNS records — pridaj ich u svojho domain registrara
3. Počkaj 5–60 min na propagáciu DNS
4. Aktualizuj env vars:
   - `BETTER_AUTH_URL=https://samuel-barber.sk`
   - `NEXT_PUBLIC_APP_URL=https://samuel-barber.sk`
5. **Redeploy** (Vercel → Deployments → 3-dot menu → Redeploy)

A overuj doménu v Resende:

1. Resend → Domains → Add → `samuel-barber.sk`
2. Pridaj DNS records (DKIM, SPF) u registrara
3. Po overení zmeň `RESEND_FROM_EMAIL=rezervacia@samuel-barber.sk`

### 9. Cleanup `.env.local.example`

Aktualizuj `.env.local.example` aby obsahoval **všetky** premenné, ktoré appka potrebuje:

```bash
# Database
DATABASE_URL=""

# Better Auth
BETTER_AUTH_SECRET=""
BETTER_AUTH_URL="http://localhost:3000"

# Public URLs
NEXT_PUBLIC_APP_URL="http://localhost:3000"

# Email
RESEND_API_KEY=""
RESEND_FROM_EMAIL="onboarding@resend.dev"
ADMIN_EMAIL=""
```

### 10. Posledný commit a push

```bash
git add .
git commit -m "task 14: production deployment with migrations and Vercel setup"
git push
```

Vercel automaticky redeploy-ne.

## Acceptance Criteria

- [ ] `src/db/migrations/0000_...sql` existuje (migration súbor)
- [ ] `src/db/migrate-runner.ts` existuje a `pnpm db:migrate` funguje
- [ ] Neon má 2 branche: `main` (dev) a `prod`
- [ ] Vercel projekt je vytvorený a appka beží na `https://booq-me-xxx.vercel.app`
- [ ] Production DB má všetky tabuľky + 5 služieb + admin usera
- [ ] Rezervačný flow funguje na production
- [ ] Emaily prichádzajú (v Resend dashboarde vidíš production sends)
- [ ] Admin login funguje na production
- [ ] `.env.local.example` obsahuje všetky env vars

## Bonus

- Pridaj **Vercel Analytics** (Vercel → Project → Analytics → Enable) — zadarmo
- Nastav **Sentry** pre error tracking (free tier 5k errorov/mesiac)
- Pridaj custom doménu a SSL (zadarmo cez Let's Encrypt, Vercel to robí auto)
- Nastav **Slack webhook** v Vercele aby ťa notifikoval pri každom deployi

## Tipy a riešenia problémov

**Problém:** Build na Vercel padne s `Module not found: Can't resolve '@/db'`
**Riešenie:** Skontroluj `tsconfig.json` že má `"paths": { "@/*": ["./src/*"] }`. Aj
`next.config.ts` musí byť OK.

**Problém:** Build prejde, ale runtime error: `DATABASE_URL is not set`
**Riešenie:** Env vars v Vercel sa neaplikujú retroaktívne. Pridaj env var → musíš
**redeploy-núť** (3-dot menu → Redeploy).

**Problém:** `Error: connect ETIMEDOUT` z Vercel na Neon
**Riešenie:** Neon free tier má **suspend** mode — po 5 min nečinnosti sa DB vypne. Prvý
request po wake-up trvá 1-2s. Vercel server function má timeout 10s, takže by mal stihnúť.
Ak nie, upgradni Neon na "Always-on" alebo skús znova.

**Problém:** Emaily z production nechodia
**Riešenie:**
1. Skontroluj že `RESEND_API_KEY` v Vercel env vars je správny
2. Resend dashboard → Emails — vidíš tam pokus o odoslanie?
3. Ak vidíš "Domain not verified" — používaš svoju doménu ale neni overená. Vráť sa na
   `onboarding@resend.dev` alebo dokonči overenie domény.

**Problém:** Admin login na production hodí "Invalid email or password" hoci heslo je správne
**Riešenie:** Skontroluj že **`BETTER_AUTH_URL` v Vercel env vars je správna production URL**
(NIE `http://localhost:3000`). Better Auth porovnáva URL pri validácii sessions.

## Pýtanie sa Claude Code

- *"Vysvetli mi rozdiel medzi Vercel Edge Functions a Serverless Functions. Čo z toho beží
  na Next.js routes by default?"*
- *"Moja appka na Vercele má cold start ~2 sekundy. Ako to znížiť?"*
- *"Chcem nastaviť **preview deployments** — keď urobím PR, dostanem unique URL na otestovanie.
  Ako sa to robí?"*

## Hotovo! 🎉

Práve si nasadil reálnu web aplikáciu. Samuel môže prijímať rezervácie cez web a ty máš
v ruke:

- Funkčnú Next.js + Postgres + Auth + Email appku
- Hosting zdarma (Vercel + Neon + Resend free tier)
- Skill set ktorý ťa berie do akéhokoľvek modern web stacku

**Čo ďalej?**

- **Pridaj multi-staff** (Samuel + Robert) — rozšír schemu o `staff` tabuľku a uprav booking flow
- **Pridaj online platbu** cez Stripe — overuje rezerváciu zaplatením
- **Pridaj SMS reminder** deň pred termínom (Twilio + cron na Vercele)
- **Pridaj recenzie** zákazníkov (post-booking email s linkom na hodnotenie)
- **Pridaj kalendár export** — booking pošli ako `.ics` súbor

**Gratulujem!** 🚀
