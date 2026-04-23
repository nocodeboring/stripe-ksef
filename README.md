# Integracja Stripe z KSeF — przewodnik techniczny dla developerów

[![stripto.pl — automatyzacja fakturowania Stripe z KSeF](https://stripto.pl/og.png)](https://stripto.pl)

[![Stripe](https://img.shields.io/badge/Stripe-API-635BFF?logo=stripe&logoColor=white)](https://stripe.com/docs/api)
[![KSeF](https://img.shields.io/badge/KSeF-FA(3)-2C5282)](https://ksef.podatki.gov.pl/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](./LICENSE)

> Kompletny przewodnik po **integracji Stripe z Krajowym Systemem e-Faktur (KSeF)** — mapowanie płatności Stripe na faktury ustrukturyzowane FA(3), obsługa webhooków, wystawianie e-faktur w modelu B2B, najczęstsze błędy wdrożeniowe.

Od **1 kwietnia 2026 r.** obowiązek wystawiania faktur w KSeF obejmuje wszystkich czynnych podatników VAT w Polsce. Jeśli przyjmujesz płatności przez Stripe w transakcjach B2B, musisz zadbać o prawidłowy obieg faktury ustrukturyzowanej do KSeF.

Ten dokument powstał jako baza wiedzy dla projektu [**stripto.pl**](https://stripto.pl) — rozwiązania automatyzującego fakturowanie Stripe zgodnie z polskim KSeF.

---

## Spis treści

- [Dlaczego Stripe + KSeF wymaga dodatkowej warstwy?](#dlaczego-stripe--ksef-wymaga-dodatkowej-warstwy)
- [Architektura integracji](#architektura-integracji)
- [Harmonogram KSeF 2026](#harmonogram-ksef-2026)
- [Mapowanie danych Stripe → FA(3)](#mapowanie-danych-stripe--fa3)
- [Obsługa webhooków Stripe](#obsługa-webhooków-stripe)
- [Przykład: Payment Intent → e-faktura KSeF](#przykład-payment-intent--e-faktura-ksef)
- [Uwierzytelnianie w KSeF](#uwierzytelnianie-w-ksef)
- [Obsługa błędów i retry](#obsługa-błędów-i-retry)
- [Edge cases](#edge-cases)
- [FAQ](#faq)

---

## Dlaczego Stripe + KSeF wymaga dodatkowej warstwy?

Stripe natywnie obsługuje fakturowanie (Stripe Invoicing), ale **nie komunikuje się z polskim KSeF**. Faktury generowane przez Stripe:

- są PDF-ami, nie dokumentami XML w schemacie FA(3),
- nie zawierają numeru KSeF (`KSeF_ID`) ani numeru referencyjnego,
- nie trafiają do repozytorium Ministerstwa Finansów,
- nie spełniają wymogu wystawienia faktury ustrukturyzowanej.

Od 1 kwietnia 2026 faktura B2B wystawiona poza KSeF **nie jest fakturą w rozumieniu ustawy o VAT**. Trzeba więc zbudować warstwę pośrednią, która po zdarzeniu płatności w Stripe wystawi fakturę w KSeF i powiąże jej numer z identyfikatorem transakcji Stripe.

## Architektura integracji

Typowy pipeline wygląda tak:

```
┌─────────┐   payment_intent.succeeded   ┌───────────────┐   wyślij FA(3)   ┌─────────┐
│ Stripe  │ ───────────────────────────▶ │ Twój backend  │ ────────────────▶│  KSeF   │
└─────────┘                              │  (stripto.pl) │                  └────┬────┘
                                         └───────┬───────┘                       │
                                                 │       KSeF_ID + UPO           │
                                                 │◀──────────────────────────────┘
                                                 │
                                                 ▼
                                         ┌───────────────┐
                                         │  DB / CRM /   │
                                         │  e-mail do    │
                                         │  kontrahenta  │
                                         └───────────────┘
```

Kluczowe komponenty:

1. **Webhook listener** — odbiera zdarzenia `payment_intent.succeeded`, `charge.succeeded`, `invoice.paid`.
2. **Transformer** — konwertuje metadane Stripe na XML zgodny ze schemą FA(3).
3. **KSeF client** — uwierzytelnia się tokenem, wysyła sesję interaktywną lub wsadową, odbiera `KSeF_reference_number` i UPO.
4. **Reconciliation** — zapisuje mapowanie `payment_intent.id ↔ ksef_reference_number` w bazie.

## Harmonogram KSeF 2026

| Data | Kogo dotyczy |
|------|-------------|
| **1 lutego 2026** | Firmy z przychodem >200 mln zł (w 2024 r.) + obowiązek **odbierania** faktur w KSeF dla wszystkich |
| **1 kwietnia 2026** | Wszyscy pozostali czynni podatnicy VAT — obowiązek wystawiania |
| **1 stycznia 2027** | Mikrofirmy (faktura ≤ 450 zł, obrót ≤ 10 tys. zł/miesiąc) |

Faktury konsumenckie (B2C) **nie trafiają do KSeF** — pozostają w obiegu papierowym lub PDF. Dla Stripe oznacza to, że musisz rozróżnić, czy płatność była B2B (jest NIP → KSeF) czy B2C (nie ma NIP → standardowy PDF).

## Mapowanie danych Stripe → FA(3)

Poniżej najczęstsze pola. Pełny schemat FA(3) jest publikowany przez Ministerstwo Finansów.

| Pole FA(3) | Źródło w Stripe | Uwagi |
|------------|-----------------|-------|
| `P_1` (data wystawienia) | `created` z Payment Intent | ISO-8601 → `YYYY-MM-DD` |
| `P_2` (numer faktury) | własny licznik w Twojej bazie | KSeF nie akceptuje numeru Stripe jako numeru faktury |
| `P_6` (data sprzedaży) | data zakończenia usługi / `latest_charge.created` | |
| `Podmiot1` (sprzedawca) | dane z konfiguracji Twojego konta | NIP sprzedawcy — obowiązkowo |
| `Podmiot2` (nabywca) | `customer.tax_ids` + `customer.address` | NIP musi być w formacie 10 cyfr, bez `PL` |
| `P_13_1` (netto) | `amount` − VAT | Stripe trzyma kwoty w groszach |
| `P_14_1` (VAT) | `amount_details.tax` lub wyliczone | Uważaj na zaokrąglenia |
| `P_15` (brutto) | `amount` / 100 | Konwersja grosze → złote |
| `Waluta` | `currency` | Zwykle `PLN`; inne waluty wymagają kursu NBP z dnia poprzedniego |

### Pułapki mapowania

- **Zaokrąglenia groszy** — Stripe używa integer groszy, FA(3) wymaga kwot z dokładnością do 2 miejsc po przecinku. Zawsze dziel przez 100 po wszystkich obliczeniach, nie przed.
- **Waluta inna niż PLN** — wymaga przeliczenia po kursie średnim NBP z dnia roboczego poprzedzającego datę powstania obowiązku podatkowego. Stripe nie dostarcza kursów NBP.
- **Brak NIP-u nabywcy** — jeśli klient nie podał NIP, to jest transakcja B2C i nie wystawiasz faktury w KSeF.
- **Stripe Tax** — oblicza VAT automatycznie, ale kody stawek VAT w Stripe (`txcd_*`) nie mapują się 1:1 na polskie stawki (23%, 8%, 5%, 0%, zw., np.).

## Obsługa webhooków Stripe

Rekomendowane zdarzenia do nasłuchu:

- `payment_intent.succeeded` — podstawowy trigger dla jednorazowych płatności,
- `invoice.paid` — dla subskrypcji (Stripe Billing),
- `charge.refunded` — do wystawienia faktury korygującej w KSeF,
- `customer.updated` — do synchronizacji NIP-u i adresu.

Przykład w Node.js (Express):

```ts
import Stripe from 'stripe';
import express from 'express';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
const app = express();

app.post(
  '/webhooks/stripe',
  express.raw({ type: 'application/json' }),
  async (req, res) => {
    const sig = req.headers['stripe-signature']!;
    let event: Stripe.Event;

    try {
      event = stripe.webhooks.constructEvent(
        req.body,
        sig,
        process.env.STRIPE_WEBHOOK_SECRET!
      );
    } catch (err) {
      return res.status(400).send(`Webhook Error: ${(err as Error).message}`);
    }

    if (event.type === 'payment_intent.succeeded') {
      const pi = event.data.object as Stripe.PaymentIntent;
      await enqueueKsefInvoice(pi); // async job, nie blokuj webhooka
    }

    res.json({ received: true });
  }
);
```

**Zasada #1**: nigdy nie wysyłaj do KSeF synchronicznie w handlerze webhooka. Stripe oczekuje odpowiedzi < 20 s, a sesja KSeF + odbiór UPO może trwać dłużej. Użyj kolejki (BullMQ, SQS, Cloud Tasks).

## Przykład: Payment Intent → e-faktura KSeF

Uproszczony transformer (pseudokod TypeScript):

```ts
interface KsefInvoicePayload {
  sellerNip: string;
  buyerNip: string;
  buyerName: string;
  buyerAddress: Address;
  issueDate: string;        // YYYY-MM-DD
  saleDate: string;
  invoiceNumber: string;    // z Twojego licznika, nie ze Stripe
  currency: 'PLN' | string;
  lines: InvoiceLine[];
  totalNet: number;
  totalVat: number;
  totalGross: number;
}

async function buildKsefPayload(
  pi: Stripe.PaymentIntent
): Promise<KsefInvoicePayload> {
  const customer = await stripe.customers.retrieve(pi.customer as string, {
    expand: ['tax_ids'],
  }) as Stripe.Customer;

  const nip = customer.tax_ids?.data?.find(
    (t) => t.type === 'eu_vat' && t.value.startsWith('PL')
  )?.value.replace(/^PL/, '');

  if (!nip) {
    throw new Error('Brak NIP — faktura B2C, pomijam KSeF');
  }

  return {
    sellerNip: process.env.SELLER_NIP!,
    buyerNip: nip,
    buyerName: customer.name ?? '',
    buyerAddress: toPolishAddress(customer.address),
    issueDate: todayIso(),
    saleDate: toIso(pi.created),
    invoiceNumber: await nextInvoiceNumber(),
    currency: pi.currency.toUpperCase(),
    lines: await buildLines(pi),
    totalNet: toZloty(pi.amount - (pi.amount_details?.tax ?? 0)),
    totalVat: toZloty(pi.amount_details?.tax ?? 0),
    totalGross: toZloty(pi.amount),
  };
}

const toZloty = (grosze: number) => Number((grosze / 100).toFixed(2));
```

Następnie payload jest serializowany do XML wg schematu FA(3) i wysyłany do API KSeF jako sesja interaktywna lub wsadowa.

## Uwierzytelnianie w KSeF

KSeF oferuje trzy metody uwierzytelniania:

1. **Token autoryzacyjny** — generowany w aplikacji KSeF, rekomendowany dla integracji serwer-serwer.
2. **Podpis kwalifikowany** — dla osób fizycznych.
3. **Pieczęć kwalifikowana** — dla podmiotów.

Dla integracji Stripe → KSeF praktycznie zawsze używasz **tokena** wygenerowanego raz przez osobę uprawnioną (w uprawnieniach co najmniej: wystawianie faktur). Token przechowuj w secret managerze (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault), nigdy w kodzie ani w `.env` w repozytorium.

Środowiska:
- **test** — `https://ksef-test.mf.gov.pl/` — wolne limity, używaj do CI/CD,
- **produkcja** — `https://ksef.mf.gov.pl/`.

## Obsługa błędów i retry

Typowe błędy KSeF, które musisz obsłużyć:

| Kod / sytuacja | Przyczyna | Strategia |
|----------------|-----------|-----------|
| Niepoprawny NIP nabywcy | Literówka lub NIP zagraniczny | Walidacja algorytmiczna przed wysyłką, fallback na B2C |
| Sesja wygasła | Token sesji ma krótki TTL | Re-login, retry z nowym tokenem |
| 429 / throttling | Za dużo requestów | Exponential backoff, kolejka |
| Błąd schematu XML | Pole niespełnia FA(3) | Walidacja XSD lokalnie przed wysyłką |
| UPO długo nie przychodzi | Kolejka MF | Poll co 30 s, max 15 min, potem alert |

**Zasada #2**: idempotencja. Każde zdarzenie Stripe może przyjść kilka razy (retry, duplicate). Użyj `event.id` jako klucza idempotentnego w swojej bazie, żeby nie wystawić dwóch faktur KSeF za tę samą płatność.

## Edge cases

### Zwroty i korekty (refundy w Stripe)

`charge.refunded` w Stripe → **faktura korygująca** w KSeF. Referencja do pierwotnego numeru KSeF jest obowiązkowa w polach korekty FA(3).

### Subskrypcje Stripe Billing

Każda faktura cykliczna (`invoice.paid`) generuje osobną e-fakturę w KSeF. Uważaj na:
- proraty (zmiana planu w trakcie okresu),
- kredyty (Customer Balance),
- kupony i rabaty — muszą być widoczne w pozycjach FA(3).

### Transakcje w walutach obcych

FA(3) wymaga kwoty w PLN niezależnie od waluty rozliczeniowej. Dołącz kurs NBP z dnia poprzedzającego powstanie obowiązku podatkowego. W praktyce: cron raz dziennie pobiera kursy z API NBP (`https://api.nbp.pl/api/exchangerates/tables/A/`).

### Płatności w trybie delayed capture

Jeśli używasz `capture_method: manual`, obowiązek podatkowy powstaje w momencie `capture`, nie `authorize`. Nasłuchuj na `payment_intent.amount_capturable_updated` i `charge.captured`.

### Anonimowi klienci (Stripe bez Customer)

Brak obiektu Customer = brak szans na zdobycie NIP → traktuj jako B2C. Jeśli potrzebujesz faktury, wymuś utworzenie Customera z NIP-em przed płatnością (np. w Stripe Checkout z `customer_creation: 'always'` i `tax_id_collection: { enabled: true }`).

## FAQ

**Czy Stripe samodzielnie wyśle fakturę do KSeF?**
Nie. Stripe nie ma wbudowanej integracji z polskim KSeF. Potrzebujesz własnego backendu lub zewnętrznego rozwiązania (np. [stripto.pl](https://stripto.pl)).

**Czy faktura w PDF ze Stripe Invoicing wystarczy po 1 kwietnia 2026?**
Nie w transakcjach B2B. Po tej dacie fakturą w rozumieniu ustawy o VAT jest tylko dokument w KSeF. PDF można wysłać kontrahentowi dodatkowo, ale dokumentem prawnym jest XML z KSeF.

**Co z klientami z UE spoza Polski?**
Kontrahenci bez polskiego NIP nie są objęci obowiązkiem odbioru przez KSeF. Wystawiasz dla nich zwykłą e-fakturę PDF z odwrotnym obciążeniem lub w procedurze OSS, poza systemem.

**Jak często muszę odświeżać token KSeF?**
Token sesji interaktywnej wygasa po stosunkowo krótkim czasie (kilkadziesiąt minut). Token autoryzacyjny (do generowania sesji) jest długoterminowy. W praktyce: refreshuj sesję przed każdą kampanią wysyłek lub trzymaj persistent session z auto-refresh.

**Czy można używać Stripe Tax razem z KSeF?**
Tak, ale traktuj Stripe Tax jako źródło kwoty VAT, a nie źródło stawki. Mapowanie `txcd_*` na polskie stawki musisz zrobić sam.

**Czy `test mode` Stripe działa z testem KSeF?**
Tak, i tak należy robić — Stripe test mode + KSeF test environment (`ksef-test.mf.gov.pl`). Nigdy nie mieszaj produkcyjnego Stripe z testowym KSeF (i odwrotnie).

---

## Zobacz też

- [stripto.pl](https://stripto.pl) — automatyzacja fakturowania Stripe + KSeF
- [Oficjalna dokumentacja KSeF](https://ksef.podatki.gov.pl/)
- [Stripe API reference](https://stripe.com/docs/api)
- [Schemat FA(3) — Ministerstwo Finansów](https://www.podatki.gov.pl/ksef/)

## Licencja

MIT — patrz [LICENSE](./LICENSE).

## Wsparcie

Masz pytanie o integrację Stripe z KSeF lub znalazłeś błąd w tym przewodniku? Otwórz issue albo napisz przez [stripto.pl/kontakt](https://stripto.pl).

---

**Słowa kluczowe:** Stripe KSeF, integracja Stripe KSeF, Stripe faktury Polska, KSeF webhook Stripe, FA(3) Stripe, Payment Intent KSeF, Stripe Invoicing KSeF, faktura ustrukturyzowana Stripe, API KSeF Node.js, Stripe Polska VAT.
