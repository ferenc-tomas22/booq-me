# Task 13 — Email Notifikácie cez Resend

**Modul:** 5 — Emaily a deploy
**Čas:** ~45 min
**Obtiažnosť:** ★★★☆☆

## Čo sa naučíš

- Ako funguje **transakčný email** (SMTP vs API services)
- Ako si nastaviť **Resend** účet a získať API key
- Ako napísať **email templates** v React Email
- Ako poslať email zo server action (best practice: fire-and-forget)

## Background

### Prečo Resend a nie nodemailer + Gmail?

- **Gmail SMTP** = rate limit 500/deň, často padá do spamu, autentifikácia komplikovaná
- **Resend** = API key, free tier 100 emailov/deň + 3000/mesiac, dobrá deliverability, krásne docs

Pre MVP a malé business je Resend ideálny. Aj la-fly ho používa.

### React Email

Email templates písané v JSX. Pod kapotou sa to transpile-uje na (ošklivý ale univerzálny)
HTML s inline štýlmi. Bez React Email by si musel písať `<table>` špageti, ktoré fungujú aj
v Outlook 2007.

### Fire-and-forget pattern

Posielanie emailu môže trvať 1–3 sekundy. Nechceš, aby tvoj `createBooking` na to čakal —
zákazník vidí spinner zbytočne dlho. Riešenie:

```ts
// ❌ ZLE — booking response čaká na email
await sendEmail(...);
return ok(...);

// ✅ DOBRE — booking response príde okamžite, email letí asynchronne
void sendEmail(...).catch(console.error);
return ok(...);
```

`void` povie TypeScriptu "viem že vraciam Promise, nepýtaj sa". `.catch()` zabezpečí, že
exception nezhodí celý request.

## Tvoja úloha

### 1. Zaregistruj sa na Resend

1. Choď na [resend.com](https://resend.com) → Sign Up (cez GitHub)
2. **Skip** verifikáciu domény zatiaľ — pre dev použijeme defaultnú doménu `onboarding@resend.dev`
3. **API Keys → Create API Key** → pomenuj "booq-me dev" → skopíruj kľúč

### 2. Pridaj do `.env.local`

```bash
RESEND_API_KEY="re_..."
RESEND_FROM_EMAIL="onboarding@resend.dev"
ADMIN_EMAIL="samuel@booq-me.local"
```

A do `.env.local.example`:

```bash
RESEND_API_KEY=""
RESEND_FROM_EMAIL="onboarding@resend.dev"
ADMIN_EMAIL=""
```

> ⚠️ V production použiješ vlastnú doménu — `onboarding@resend.dev` ide často do spamu. Po
> deploye nastavíš `RESEND_FROM_EMAIL="rezervacia@samuel-barber.sk"` a v Resend overíš
> doménu cez DNS records.

### 3. Nainštaluj balíky

```bash
pnpm add resend @react-email/components
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
  dateTime: string; // už naformátovaný "22.05.2026 13:15"
  priceEur: number;
  durationMinutes: number;
  salonName: string;
  salonAddress: string;
};

export const BookingConfirmationEmail = ({
  customerName, serviceName, dateTime, priceEur, durationMinutes, salonName, salonAddress,
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
          <Text style={{ margin: '4px 0' }}><strong>Dĺžka:</strong> {durationMinutes} min</Text>
          <Text style={{ margin: '4px 0' }}><strong>Cena:</strong> {priceEur} €</Text>
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
import { Resend } from 'resend';
import { render } from '@react-email/render';
import { BookingConfirmationEmail } from '../../emails/booking-confirmation';
import { NewBookingAdminEmail } from '../../emails/new-booking-admin';

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
  priceEur: number;
  durationMinutes: number;
};

export const sendBookingConfirmationToCustomer = async (props: SendCustomerProps) => {
  const dateTime = props.startTime.toLocaleString('sk-SK', {
    day: '2-digit', month: '2-digit', year: 'numeric',
    hour: '2-digit', minute: '2-digit', timeZone: 'Europe/Bratislava',
  });

  const html = await render(
    BookingConfirmationEmail({
      customerName: props.customerName,
      serviceName: props.serviceName,
      dateTime,
      priceEur: props.priceEur,
      durationMinutes: props.durationMinutes,
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
  const dateTime = props.startTime.toLocaleString('sk-SK', {
    day: '2-digit', month: '2-digit', year: 'numeric',
    hour: '2-digit', minute: '2-digit', timeZone: 'Europe/Bratislava',
  });

  const html = await render(NewBookingAdminEmail({ ...props, dateTime }));

  return resend.emails.send({
    from: process.env.RESEND_FROM_EMAIL!,
    to: process.env.ADMIN_EMAIL!,
    subject: `Nová rezervácia: ${props.customerName} · ${dateTime}`,
    html,
  });
};
```

### 6. Zapoj emaily do `createBooking` action

V `src/app/rezervacia/actions.ts`, pred `return ok(...)`:

```ts
// Fire-and-forget — neblokuje response
void sendBookingConfirmationToCustomer({
  to: parsed.data.customerEmail,
  customerName: parsed.data.customerName,
  serviceName: service.name,
  startTime,
  priceEur: service.priceEur,
  durationMinutes: service.durationMinutes,
}).catch((err) => console.error('Failed to send customer email:', err));

void sendNewBookingToAdmin({
  customerName: parsed.data.customerName,
  customerEmail: parsed.data.customerEmail,
  customerPhone: parsed.data.customerPhone,
  serviceName: service.name,
  startTime,
  note: parsed.data.note,
}).catch((err) => console.error('Failed to send admin email:', err));
```

A nezabudni importy hore:

```ts
import { sendBookingConfirmationToCustomer, sendNewBookingToAdmin } from '@/lib/email';
```

### 7. Otestuj v dev móde

```bash
pnpm dev
```

- Spravi novú rezerváciu s **TVOJÍM emailom** (napr. `samuel@booq-me.local` má tvoj real
  email v `.env.local`)
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
git commit -m "task 13: resend email notifications for booking confirmations"
git push
```

## Acceptance Criteria

- [ ] Resend účet vytvorený, API key v `.env.local`
- [ ] `emails/` priečinok s 2 templates
- [ ] `src/lib/email.ts` exportuje 2 funkcie
- [ ] `createBooking` action volá emaily fire-and-forget
- [ ] Po vytvorení rezervácie:
  - [ ] Zákazník dostane potvrdzovací mail
  - [ ] Admin dostane notifikáciu o novej rezervácii
- [ ] Email response čas pre `createBooking` je stále < 1s (emaily idú async)
- [ ] V Resend dashboarde vidíš odoslané emaily

## Tipy a riešenia problémov

**Problém:** Email neprišiel
**Riešenie:**
1. Skontroluj Resend dashboard → Emails — je tam záznam?
2. Ak je tam: skontroluj **spam folder** (`onboarding@resend.dev` často padá do spamu)
3. Ak nie je: v Vercel/lokálne logy hľadaj `Failed to send`

**Problém:** `Error: Missing required parameter: from`
**Riešenie:** `RESEND_FROM_EMAIL` nie je v `.env.local` alebo má preklep. Vyžaduje formát
`name@domain.com`.

**Problém:** `Resend hodí 403 — Domain not verified`
**Riešenie:** Pre dev musíš použiť `onboarding@resend.dev` ako `from`. Pre vlastnú doménu
musíš overiť DNS (urobíme v Tasku 14 pri deployi).

**Problém:** Emailové templates vyzerajú v Gmaile inak ako v Outlooku
**Riešenie:** To je *normálne* — email clients renderujú HTML inak. Drž sa `@react-email/components`
(`<Section>`, `<Container>`, ...) a nepoužívaj moderný CSS (grid, flexbox bez fallbacku).

## Pýtanie sa Claude Code

- *"Vysvetli mi rozdiel medzi transakčným a marketing emailom. Pre rezervácie posielame
  transakčné — čo to znamená pre GDPR?"*
- *"Ak Resend padne (downtime), moja rezervácia sa vytvorila ale email neprišiel. Ako spravím
  retry mechanism (napríklad cez cron job)?"*
- *"Ako prejdem z `onboarding@resend.dev` na vlastnú doménu? Aké DNS records potrebujem?"*

## Ďalší krok

➡️ [Task 14 — Deploy na Vercel](14-deploy-to-vercel.md)
