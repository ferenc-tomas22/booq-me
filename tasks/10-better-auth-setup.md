# Task 10 — Better Auth Setup

**Modul:** 4 — Admin
**Čas:** ~60–90 min
**Obtiažnosť:** ★★★★☆

## Čo sa naučíš

- Ako funguje **session-based auth** (na rozdiel od JWT)
- Ako sa do schémy pridajú **Better Auth** tabuľky
- Ako chrániť routy cez **middleware** + helper `getSessionCookie`
- Pattern **guard funkcií** na serveri

## Background

### Better Auth — prečo nie NextAuth?

[Better Auth](https://www.better-auth.com) je novší framework-agnostic auth lib. Výhody:
- TypeScript-first
- Vlastný plugin systém
- Funguje aj mimo Next.js
- Lepšia DX ako NextAuth podľa väčšiny komunity

Použijeme **email + password** strategy. Žiadny Google/GitHub OAuth — pre admin (Samuel)
úplne stačí jeden účet.

### Session vs JWT

- **JWT** = token v cookie obsahuje user data, server len overí signature. Stateless.
- **Session** = token v cookie odkazuje na DB záznam `sessions`. Server pri každom requeste
  pozrie do DB. Stateful, ale vieš okamžite invalidate session.

Better Auth defaultne používa **sessions** uložené v DB. To je presne čo chceme — keď
zmeníš heslo, všetky session sa zrušia.

### Single admin model

Pre MVP máme **jediného admin usera = Samuel**. Žiadny signup endpoint pre verejnosť.
Samuelov účet vytvoríme cez seed.

## Tvoja úloha

### 1. Nainštaluj Better Auth

```bash
pnpm add better-auth
```

### 2. Pridaj Better Auth tabuľky do schémy

V `src/db/schema.ts` pridaj na koniec (pred `Type exports`):

```ts
// ───── Better Auth tabuľky ─────
export const user = pgTable('user', {
  id: text('id').primaryKey(),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
  emailVerified: boolean('email_verified').notNull().default(false),
  image: text('image'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const session = pgTable('session', {
  id: text('id').primaryKey(),
  expiresAt: timestamp('expires_at', { withTimezone: true }).notNull(),
  token: text('token').notNull().unique(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull(),
  ipAddress: text('ip_address'),
  userAgent: text('user_agent'),
  userId: text('user_id').notNull().references(() => user.id, { onDelete: 'cascade' }),
});

export const account = pgTable('account', {
  id: text('id').primaryKey(),
  accountId: text('account_id').notNull(),
  providerId: text('provider_id').notNull(),
  userId: text('user_id').notNull().references(() => user.id, { onDelete: 'cascade' }),
  password: text('password'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull(),
});

export const verification = pgTable('verification', {
  id: text('id').primaryKey(),
  identifier: text('identifier').notNull(),
  value: text('value').notNull(),
  expiresAt: timestamp('expires_at', { withTimezone: true }).notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }),
  updatedAt: timestamp('updated_at', { withTimezone: true }),
});

export type User = typeof user.$inferSelect;
export type Session = typeof session.$inferSelect;
```

### 3. Pushni schemu

```bash
pnpm db:push
```

### 4. Pridaj env vars

Vygeneruj náhodný secret:

```bash
openssl rand -base64 32
```

Skopíruj výstup a pridaj do `.env.local`:

```bash
BETTER_AUTH_SECRET="vygenerovany-string-tu"
BETTER_AUTH_URL="http://localhost:3000"
NEXT_PUBLIC_APP_URL="http://localhost:3000"
```

Aj do `.env.local.example`:

```bash
BETTER_AUTH_SECRET=""
BETTER_AUTH_URL="http://localhost:3000"
NEXT_PUBLIC_APP_URL="http://localhost:3000"
```

> 💡 `NEXT_PUBLIC_*` env vars sú dostupné aj v browseri. Bez `NEXT_PUBLIC_` len na serveri.

### 5. Vytvor Better Auth config `src/lib/auth.ts`

```ts
import { betterAuth } from 'better-auth';
import { drizzleAdapter } from 'better-auth/adapters/drizzle';
import { db } from '@/db';
import * as schema from '@/db/schema';

export const auth = betterAuth({
  database: drizzleAdapter(db, {
    provider: 'pg',
    schema: {
      user: schema.user,
      session: schema.session,
      account: schema.account,
      verification: schema.verification,
    },
  }),
  emailAndPassword: {
    enabled: true,
    autoSignIn: true,
  },
});
```

### 6. Vytvor API route `src/app/api/auth/[...all]/route.ts`

```ts
import { auth } from '@/lib/auth';
import { toNextJsHandler } from 'better-auth/next-js';

export const { GET, POST } = toNextJsHandler(auth);
```

### 7. Vytvor client `src/lib/auth-client.ts`

```ts
import { createAuthClient } from 'better-auth/react';

export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_APP_URL ?? 'http://localhost:3000',
});

export const { signIn, signOut, useSession } = authClient;
```

### 8. Vytvor admin usera cez seed (idempotentne)

V `src/db/seed.ts` na koniec funkcie `seed` (pred `console.log('✅ Seed dokončený.')`):

```ts
console.log('👤 Vytváram admin usera (ak este neexistuje)...');
const { auth } = await import('../lib/auth');
const { eq } = await import('drizzle-orm');
const { user } = await import('./schema');

const adminEmail = 'samuel@booq-me.local';
const adminPassword = 'samuel123'; // ZMEN PO PRVOM PRIHLASENI (Task 14)!

const [existing] = await db.select().from(user).where(eq(user.email, adminEmail));

if (existing) {
  console.log(`ℹ️ Admin uz existuje: ${adminEmail}`);
} else {
  await auth.api.signUpEmail({
    body: { email: adminEmail, password: adminPassword, name: 'Samuel Agošton' },
  });
  console.log(`✅ Admin vytvoreny: ${adminEmail} / ${adminPassword}`);
  console.log('⚠️  Po prvom prihlaseni zmen heslo cez `pnpm admin:reset-password` (Task 14)');
}
```

Spusti:

```bash
pnpm db:seed
```

Mal by si vidieť `✅ Admin vytvoreny`. Druhé spustenie ukáže `ℹ️ Admin uz existuje`.

### 9. Vytvor login stránku `src/app/admin/login/page.tsx`

```tsx
'use client';

import { useState, useTransition } from 'react';
import { useRouter } from 'next/navigation';
import { toast } from 'sonner';
import { signIn } from '@/lib/auth-client';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';

export default function AdminLoginPage() {
  const router = useRouter();
  const [isPending, startTransition] = useTransition();
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    startTransition(async () => {
      const { error } = await signIn.email({ email, password });
      if (error) {
        toast.error(error.message ?? 'Prihlásenie zlyhalo');
        return;
      }
      router.push('/admin');
      router.refresh();
    });
  };

  return (
    <div className="max-w-sm mx-auto py-16 space-y-6">
      <h1 className="text-3xl font-bold text-center">Admin Login</h1>
      <form onSubmit={handleSubmit} className="space-y-4">
        <div>
          <Label htmlFor="email">Email</Label>
          <Input id="email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} required />
        </div>
        <div>
          <Label htmlFor="password">Heslo</Label>
          <Input id="password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} required />
        </div>
        <Button type="submit" disabled={isPending} className="w-full">
          {isPending ? 'Prihlasujem...' : 'Prihlásiť sa'}
        </Button>
      </form>
    </div>
  );
}
```

### 10. Vytvor middleware na ochranu `/admin/*`

Better Auth ma helper `getSessionCookie` ktorý vyhladá session cookie s akymkolvek
prefixom (vratane `__Secure-` v produkcii). To je bezpecnejsie nez hardcoded nazov.

`src/middleware.ts`:

```ts
import { NextResponse, type NextRequest } from 'next/server';
import { getSessionCookie } from 'better-auth/cookies';

export const middleware = (request: NextRequest) => {
  // /admin/login je verejny
  if (request.nextUrl.pathname === '/admin/login') return NextResponse.next();

  const sessionCookie = getSessionCookie(request);

  if (!sessionCookie) {
    return NextResponse.redirect(new URL('/admin/login', request.url));
  }

  return NextResponse.next();
};

export const config = {
  matcher: ['/admin/:path*'],
};
```

> ⚠️ **Middleware kontroluje len existenciu cookie**, nie jej platnost. Pre 99% pripadov to
> staci (zruseny session sa zmaze z DB pri logoute, takze pri dalsiom requeste sa cookie
> nepouzije). Pre paranoia level: validuj cookie na serveri vo `layout.tsx` cez
> `auth.api.getSession()` — to robime v dalsom kroku.

### 11. Vytvor admin layout `src/app/admin/layout.tsx`

```tsx
import { redirect } from 'next/navigation';
import { headers } from 'next/headers';
import Link from 'next/link';
import { auth } from '@/lib/auth';
import { LogoutButton } from './logout-button';

export default async function AdminLayout({ children }: { children: React.ReactNode }) {
  const session = await auth.api.getSession({ headers: await headers() });

  if (!session) redirect('/admin/login');

  return (
    <div>
      <header className="border-b mb-8">
        <div className="container mx-auto flex h-14 items-center justify-between px-4">
          <nav className="flex gap-6 text-sm">
            <Link href="/admin" className="font-medium">Rezervácie</Link>
            <Link href="/admin/sluzby" className="font-medium">Služby</Link>
            <Link href="/admin/blokovanie" className="font-medium">Blokovanie</Link>
          </nav>
          <div className="flex items-center gap-4">
            <span className="text-sm text-muted-foreground">{session.user.email}</span>
            <LogoutButton />
          </div>
        </div>
      </header>
      <main>{children}</main>
    </div>
  );
}
```

### 12. Vytvor logout button `src/app/admin/logout-button.tsx`

```tsx
'use client';

import { useRouter } from 'next/navigation';
import { signOut } from '@/lib/auth-client';
import { Button } from '@/components/ui/button';

export const LogoutButton = () => {
  const router = useRouter();

  return (
    <Button
      variant="ghost"
      size="sm"
      onClick={async () => {
        await signOut();
        router.push('/admin/login');
      }}
    >
      Odhlásiť
    </Button>
  );
};
```

### 13. Vytvor stub admin home `src/app/admin/page.tsx`

```tsx
export default function AdminHomePage() {
  return (
    <div className="container mx-auto px-4">
      <h1 className="text-3xl font-bold">Admin — Rezervácie</h1>
      <p className="text-muted-foreground mt-2">Bude tu zoznam rezervácií (Task 11).</p>
    </div>
  );
}
```

### 14. Otestuj

```bash
pnpm dev
```

- Otvor `/admin` → redirect na `/admin/login`
- Vyplň `samuel@booq-me.local` / `samuel123` → klik "Prihlásiť sa"
- Mal by si byť na `/admin` s nav header-om
- V DevTools → Application → Cookies → vidíš cookie s prefixom `better-auth.session_token`
- Klik "Odhlásiť" → redirect na login
- Skús `/admin` znova bez session — redirect

### 15. Commit

```bash
git add .
git commit -m "task 10: better-auth setup with admin login, getSessionCookie middleware, idempotent seed"
git push
```

## Acceptance Criteria

- [ ] Better Auth tabuľky (`user`, `session`, `account`, `verification`) v DB
- [ ] Admin user vytvorený cez seed (`samuel@booq-me.local`)
- [ ] Druhé spustenie `pnpm db:seed` ukáže `ℹ️ Admin uz existuje` (nie crash)
- [ ] `/admin/login` zobrazí login formulár
- [ ] Úspešné prihlásenie redirectuje na `/admin`
- [ ] `/admin/*` routy sú chránené middleware-om cez `getSessionCookie`
- [ ] V admin headere vidíš email prihláseného usera a tlačidlo "Odhlásiť"
- [ ] Odhlásenie zruší session a redirectne na login
- [ ] Nesprávne heslo → toast error

## Tipy a riešenia problémov

**Problém:** `Error: Cannot find module 'better-auth/cookies'`
**Riešenie:** `pnpm add better-auth` musí byť vykonaný. Skontroluj verziu: `pnpm list
better-auth` — `getSessionCookie` je v `v1.x+`.

**Problém:** Po login redirecte zostane na `/admin/login`
**Riešenie:** Cookie sa ešte nestihla propagovať. Pridaj `router.refresh()` PRED
`router.push()` (mám tam to).

**Problém:** `Error: auth is not defined` v seede
**Riešenie:** Better Auth sa neda importovať na top-level seedu (Next.js related side
effects). Pouzi `const { auth } = await import('../lib/auth')` (mam tam to).

**Problém:** Middleware spadne na `Cannot find name 'getSessionCookie'`
**Riešenie:** Import je `from 'better-auth/cookies'` (NIE z `'better-auth'`).

## Pýtanie sa Claude Code

- *"Vysvetli mi rozdiel medzi session-based auth a JWT. Better Auth používa sessions —
  prečo je to default?"*
- *"Ako bezpečne uložím heslo do DB? Better Auth to robí ako?"*
- *"Môj middleware povolí prístup len ak existuje cookie. Ale čo ak niekto skopíruje
  cookie? Ako sa to dá zabrániť?"*

## Ďalší krok

➡️ [Task 11 — Admin: zoznam rezervácií](11-admin-bookings-list.md)
