# Task 13 — Email Notifikácie cez Resend

**Modul:** 5 — Emaily a deploy
**Čas:** ~45–60 min
**Obtiažnosť:** ★★★☆☆

## Čo sa naučíš

- Ako funguje **transakčný email** (SMTP vs API services)
- Ako si nastaviť **Resend** účet a získať API key
- Ako napísať **email templates** v React Email
- Ako poslať email zo server action (a prečo `fire-and-forget` na Vercele NEFUNGUJE)

## Background

### Prečo Resend a nie nodemailer + Gmail?

- **Gmail SMTP** = rate limit 500/deň, často padá do spamu, autentifikácia komplikovaná
- **Resend** = API key, free tier 100 emailov/deň + 3000/mesiac, dobrá deliverability,
  krásne docs

Pre MVP a malé business je Resend ideálny.

### React Email

Email templates písané v JSX. Pod kapotou sa to transpile-uje na (ošklivý ale univerzálny)
HTML s inline štýlmi. Bez React Email by si musel písať `<table>` špageti, ktoré fungujú
aj v Outlook 2007.

### Pozor: `fire-and-forget` na Vercele NEFUNGUJE

Naivný pattern v server action:

```ts
// ❌ Funguje LOKALNE, ale na Vercele email nikdy nepride
void sendEmail(...).catch(console.error);
return ok(...);
```

Vercel **serverless functions** fungujú tak, že akonáhle funkcia returneuje, runtime
funkciu **zmrazí** — všetky pending promises sa zahodia. Lokálne (`pnpm dev`) to funguje,
lebo Node.js proces beží ďalej; na Vercele nie.

**Dve riešenia:**

1. **Await pred returnom** (najjednoduchšie pre MVP):
   ```ts
   await sendEmail(...);
   return ok(...);
   ```
   Cena: response sa predĺži o ~1s. Pre booking je to OK — user už klikol, čaká na potvrdenie.

2. **`waitUntil` z Vercel funkcií** (production-grade):
   ```ts
   import { waitUntil } from '@vercel/functions';
   waitUntil(sendEmail(...));
   return ok(...);
   ```
   Toto **Vercelu povie, aby nezmrazil funkciu** kým promise nedobehne. Response je rýchla.

V tomto tasku použijeme **`await`** (jednoduchšie, menej magic). V production grade apke
prejdeme na `waitUntil`.

## Tvoja úloha

### 1. Zaregistruj sa na Resend

1. Choď na [resend.com](https://resend.com) → Sign Up (cez GitHub)
2. **Skip** verifikáciu domény zatiaľ — pre dev použijeme defaultnú doménu
   `onboarding@resend.dev`
3. **API Keys → Create API Key** → pomenuj "booq-me dev" → skopíruj kľúč

### 2. Pridaj do `.env.local`

```bash
RESEND_API_KEY="re_..."
RESEND_FROM_EMAIL="onboarding@resend.dev"
ADMIN_EMAIL="tvoj-realny-email@example.com"
```

A do `.env.local.example`:

```bash
RESEND_API_KEY=""
RESEND_FROM_EMAIL="onboarding@resend.dev"
ADMIN_EMAIL=""
```

> ⚠️ V production použiješ vlastnú doménu — `onboarding@resend.dev` ide často do spamu.
> Po deploye nastavíš `RESEND_FROM_EMAIL="rezervacia@samuel-barber.sk"` a v Resend overíš
> doménu cez DNS records.

> 💡 Pre testovanie dovedna použij **svoj realny email** pre `ADMIN_EMAIL` — uvidíš tam
> dorazené notifikácie.

### 3. Nainštaluj balíky

```bash
pnpm add resend @react-email/components @react-email/render
pnpm add -D react-email
```

### 4. Vytvor email templates v priečinku `emails/`

V rooti projektu (nie v `src/`) vytvor `emails/booking-confirmation.tsx`:

```tsx
import {
  Body, Container, Head, Heading, Hr, Html, Preview, Section, Text,
} from '@react-email/components';

type Props = {
  customerName: string;
  serviceName: string;
  dateTime: string; // uz naformatovany "22.05.2026 13:15"
  priceFormatted: string; // uz naformatovana "18,00 €"
  durationFormatted: string; // "45 min"
  salonName: string;
  salonAddress: string;
};

export const BookingConfirmationEmail = ({
  customerName, serviceName, dateTime, priceFormatted, durationFormatted,
  salonName, salonAddress,
}: Props) => (
  <Html>
    <Head />
    <Preview>Potvrdenie rezervácie u {salonName}</Preview>
    <Body style={{ fontFamily: 'sans-serif', backgroundColor: '#f9fafb', padding: '24px' }}>
      <Container style={{ maxWidth: 560, margin: '0 auto', backgroundColor: 'white', padding: '32px', borderRadius: 8 }}>
        <Heading style={{ fontSize: 24 }}>Tvoja rezervácia je potvrdená ✅</Heading>
        <Text>Ahoj {customerName},</Text>
        <Text>ďakujeme za rezerváciu. Tu sú detaily:</Text>

        <Section style={{ backgroundColor: '#f3f4f6', padding: 16, borderRadius: 6, marginTop: 16 }}>
          <Text style={{ margin: '4px 0' }}><strong>Služba:</strong> {serviceName}</Text>
          <Text style={{ margin: '4px 0' }}><strong>Termín:</strong> {dateTime}</Text>
          <Text style={{ margin: '4px 0' }}><strong>Dĺžka:</strong> {durationFormatted}</Text>
          <Text style={{ margin: '4px 0' }}><strong>Cena:</strong> {priceFormatted}</Text>
        </Section>

        <Hr style={{ marginTop: 24 }} />

        <Text>
          <strong>{salonName}</strong>
          <br />
          {salonAddress}
        </Text>

        <Text style={{ fontSize: 12, color: '#6b7280', marginTop: 24 }}>
          Ak chceš termín zmeniť alebo zrušiť, napíš nám.
        </Text>
      </Container>
    </Body>
  </Html>
);

export default BookingConfirmationEmail;
```

A `emails/new-booking-admin.tsx`:

```tsx
import {
  Body, Container, Head, Heading, Html, Preview, Section, Text,
} from '@react-email/components';

type Props = {
  customerName: string;
  customerEmail: string;
  customerPhone: string;
  serviceName: string;
  dateTime: string;
  note?: string | null;
};

export const NewBookingAdminEmail = ({
  customerName, customerEmail, customerPhone, serviceName, dateTime, note,
}: Props) => (
  <Html>
    <Head />
    <Preview>Nová rezervácia: {customerName}, {dateTime}</Preview>
    <Body style={{ fontFamily: 'sans-serif', padding: '24px' }}>
      <Container style={{ maxWidth: 560, margin: '0 auto' }}>
        <Heading style={{ fontSize: 20 }}>Nová rezervácia</Heading>
        <Section>
          <Text><strong>Zákazník:</strong> {customerName}</Text>
          <Text><strong>Email:</strong> {customerEmail}</Text>
          <Text><strong>Telefón:</strong> {customerPhone}</Text>
          <Text><strong>Služba:</strong> {serviceName}</Text>
          <Text><strong>Termín:</strong> {dateTime}</Text>
          {note && <Text><strong>Poznámka:</strong> {note}</Text>}
        </Section>
      </Container>
    </Body>
  </Html>
);

export default NewBookingAdminEmail;
```

### 5. Vytvor `src/lib/email.ts`

```ts
import { render } from '@react-email/render';
import { Resend } from 'resend';
import { BookingConfirmationEmail } from '../../emails/booking-confirmation';
import { NewBookingAdminEmail } from '../../emails/new-booking-admin';
import { formatBratislavaDateTime } from './timezone';
import { formatPriceFromCents, formatDuration } from './format';

const resend = new Resend(process.env.RESEND_API_KEY);

const SALON = {
  name: 'Samuel Agošton — Barber',
  address: 'Hlavná 12, 811 01 Bratislava',
};

type SendCustomerProps = {
  to: string;
  customerName: string;
  serviceName: string;
  startTime: Date;
  priceCents: number;
  durationMinutes: number;
};

export const sendBookingConfirmationToCustomer = async (props: SendCustomerProps) => {
  const dateTime = formatBratislavaDateTime(props.startTime);

  const html = await render(
    BookingConfirmationEmail({
      customerName: props.customerName,
      serviceName: props.serviceName,
      dateTime,
      priceFormatted: formatPriceFromCents(props.priceCents),
      durationFormatted: formatDuration(props.durationMinutes),
      salonName: SALON.name,
      salonAddress: SALON.address,
    }),
  );

  return resend.emails.send({
    from: process.env.RESEND_FROM_EMAIL!,
    to: props.to,
    subject: `Potvrdenie rezervácie · ${dateTime}`,
    html,
  });
};

type SendAdminProps = {
  customerName: string;
  customerEmail: string;
  customerPhone: string;
  serviceName: string;
  startTime: Date;
  note?: string | null;
};

export const sendNewBookingToAdmin = async (props: SendAdminProps) => {
  const dateTime = formatBratislavaDateTime(props.startTime);

  const html = await render(NewBookingAdminEmail({ ...props, dateTime }));

  return resend.emails.send({
    from: process.env.RESEND_FROM_EMAIL!,
    to: process.env.ADMIN_EMAIL!,
    subject: `Nová rezervácia: ${props.customerName} · ${dateTime}`,
    html,
  });
};
```

> 💡 **`await render(...)`** — od `@react-email/render` v1.x sa render stal async.
> Vracia Promise<string>.

### 6. Zapoj emaily do `createBooking` action

V `src/app/rezervacia/actions.ts`, **pred** `return ok(...)`:

```ts
// Posli emaily SYNCHRONNE (Vercel zmrazi funkciu po returne, takze fire-and-forget by zlyhal).
// Trvanie ~1s je akceptovatelne — user uz klikol, caka na potvrdenie tak ci tak.
const emailPromises = [
  sendBookingConfirmationToCustomer({
    to: parsed.data.customerEmail,
    customerName: parsed.data.customerName,
    serviceName: service.name,
    startTime,
    priceCents: service.priceCents,
    durationMinutes: service.durationMinutes,
  }).catch((err) => console.error('Customer email failed:', err)),

  sendNewBookingToAdmin({
    customerName: parsed.data.customerName,
    customerEmail: parsed.data.customerEmail,
    customerPhone: parsed.data.customerPhone,
    serviceName: service.name,
    startTime,
    note: parsed.data.note,
  }).catch((err) => console.error('Admin email failed:', err)),
];

await Promise.allSettled(emailPromises);
```

Nezabudni importy hore:

```ts
import { sendBookingConfirmationToCustomer, sendNewBookingToAdmin } from '@/lib/email';
```

> 💡 **`Promise.allSettled`** počká na všetky promises, **bez ohľadu** na to či zlyhajú.
> Ak email zlyhá, rezervácia sa **stále** vytvorí (insert prebehol skôr) — user dostane
> success, ale my si v logoch zaznamenáme problém. Lepšie než celá rezervácia zlyhá pre
> zlý SMTP.

### 7. Otestuj v dev móde

```bash
pnpm dev
```

- Sprav novú rezerváciu s **tvojím emailom**
- Po submite počkaj ~2-3 sekundy
- Skontroluj inbox — mal by tam byť mail z `onboarding@resend.dev`
- Skontroluj aj `ADMIN_EMAIL` (admin notifikácia)

> 💡 **Resend tip**: V Resend dashboarde máš `Emails` tab kde vidíš každý odoslaný email,
> jeho status, a celý HTML. Super pre debugging.

### 8. (Bonus) Preview emailov bez odoslania

React Email má dev server na náhľad templates:

```bash
pnpm dlx react-email dev --dir emails
```

Otvor `http://localhost:3000` (alebo iný port ak je 3000 obsadený) — uvidíš všetky tvoje
emaily a môžeš si pohrať s props.

### 9. Commit

```bash
git add .
git commit -m "task 13: resend email notifications for bookings (synchronous send, Vercel-safe)"
git push
```

## Acceptance Criteria

- [ ] Resend účet vytvorený, API key v `.env.local`
- [ ] `emails/` priečinok s 2 templates
- [ ] `src/lib/email.ts` exportuje 2 funkcie
- [ ] `createBooking` action **awaituje** odoslanie emailov pred returnom
- [ ] Po vytvorení rezervácie:
  - [ ] Zákazník dostane potvrdzovací email
  - [ ] Admin dostane notifikáciu o novej rezervácii
- [ ] Email response čas pre `createBooking` je 1-3s (vidíš v Network tab DevTools)
- [ ] V Resend dashboarde vidíš odoslané emaily so statusom "delivered"
- [ ] Cena v emaili je `"18,00 €"`, termín v Bratislavskej zone (`"22.05.2026 13:15"`)
- [ ] Ak email zlyhá (skús: pokaz `RESEND_API_KEY`), rezervácia sa stále vytvorí (nie crash)

## Tipy a riešenia problémov

**Problém:** Email neprišiel
**Riešenie:**
1. Skontroluj Resend dashboard → Emails — je tam záznam?
2. Ak je tam: skontroluj **spam folder** (`onboarding@resend.dev` často padá do spamu)
3. Ak nie je: v termináli kde beží `pnpm dev` hľadaj `Customer email failed:` alebo
   `Admin email failed:`

**Problém:** `Error: Missing required parameter: from`
**Riešenie:** `RESEND_FROM_EMAIL` nie je v `.env.local` alebo má preklep. Vyžaduje formát
`name@domain.com`.

**Problém:** `Resend hodí 403 — Domain not verified`
**Riešenie:** Pre dev musíš použiť `onboarding@resend.dev` ako `from`. Pre vlastnú doménu
musíš overiť DNS (urobíme v Tasku 14 pri deployi).

**Problém:** Emaily v Gmaile vyzerajú inak ako v Outlooku
**Riešenie:** To je *normálne* — email clients renderujú HTML inak. Drž sa
`@react-email/components` (`<Section>`, `<Container>`, ...) a nepoužívaj moderný CSS
(grid, flexbox bez fallbacku).

## Pýtanie sa Claude Code

- *"Vysvetli mi rozdiel medzi transakčným a marketing emailom. Pre rezervácie posielame
  transakčné — čo to znamená pre GDPR?"*
- *"Vercel `waitUntil` — ako presne to funguje? Kedy ho použiť namiesto `await`?"*
- *"Ak Resend padne (downtime), moja rezervácia sa vytvorila ale email neprišiel. Ako
  spravím retry mechanism?"*

## Ďalší krok

➡️ [Task 14 — Deploy na Vercel](14-deploy-to-vercel.md)
