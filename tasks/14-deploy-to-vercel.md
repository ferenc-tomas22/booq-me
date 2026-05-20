# Task 14 — Deploy na Vercel

**Modul:** 5 — Emaily a deploy
**Čas:** ~75–90 min
**Obtiažnosť:** ★★★★★

## Čo sa naučíš

- Ako sa **deployne** Next.js appka na Vercel cez git
- Ako sa nastavujú **environment variables** pre production
- Ako zmigrujeme z `drizzle-kit push` na `drizzle-kit migrate` (production-safe)
- Ako zmeniť admin heslo cez CLI script
- Ako sa pripojí **custom domain** (voliteľné)

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
2. Reviewneme SQL (otvor `0000_...sql` a prečítaj)
3. `pnpm db:migrate` → aplikuje migráciu (s migration history)

## Tvoja úloha

### 1. Vygeneruj prvú migration

```bash
pnpm db:generate
```

Drizzle ti v `src/db/migrations/` vytvorí súbor `0000_xxx.sql` so všetkými `CREATE TABLE`
príkazmi. Otvor ho a prejdi si ho — toto sa spustí na production DB.

### 2. Vytvor `src/db/migrate-runner.ts`

```ts
import { drizzle } from 'drizzle-orm/postgres-js';
import { migrate } from 'drizzle-orm/postgres-js/migrator';
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

(Skript `db:migrate` v `package.json` ho už volá — pridali sme ho v Tasku 05.)

### 3. Vytvor admin password reset script

V `src/db/admin-reset-password.ts`:

```ts
import { auth } from '@/lib/auth';

const run = async () => {
  const [email, newPassword] = process.argv.slice(2);

  if (!email || !newPassword) {
    console.error('Usage: pnpm admin:reset-password <email> <new-password>');
    process.exit(1);
  }

  if (newPassword.length < 8) {
    console.error('❌ Heslo musí mať aspoň 8 znakov.');
    process.exit(1);
  }

  console.log(`🔐 Reseting password for ${email}...`);

  try {
    // Better Auth admin API: nastavi nove heslo pre usera podla emailu.
    // Vytahneme user ID najprv:
    const { db } = await import('@/db');
    const { user } = await import('@/db/schema');
    const { eq } = await import('drizzle-orm');

    const [u] = await db.select().from(user).where(eq(user.email, email));
    if (!u) {
      console.error(`❌ User ${email} not found.`);
      process.exit(1);
    }

    // Better Auth Context API
    const ctx = await auth.$context;
    const hash = await ctx.password.hash(newPassword);
    const { account } = await import('@/db/schema');
    await db
      .update(account)
      .set({ password: hash, updatedAt: new Date() })
      .where(eq(account.userId, u.id));

    console.log('✅ Heslo zmenene.');
    process.exit(0);
  } catch (err) {
    console.error('❌ Reset zlyhal:', err);
    process.exit(1);
  }
};

run();
```

Pridaj skript do `package.json`:

```json
{
  "scripts": {
    "admin:reset-password": "tsx --env-file=.env.local src/db/admin-reset-password.ts"
  }
}
```

Otestuj lokálne (zatiaľ na local DB):

```bash
pnpm admin:reset-password samuel@booq-me.local moje-nove-heslo-123
```

Skús sa prihlásiť do `/admin/login` s novým heslom — malo by fungovať.

### 4. Vytvor Neon production branch

V Neon Console:

1. Choď do svojho projektu `booq-me`
2. **Branches → Create branch**
3. Pomenuj `prod`, parent = `main`
4. Skopíruj nový **connection string** pre `prod` branch

> 💡 **Prečo branch?** `main` použiješ pre dev (môžeš experimentovať, resetovať). `prod`
> je tvoj production — len Vercel zaň hovorí.

### 5. Sprav prvý deploy na Vercel (s placeholder env vars)

1. Choď na [vercel.com](https://vercel.com) → Sign Up cez GitHub
2. **Add New → Project**
3. Vyber `ferenc-tomas22/booq-me` z listu
4. **Framework preset**: Next.js (auto-detekuje)
5. **Root directory**: nechaj prázdne (Next.js je v rooti)
6. **Environment Variables** — pridaj **placeholder hodnoty** (po prvom deployi ich
   upravíme keď budeme vedieť real URL):
   - `DATABASE_URL` → connection string z **prod branch** Neonu
   - `BETTER_AUTH_SECRET` → vygeneruj nový cez `openssl rand -base64 32`
   - `BETTER_AUTH_URL` → `https://booq-me.vercel.app` (PLACEHOLDER — opravíme za chvíľu)
   - `NEXT_PUBLIC_APP_URL` → `https://booq-me.vercel.app` (PLACEHOLDER)
   - `RESEND_API_KEY`
   - `RESEND_FROM_EMAIL` → `onboarding@resend.dev` zatiaľ
   - `ADMIN_EMAIL` → tvoj/Samuelov real email
7. **Deploy**!

Po ~2 min: appka beží na `https://booq-me-xxx.vercel.app`. **Skopíruj si tento URL.**

### 6. Oprav env vars na real URL a redeploy

Toto je **najčastejšia chyba** ktorú deveri robia. Login nebude fungovať kým `BETTER_AUTH_URL`
nezosúhlasí s real production URL.

1. Vercel → Project → **Settings → Environment Variables**
2. Aktualizuj `BETTER_AUTH_URL` na skutočnú URL (napr. `https://booq-me-xxx.vercel.app`)
3. Aktualizuj `NEXT_PUBLIC_APP_URL` rovnako
4. **Redeploy** — Vercel → Deployments → najnovší → 3-dot menu → **Redeploy**

> ⚠️ Env vars sa neaplikujú automaticky na existujúci deploy. Musíš redeploy-núť.

### 7. Aplikuj migrácie na production DB

Vercel **automaticky nespúšťa migrácie**. Najjednoduchšie: lokálne, s `DATABASE_URL` na
prod connection.

```bash
# V novom termináli (aby si si neomilom nemenil .env.local):
export DATABASE_URL="postgres://...prod...neon.tech/..."

# Spusti migrácie
pnpm db:migrate
```

Po skončení **zatvor terminál** (alebo `unset DATABASE_URL`) — aby si si nepushuvál na
prod omylom.

### 8. Vytvor admin usera v production

Stále s prod DB:

```bash
export DATABASE_URL="postgres://...prod...neon.tech/..."
pnpm db:seed
```

Toto vytvorí 5 služieb + admin usera v prod DB.

> ⚠️ **Defaultné heslo `samuel123`** je v `seed.ts`. **Hneď ho zmeň**:
> ```bash
> pnpm admin:reset-password samuel@booq-me.local Tvoje-silne-heslo-123
> ```
> (Stále s `DATABASE_URL` na prod.) Otestuj prihlásenie na `https://booq-me-xxx.vercel.app/admin/login`.

### 9. Otestuj production

Otvor `https://booq-me-xxx.vercel.app`:

- ✓ Landing page funguje (cena `"18,00 €"`)
- ✓ Rezervačný flow (vyber službu → dátum → čas → vyplň → submit)
- ✓ Skontroluj email že prišiel
- ✓ Skontroluj v Resend dashboarde že email letel z **production** (rozliš podľa
  timestampu)
- ✓ Prihlás sa do `/admin/login` s novým heslom
- ✓ Vidíš rezerváciu v `/admin`
- ✓ Skús zrušiť — funguje
- ✓ Skús pridať novú službu — funguje
- ✓ Skús pridať blokovanie na zajtra → vráť sa do `/rezervacia` v incognito → slot je
  zablokovaný

Ak niečo nefunguje:

- **Vercel → Project → Logs** → uvidíš runtime errory
- **Vercel → Project → Deployments → [najnovší] → View Function Logs** → server action logy

### 10. (Voliteľne) Vlastná doména

Ak máš doménu (napr. `samuel-barber.sk`):

1. Vercel → Project → Settings → Domains → Add → `samuel-barber.sk`
2. Vercel ti ukáže DNS records — pridaj ich u svojho domain registrara
3. Počkaj 5–60 min na propagáciu DNS
4. Aktualizuj env vars (Settings → Environment Variables):
   - `BETTER_AUTH_URL=https://samuel-barber.sk`
   - `NEXT_PUBLIC_APP_URL=https://samuel-barber.sk`
5. **Redeploy**

A overuj doménu v Resende:

1. Resend → Domains → Add → `samuel-barber.sk`
2. Pridaj DNS records (DKIM, SPF) u registrara
3. Po overení zmeň `RESEND_FROM_EMAIL=rezervacia@samuel-barber.sk` (v Vercele + Redeploy)

### 11. Cleanup `.env.local.example`

Aktualizuj `.env.local.example` aby obsahoval **všetky** premenné:

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

### 12. Posledný commit a push

```bash
git add .
git commit -m "task 14: production deployment with migrations, admin password reset, Vercel setup"
git push
```

Vercel automaticky redeploy-ne.

## Acceptance Criteria

- [ ] `src/db/migrations/0000_...sql` existuje (migration súbor)
- [ ] `src/db/migrate-runner.ts` existuje a `pnpm db:migrate` funguje
- [ ] `src/db/admin-reset-password.ts` existuje a `pnpm admin:reset-password` funguje lokálne
- [ ] Neon má 2 branche: `main` (dev) a `prod`
- [ ] Vercel projekt je vytvorený a appka beží na production URL
- [ ] `BETTER_AUTH_URL` a `NEXT_PUBLIC_APP_URL` v Vercel env vars sú **real production URL**
- [ ] Production DB má všetky tabuľky + 5 služieb + admin usera
- [ ] Defaultné heslo `samuel123` je v production **ZMENENÉ** cez `admin:reset-password`
- [ ] Rezervačný flow funguje na production end-to-end
- [ ] Emaily prichádzajú (v Resend dashboarde vidíš production sends)
- [ ] Admin login funguje na production
- [ ] `.env.local.example` obsahuje všetky env vars

## Bonus

- Pridaj **Vercel Analytics** (Vercel → Project → Analytics → Enable) — zadarmo
- Nastav **Sentry** pre error tracking (free tier 5k errorov/mesiac)
- Pridaj custom doménu a SSL (zadarmo cez Let's Encrypt, Vercel to robí auto)
- Nastav **Vercel preview deployments** pre PR-y (vyžaduje branch workflow)
- Pridaj **rate limiting** na rezervačný flow (cez `@vercel/edge-config` alebo Upstash)

## Tipy a riešenia problémov

**Problém:** Build na Vercel padne s `Module not found: Can't resolve '@/db'`
**Riešenie:** Skontroluj `tsconfig.json` že má `"paths": { "@/*": ["./src/*"] }`.

**Problém:** Build prejde, ale runtime error: `DATABASE_URL is not set`
**Riešenie:** Env vars v Vercel sa neaplikujú retroaktívne. Pridaj env var → musíš
**redeploy-núť** (3-dot menu → Redeploy).

**Problém:** `Error: connect ETIMEDOUT` z Vercel na Neon
**Riešenie:** Neon free tier má **suspend** mode — po 5 min nečinnosti sa DB vypne. Prvý
request po wake-up trvá 1-2s. Vercel server function má timeout 10s, takže by mal stihnúť.
Ak nie, skús znova.

**Problém:** Emaily z production nechodia
**Riešenie:**
1. Skontroluj že `RESEND_API_KEY` v Vercel env vars je správny
2. Resend dashboard → Emails — vidíš tam pokus o odoslanie?
3. Ak vidíš "Domain not verified" — používaš svoju doménu ale nie je overená. Vráť sa na
   `onboarding@resend.dev` alebo dokonči overenie domény.

**Problém:** Admin login hodí "Invalid email or password" hoci heslo je správne
**Riešenie:** **Najčastejšia chyba** — `BETTER_AUTH_URL` v Vercel env vars je nesprávna.
Skontroluj že je nastavená na real production URL (NIE `http://localhost:3000`). Po
úprave **REDEPLOY**.

**Problém:** Spustil som `pnpm db:seed` s production `DATABASE_URL` a teraz mam dva
"Samuel" v admin tabulke
**Riešenie:** Pred-update seedu (Task 06) — zmaze a recreate-ne business data, ale len
checkuje existenciu admina. Stary admin zostal. Vymaz duplicitu manualne v Drizzle Studio.

## Pýtanie sa Claude Code

- *"Vysvetli mi rozdiel medzi Vercel Edge Functions a Serverless Functions. Čo z toho beží
  na Next.js routes by default?"*
- *"Moja appka na Vercele má cold start ~2 sekundy. Ako to znížiť?"*
- *"Chcem nastaviť **preview deployments** — keď urobím PR, dostanem unique URL. Ako sa
  to robí?"*

## Hotovo! 🎉

Práve si nasadil reálnu web aplikáciu. Samuel môže prijímať rezervácie cez web a ty máš
v ruke:

- Funkčnú Next.js + Postgres + Auth + Email appku
- Hosting zdarma (Vercel + Neon + Resend free tier)
- Skill set, ktorý ťa berie do akéhokoľvek modern web stacku

**Čo ďalej?**

- **Pridaj multi-staff** (Samuel + Robert) — rozšír schemu o `staff` tabuľku a uprav booking flow
- **Pridaj online platbu** cez Stripe — overuje rezerváciu zaplatením
- **Pridaj SMS reminder** deň pred termínom (Twilio + cron na Vercele)
- **Pridaj recenzie** zákazníkov (post-booking email s linkom na hodnotenie)
- **Pridaj kalendár export** — booking pošli ako `.ics` súbor
- **Pridaj rate limiting** aby ti niekto nespamoval booking endpoint

**Gratulujem!** 🚀
