# Task 12 — Admin: Služby + Blokovanie Termínov

**Modul:** 4 — Admin
**Čas:** ~60–90 min
**Obtiažnosť:** ★★★☆☆

## Čo sa naučíš

- Komplexný **CRUD pattern** (Create / Read / Update / Delete) s formulármi a server actions
- Ako spraviť **datetime picker** pre rozsah dátumov (dovolenka od–do)
- Ako sa pracuje so **soft delete** (status = INACTIVE namiesto reálneho DELETE)

## Background

Samuel potrebuje vedieť:

1. **Spravovať služby** — pridať novú (napr. "Holenie hlavy"), zmeniť cenu, deaktivovať
2. **Blokovať termíny** — keď ide na dovolenku (od 1.7. do 15.7.) alebo má neplánovanú prestávku

### Soft delete vs hard delete

- **Hard delete** = `DELETE FROM services WHERE id = ...`. Záznam zmizne.
- **Soft delete** = `UPDATE services SET status = 'INACTIVE'`. Záznam zostáva.

Pre nás je **soft delete** lepší — historické rezervácie majú referenciu na službu, a
chceme ich vedieť zobraziť aj keď službu už neponukáme. Inak by FK constraint blokol delete.

### `<input type="datetime-local">` a timezone

Browser input `<input type="datetime-local">` ti vráti string ako `"2026-05-22T14:00"` —
**bez timezone info**. `new Date(...)` ho interpretuje ako **lokálny čas browser-a**. Pre
admin appku (Samuel pracuje vždy v Bratislave) to je v poriadku — ale na serveri to musíme
konvertovať explicitne cez `buildBratislavaDateTime` z Tasku 08.

## Tvoja úloha

### 1. Nainstaluj shadcn Dialog (potrebujeme ho v service form-e)

```bash
pnpm dlx shadcn@latest add dialog
```

### Časť A: Správa služieb

#### A1. Server actions `src/app/admin/sluzby/actions.ts`

```ts
'use server';

import { eq } from 'drizzle-orm';
import { revalidatePath } from 'next/cache';
import { headers } from 'next/headers';
import { z } from 'zod';
import { db } from '@/db';
import { services } from '@/db/schema';
import { auth } from '@/lib/auth';
import { ok, fail, type ActionResult } from '@/lib/action-result';

const requireAdmin = async () => {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) throw new Error('Unauthorized');
};

// Cena vstupuje v eurach (napr. "18.50"), v DB ulozime cents (1850).
const serviceSchema = z.object({
  name: z.string().min(2).max(100),
  description: z.string().max(500).optional().transform((v) => v?.trim() || null),
  durationMinutes: z.coerce.number().int().min(5).max(480),
  priceEur: z.coerce.number().min(0).max(9999),
});

const toCents = (eur: number): number => Math.round(eur * 100);

export const createService = async (input: z.infer<typeof serviceSchema>): Promise<ActionResult> => {
  try {
    await requireAdmin();
    const parsed = serviceSchema.parse(input);
    await db.insert(services).values({
      name: parsed.name,
      description: parsed.description,
      durationMinutes: parsed.durationMinutes,
      priceCents: toCents(parsed.priceEur),
    });
    revalidatePath('/admin/sluzby');
    revalidatePath('/rezervacia');
    revalidatePath('/');
    return ok(undefined);
  } catch (err) {
    return fail((err as Error).message);
  }
};

export const updateService = async (
  id: string,
  input: z.infer<typeof serviceSchema>,
): Promise<ActionResult> => {
  try {
    await requireAdmin();
    const parsed = serviceSchema.parse(input);
    await db.update(services).set({
      name: parsed.name,
      description: parsed.description,
      durationMinutes: parsed.durationMinutes,
      priceCents: toCents(parsed.priceEur),
    }).where(eq(services.id, id));
    revalidatePath('/admin/sluzby');
    revalidatePath('/rezervacia');
    revalidatePath('/');
    return ok(undefined);
  } catch (err) {
    return fail((err as Error).message);
  }
};

export const toggleServiceStatus = async (id: string): Promise<ActionResult> => {
  try {
    await requireAdmin();
    const [current] = await db.select().from(services).where(eq(services.id, id));
    if (!current) return fail('Služba neexistuje');
    await db
      .update(services)
      .set({ status: current.status === 'ACTIVE' ? 'INACTIVE' : 'ACTIVE' })
      .where(eq(services.id, id));
    revalidatePath('/admin/sluzby');
    revalidatePath('/rezervacia');
    revalidatePath('/');
    return ok(undefined);
  } catch (err) {
    return fail((err as Error).message);
  }
};
```

#### A2. Stránka `src/app/admin/sluzby/page.tsx`

```tsx
import { db } from '@/db';
import { services } from '@/db/schema';
import { Badge } from '@/components/ui/badge';
import {
  Table, TableBody, TableCell, TableHead, TableHeader, TableRow,
} from '@/components/ui/table';
import { formatPriceFromCents, formatDuration } from '@/lib/format';
import { ServiceFormDialog } from './service-form-dialog';
import { ToggleStatusButton } from './toggle-status-button';

export default async function AdminServicesPage() {
  const allServices = await db.select().from(services).orderBy(services.name);

  return (
    <div className="container mx-auto px-4 space-y-6">
      <div className="flex justify-between items-center">
        <h1 className="text-3xl font-bold">Služby</h1>
        <ServiceFormDialog mode="create" />
      </div>

      <Table>
        <TableHeader>
          <TableRow>
            <TableHead>Názov</TableHead>
            <TableHead>Dĺžka</TableHead>
            <TableHead>Cena</TableHead>
            <TableHead>Stav</TableHead>
            <TableHead></TableHead>
          </TableRow>
        </TableHeader>
        <TableBody>
          {allServices.map((service) => (
            <TableRow key={service.id}>
              <TableCell className="font-medium">{service.name}</TableCell>
              <TableCell>{formatDuration(service.durationMinutes)}</TableCell>
              <TableCell>{formatPriceFromCents(service.priceCents)}</TableCell>
              <TableCell>
                <Badge variant={service.status === 'ACTIVE' ? 'default' : 'secondary'}>
                  {service.status === 'ACTIVE' ? 'Aktívna' : 'Neaktívna'}
                </Badge>
              </TableCell>
              <TableCell className="space-x-2">
                <ServiceFormDialog mode="edit" service={service} />
                <ToggleStatusButton serviceId={service.id} currentStatus={service.status} />
              </TableCell>
            </TableRow>
          ))}
        </TableBody>
      </Table>
    </div>
  );
}
```

#### A3. Dialog formulár `src/app/admin/sluzby/service-form-dialog.tsx`

```tsx
'use client';

import { useState, useTransition } from 'react';
import { useRouter } from 'next/navigation';
import { toast } from 'sonner';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Textarea } from '@/components/ui/textarea';
import {
  Dialog, DialogContent, DialogFooter, DialogHeader, DialogTitle, DialogTrigger,
} from '@/components/ui/dialog';
import { createService, updateService } from './actions';
import type { Service } from '@/db/schema';

type Props = { mode: 'create' } | { mode: 'edit'; service: Service };

export const ServiceFormDialog = (props: Props) => {
  const router = useRouter();
  const [open, setOpen] = useState(false);
  const [isPending, startTransition] = useTransition();

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    const input = {
      name: String(formData.get('name')),
      description: String(formData.get('description') ?? ''),
      durationMinutes: Number(formData.get('durationMinutes')),
      priceEur: Number(formData.get('priceEur')),
    };

    startTransition(async () => {
      const result = props.mode === 'create'
        ? await createService(input)
        : await updateService(props.service.id, input);

      if (!result.ok) {
        toast.error(result.error);
        return;
      }
      toast.success(props.mode === 'create' ? 'Služba pridaná' : 'Služba upravená');
      setOpen(false);
      router.refresh();
    });
  };

  // Pre edit mod: konvertuj centy spat na eura pre input (cena zobrazena uzivatelovi v eurach).
  const defaultPriceEur = props.mode === 'edit' ? props.service.priceCents / 100 : 15;
  const defaults = props.mode === 'edit' ? props.service : null;

  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogTrigger asChild>
        <Button variant={props.mode === 'create' ? 'default' : 'outline'} size="sm">
          {props.mode === 'create' ? 'Pridať službu' : 'Upraviť'}
        </Button>
      </DialogTrigger>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>{props.mode === 'create' ? 'Nová služba' : 'Úprava služby'}</DialogTitle>
        </DialogHeader>
        <form onSubmit={handleSubmit} className="space-y-4">
          <div>
            <Label htmlFor="name">Názov</Label>
            <Input id="name" name="name" required defaultValue={defaults?.name} />
          </div>
          <div>
            <Label htmlFor="description">Popis</Label>
            <Textarea id="description" name="description" defaultValue={defaults?.description ?? ''} />
          </div>
          <div className="grid grid-cols-2 gap-4">
            <div>
              <Label htmlFor="durationMinutes">Dĺžka (min)</Label>
              <Input
                id="durationMinutes" name="durationMinutes" type="number" required
                min={5} max={480}
                defaultValue={defaults?.durationMinutes ?? 30}
              />
            </div>
            <div>
              <Label htmlFor="priceEur">Cena (€)</Label>
              <Input
                id="priceEur" name="priceEur" type="number" step="0.5" required
                min={0}
                defaultValue={defaultPriceEur}
              />
            </div>
          </div>
          <DialogFooter>
            <Button type="submit" disabled={isPending}>
              {isPending ? 'Ukladám...' : 'Uložiť'}
            </Button>
          </DialogFooter>
        </form>
      </DialogContent>
    </Dialog>
  );
};
```

#### A4. Toggle status button `src/app/admin/sluzby/toggle-status-button.tsx`

```tsx
'use client';

import { useTransition } from 'react';
import { useRouter } from 'next/navigation';
import { toast } from 'sonner';
import { Button } from '@/components/ui/button';
import { toggleServiceStatus } from './actions';

type Props = { serviceId: string; currentStatus: 'ACTIVE' | 'INACTIVE' };

export const ToggleStatusButton = ({ serviceId, currentStatus }: Props) => {
  const router = useRouter();
  const [isPending, startTransition] = useTransition();

  return (
    <Button
      variant="outline" size="sm" disabled={isPending}
      onClick={() => {
        startTransition(async () => {
          const result = await toggleServiceStatus(serviceId);
          if (!result.ok) return toast.error(result.error);
          toast.success(currentStatus === 'ACTIVE' ? 'Deaktivované' : 'Aktivované');
          router.refresh();
        });
      }}
    >
      {currentStatus === 'ACTIVE' ? 'Deaktivovať' : 'Aktivovať'}
    </Button>
  );
};
```

### Časť B: Blokovanie termínov

#### B1. Server actions `src/app/admin/blokovanie/actions.ts`

```ts
'use server';

import { eq } from 'drizzle-orm';
import { revalidatePath } from 'next/cache';
import { headers } from 'next/headers';
import { z } from 'zod';
import { db } from '@/db';
import { blockedSlots } from '@/db/schema';
import { auth } from '@/lib/auth';
import { ok, fail, type ActionResult } from '@/lib/action-result';
import { buildBratislavaDateTime } from '@/lib/timezone';

const requireAdmin = async () => {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) throw new Error('Unauthorized');
};

const blockSchema = z.object({
  // Format z <input type="datetime-local">: "YYYY-MM-DDTHH:mm"
  startLocal: z.string().regex(/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}$/, 'Neplatny format datumu'),
  endLocal: z.string().regex(/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}$/, 'Neplatny format datumu'),
  reason: z.string().max(200).optional().transform((v) => v?.trim() || null),
});

// Konvertuje "YYYY-MM-DDTHH:mm" (lokalny cas) na UTC Date cez Europe/Bratislava.
const parseLocal = (str: string): Date => {
  const [datePart, timePart] = str.split('T');
  return buildBratislavaDateTime(datePart, timePart);
};

export const createBlock = async (input: z.infer<typeof blockSchema>): Promise<ActionResult> => {
  try {
    await requireAdmin();
    const parsed = blockSchema.parse(input);

    const startTime = parseLocal(parsed.startLocal);
    const endTime = parseLocal(parsed.endLocal);

    if (endTime <= startTime) return fail('Koniec musí byť po začiatku');
    if (startTime < new Date()) return fail('Nemôžeš blokovať čas v minulosti');

    await db.insert(blockedSlots).values({ startTime, endTime, reason: parsed.reason });
    revalidatePath('/admin/blokovanie');
    revalidatePath('/rezervacia');
    return ok(undefined);
  } catch (err) {
    return fail((err as Error).message);
  }
};

export const deleteBlock = async (id: string): Promise<ActionResult> => {
  try {
    await requireAdmin();
    await db.delete(blockedSlots).where(eq(blockedSlots.id, id));
    revalidatePath('/admin/blokovanie');
    revalidatePath('/rezervacia');
    return ok(undefined);
  } catch (err) {
    return fail((err as Error).message);
  }
};
```

#### B2. Stránka `src/app/admin/blokovanie/page.tsx`

```tsx
import { gte } from 'drizzle-orm';
import { db } from '@/db';
import { blockedSlots } from '@/db/schema';
import {
  Table, TableBody, TableCell, TableHead, TableHeader, TableRow,
} from '@/components/ui/table';
import { formatBratislavaDateTime } from '@/lib/timezone';
import { BlockForm } from './block-form';
import { DeleteBlockButton } from './delete-block-button';

export default async function AdminBlocksPage() {
  const all = await db
    .select()
    .from(blockedSlots)
    .where(gte(blockedSlots.endTime, new Date()))
    .orderBy(blockedSlots.startTime);

  return (
    <div className="container mx-auto px-4 space-y-8">
      <h1 className="text-3xl font-bold">Blokovanie termínov</h1>

      <div className="border rounded-lg p-6">
        <h2 className="text-lg font-semibold mb-4">Pridať nové blokovanie</h2>
        <BlockForm />
      </div>

      <div>
        <h2 className="text-lg font-semibold mb-4">Aktívne a budúce blokovania</h2>
        {all.length === 0 ? (
          <p className="text-muted-foreground">Žiadne blokovania.</p>
        ) : (
          <Table>
            <TableHeader>
              <TableRow>
                <TableHead>Od</TableHead>
                <TableHead>Do</TableHead>
                <TableHead>Dôvod</TableHead>
                <TableHead></TableHead>
              </TableRow>
            </TableHeader>
            <TableBody>
              {all.map((b) => (
                <TableRow key={b.id}>
                  <TableCell>{formatBratislavaDateTime(b.startTime)}</TableCell>
                  <TableCell>{formatBratislavaDateTime(b.endTime)}</TableCell>
                  <TableCell>{b.reason ?? '—'}</TableCell>
                  <TableCell><DeleteBlockButton id={b.id} /></TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        )}
      </div>
    </div>
  );
}
```

#### B3. Block formulár `src/app/admin/blokovanie/block-form.tsx`

```tsx
'use client';

import { useTransition } from 'react';
import { useRouter } from 'next/navigation';
import { toast } from 'sonner';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { createBlock } from './actions';

export const BlockForm = () => {
  const router = useRouter();
  const [isPending, startTransition] = useTransition();

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    const startLocal = String(formData.get('start'));
    const endLocal = String(formData.get('end'));
    const reason = String(formData.get('reason') ?? '');

    startTransition(async () => {
      const result = await createBlock({ startLocal, endLocal, reason });
      if (!result.ok) return toast.error(result.error);
      toast.success('Blokovanie pridané');
      (e.target as HTMLFormElement).reset();
      router.refresh();
    });
  };

  return (
    <form onSubmit={handleSubmit} className="grid grid-cols-1 md:grid-cols-4 gap-4">
      <div>
        <Label htmlFor="start">Od</Label>
        <Input id="start" name="start" type="datetime-local" required />
      </div>
      <div>
        <Label htmlFor="end">Do</Label>
        <Input id="end" name="end" type="datetime-local" required />
      </div>
      <div>
        <Label htmlFor="reason">Dôvod (voliteľné)</Label>
        <Input id="reason" name="reason" placeholder="Dovolenka, Sviatok..." />
      </div>
      <div className="flex items-end">
        <Button type="submit" disabled={isPending} className="w-full">
          {isPending ? 'Ukladám...' : 'Pridať'}
        </Button>
      </div>
    </form>
  );
};
```

#### B4. Delete button `src/app/admin/blokovanie/delete-block-button.tsx`

```tsx
'use client';

import { useTransition } from 'react';
import { useRouter } from 'next/navigation';
import { toast } from 'sonner';
import { Button } from '@/components/ui/button';
import { deleteBlock } from './actions';

export const DeleteBlockButton = ({ id }: { id: string }) => {
  const router = useRouter();
  const [isPending, startTransition] = useTransition();

  return (
    <Button
      variant="outline" size="sm" disabled={isPending}
      onClick={() => {
        if (!confirm('Naozaj odstrániť toto blokovanie?')) return;
        startTransition(async () => {
          const result = await deleteBlock(id);
          if (!result.ok) return toast.error(result.error);
          toast.success('Blokovanie odstránené');
          router.refresh();
        });
      }}
    >
      Odstrániť
    </Button>
  );
};
```

### 5. Otestuj

```bash
pnpm dev
```

**Služby:**
- `/admin/sluzby` zobrazí všetky služby (cena ako `"18,00 €"`)
- "Pridať službu" → vyplň formulár (cena v eurach, napr. `18.5`) → mali by si vidieť v
  tabuľke a aj v `/` landingu, cena formátovaná správne
- "Upraviť" → zmeň cenu z `18.5` na `20` → uvidíš `20,00 €` okamžite
- "Deaktivovať" → služba sa stane neaktívna → v `/rezervacia` flow ju neuvidíš

**Blokovanie:**
- `/admin/blokovanie` zobrazí blokovania (čas v Bratislavskej zone)
- Pridaj blok na zajtra 14:00 – 15:00, dôvod "Test"
- Choď na `/rezervacia` → vyber krátku službu → zajtra 14:00 by mal byť **zablokovaný**
- Skús pridať blok do minulosti — dostaneš error toast
- Skús pridať blok s koncom pred začiatkom — dostaneš error toast
- Odstráň blok → slot opäť voľný (po refreshi `/rezervacia`)

### 6. Commit

```bash
git add .
git commit -m "task 12: admin CRUD for services (cents) and time blocking (TZ-safe)"
git push
```

## Acceptance Criteria

- [ ] `/admin/sluzby` zobrazí tabuľku služieb s tlačidlami "Pridať/Upraviť/Deaktivovať"
- [ ] Pridanie služby — zadám cenu `18.5` → v DB sa uloží `priceCents: 1850`
- [ ] Cena v admin tabuľke aj na landingu sa zobrazí ako `"18,50 €"`
- [ ] Deaktivovaná služba sa neobjaví v `/rezervacia` flow
- [ ] `/admin/blokovanie` zobrazí budúce blokovania, časy v Bratislavskej zone
- [ ] Blokovanie sa ukladá v UTC v DB (overiteľné v Drizzle Studio — `+00`)
- [ ] Pridanie blokovania v minulosti → error
- [ ] End ≤ start → error
- [ ] Pridanie blokovania zablokuje sloty v rezervačnom flow
- [ ] Odstránenie blokovania uvoľní sloty (po refreshi)
- [ ] Všetky server actions majú `requireAdmin()` guard

## Bonus

- "Celý deň" toggle pri blokovaní (auto-fill 00:00–23:59)
- "Týždeň dovolenky" preset (vyber dátum od/do, automaticky bloky pre cele dni)
- Pri službe ukáž počet aktívnych budúcich rezervácií (s warningom pri deaktivácii)

## Tipy a riešenia problémov

**Problém:** Zadám cenu `18.5` a v DB sa uloží `1849` namiesto `1850`
**Riešenie:** Floating point — `18.5 * 100 = 1849.9999...`. Použiť `Math.round(eur * 100)`
(mám tam to).

**Problém:** `datetime-local` input ukazuje hodnotu v 24h formáte
**Riešenie:** To je výchozí formát na väčšine OS-ov a v slovenčine je to v poriadku.

**Problém:** Pridám blok na zajtra 14:00, v `/rezervacia` flow je stále voľný
**Riešenie:** Refresh stránky `/rezervacia` (kalendár sa nemení automaticky). Alebo zmeň
dátum a vráť sa naspať.

**Problém:** Edit form ukazuje cenu ako `1850` namiesto `18.5`
**Riešenie:** V `defaultPriceEur` ja delim cents na 100. Skontroluj.

## Pýtanie sa Claude Code

- *"Aký je rozdiel medzi soft delete (status flag) a hard delete (DROP)? Kedy ktoré
  použiť a aké sú implikácie pre FK constraints?"*
- *"Mám 5 server actions pre admin a všetky volajú `requireAdmin()`. Ako to spravím
  elegantnejšie (DRY)? Napríklad cez wrapper funkciu."*
- *"`Math.round(18.5 * 100)` mi dáva `1850`. Prečo `Math.round` a nie `Math.floor`? Čo by
  sa pokazilo?"*

## Ďalší krok

➡️ [Task 13 — Email notifikácie cez Resend](13-resend-emails.md)
