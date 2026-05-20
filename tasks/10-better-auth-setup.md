# Task 10 — Better Auth Setup

**Modul:** 4 — Admin
**Čas:** ~60 min
**Obtiažnosť:** ★★★★☆

## Čo sa naučíš

- Ako funguje **session-based auth** (na rozdiel od JWT)
- Ako sa do schémy pridajú **Better Auth** tabuľky
- Ako chrániť routy cez **middleware**
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
Samuelov účet vytvoríme cez seed alebo CLI command.

## Tvoja úloha

### 1. Nainštaluj Better Auth

```bash
pnpm add better-auth
```

### 2. Pridaj Better Auth tabuľky do schémy

V `src/db/schema.ts` pridaj na koniec:

```ts
// ───── Better Auth tabuľky ─────
export const user = pgTable('user', {
  id: text('id').primaryKey(),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
  emailVerified: boolean('email_verified').notNull().default(false),
  image: text('image'),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
});

export const session = pgTable('session', {
  id: text('id').primaryKey(),
  expiresAt: timestamp('expires_at').notNull(),
  token: text('token').notNull().unique(),
  createdAt: timestamp('created_at').notNull(),
  updatedAt: timestamp('updated_at').notNull(),
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
  createdAt: timestamp('created_at').notNull(),
  updatedAt: timestamp('updated_at').notNull(),
});

export const verification = pgTable('verification', {
  id: text('id').primaryKey(),
  identifier: text('identifier').notNull(),
  value: text('value').notNull(),
  expiresAt: timestamp('expires_at').notNull(),
  createdAt: timestamp('created_at'),
  updatedAt: timestamp('updated_at'),
});

export type User = typeof user.$inferSelect;
export type Session = typeof session.$inferSelect;
```

### 3. Pushni schemu

```bash
pnpm db:push
```

### 4. Pridaj `AUTH_SECRET` do `.env.local`

Vygeneruj náhodný secret:

```bash
openssl rand -base64 32
```

Skopíruj výstup a pridaj do `.env.local`:

```bash
BETTER_AUTH_SECRET="vygenerovany-string-tu"
BETTER_AUTH_URL="http://localhost:3000"
```

Aj do `.env.local.example` (bez hodnoty):

```bash
BETTER_AUTH_SECRET=""
BETTER_AUTH_URL="http://localhost:3000"
```

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

> 💡 `NEXT_PUBLIC_*` env vars sú dostupné aj v browseri. Bez `NEXT_PUBLIC_` len na serveri.

Pridaj do `.env.local`:

```bash
NEXT_PUBLIC_APP_URL="http://localhost:3000"
```

### 8. Vytvor admin usera cez seed

V `src/db/seed.ts` na koniec funkcie `seed`:

```ts
console.log('👤 Vytváram admin usera...');
const { auth } = await import('@/lib/auth');
const adminEmail = 'samuel@booq-me.local';
const adminPassword = 'samuel123'; // ZMEN po prvom prihlásení!

try {
  await auth.api.signUpEmail({
    body: { email: adminEmail, password: adminPassword, name: 'Samuel Agošton' },
  });
  console.log(`✅ Admin vytvorený: ${adminEmail} / ${adminPassword}`);
} catch (err) {
  console.log(`ℹ️ Admin už existuje (alebo error): ${(err as Error).message}`);
}
```

Spusti:

```bash
pnpm db:seed
```

### 9. Vytvor login stránku `src/app/admin/login/page.tsx`

```tsx
'use client';

import { useState, useTransition } from 'react';
import { useRouter } from 'next/navigation';
import { signIn } from '@/lib/auth-client';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { toast } from 'sonner';

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

### 10. Vytvor middleware na ochranu `/admin/*` routes

`src/middleware.ts`:

```ts
import { NextResponse, type NextRequest } from 'next/server';

export const middleware = async (request: NextRequest) => {
  // /admin/login je verejný
  if (request.nextUrl.pathname === '/admin/login') return NextResponse.next();

  // Better Auth ukladá session cookie ako `better-auth.session_token`
  const sessionCookie = request.cookies.get('better-auth.session_token');

  if (!sessionCookie) {
    return NextResponse.redirect(new URL('/admin/login', request.url));
  }

  return NextResponse.next();
};

export const config = {
  matcher: ['/admin/:path*'],
};
```

> ⚠️ **Pozor**: middleware kontroluje **len existenciu cookie**, nie jej platnosť. Pre 99%
> prípadov to stačí (sessions sa rušia z DB pri logoute). Pre paranoia level: validuj
> cookie na serveri vo `layout.tsx` cez `auth.api.getSession()`.

### 11. Vytvor base admin layout `src/app/admin/layout.tsx`

```tsx
import { redirect } from 'next/navigation';
import { headers } from 'next/headers';
import { auth } from '@/lib/auth';
import Link from 'next/link';
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
- Klik "Odhlásiť" → redirect na login
- Skús `/admin` znova bez session — redirect

### 15. Commit

```bash
git add .
git commit -m "task 10: better-auth setup with admin login and protected /admin routes"
git push
```

## Acceptance Criteria

- [ ] Better Auth tabuľky (`user`, `session`, `account`, `verification`) v DB
- [ ] Admin user vytvorený cez seed (`samuel@booq-me.local`)
- [ ] `/admin/login` zobrazí login formulár
- [ ] Úspešné prihlásenie redirectuje na `/admin`
- [ ] `/admin/*` routy sú chránené middleware-om
- [ ] V admin headere vidíš email prihláseného usera a tlačidlo "Odhlásiť"
- [ ] Odhlásenie zruší session a redirectne na login
- [ ] Nesprávne heslo → toast error

## Tipy a riešenia problémov

**Problém:** `Error: getSession is not defined` v middleware
**Riešenie:** Middleware nemôže importovať Better Auth — beží v edge runtime. Použí len
`request.cookies.get(...)` ako vyššie. Plnú session validáciu rob v server components.

**Problém:** Po login redirecte zostane na `/admin/login`
**Riešenie:** Cookie sa ešte nestihla propagovať. Skús `router.refresh()` pred `router.push()`,
alebo použí `window.location.href = '/admin'`.

**Problém:** Seed hodí error `auth is not defined`
**Riešenie:** Better Auth sa nedá importovať na top-level seedu, lebo musí byť dynamic
import. Použí `const { auth } = await import('@/lib/auth')` ako mám vyššie.

**Problém:** "Cannot find module '@/lib/auth'" v API route
**Riešenie:** Skontroluj `tsconfig.json` že `paths` má `"@/*": ["./src/*"]`. Defaultný
create-next-app to má.

## Pýtanie sa Claude Code

- *"Vysvetli mi rozdiel medzi session-based auth a JWT. Better Auth používa sessions — prečo
  je to default?"*
- *"Ako bezpečne uložím heslo do DB? Better Auth to robí ako?"*
- *"Môj middleware povolí prístup len ak existuje cookie. Ale čo ak niekto skopíruje cookie?
  Ako sa to dá zabrániť?"*

## Ďalší krok

➡️ [Task 11 — Admin: zoznam rezervácií](11-admin-bookings-list.md)
