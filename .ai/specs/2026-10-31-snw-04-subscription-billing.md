# SNW-04 — Fakturowanie i windykacja (`subscription_billing`)

**Date**: 2026-10-31
**Status**: Draft

> Reguły biznesowe: `.ai/project/domain.md` § 3, 7, 9, 12. Integracje: `.ai/project/integrations.md` (KSeF, mBank, AhaSend).
> **Blokujące:** Q-001 (sposób wystawiania faktur i KSeF) musi zostać rozstrzygnięte przed Phase 2.

## TLDR

Moduł `subscription_billing` wystawia faktury za Opłatę Wdrożeniową (990 zł), za każdy okres rozliczeniowy
(z góry, termin 7 dni) i za zaakceptowane Usługi Dodatkowe; wysyła je e-mailem (AhaSend) i do KSeF; rejestruje
płatności z mBanku; prowadzi windykację zgodnie z § 6 ust. 9 i § 7 ust. 3 umowy, przygotowując zawieszenie
Witryny i rozwiązanie umowy do decyzji człowieka. Zdarzenia płatności zasilają SNW-02 (start prac, Warunek praw).

## Problem Statement

Faktury cykliczne wystawiane ręcznie co miesiąc w różnych dniach (kotwica = Dzień Uruchomienia każdego klienta)
są podatne na pominięcia; od 1.04.2026 każda faktura B2B musi trafić do KSeF, a od 1.01.2027 obowiązują kary.
Brak śledzenia zaległości opóźnia windykację i obniża przepływy pieniężne.

## Overview and Success Measures

- **Primary outcome:** 100% okresów rozliczeniowych z fakturą w dniu `start(k)` i numerem KSeF.
- **Leading indicators:** DSO (średni czas zapłaty), odsetek faktur po terminie > 14 dni.
- **Baseline:** unknown.
- **Market / product reference:** polskie systemy fakturowe z API i KSeF (np. Fakturownia, inFakt, wFirma) — kandydaci do integracji (Q-001).

## Goals

- **REQ-001** — Faktura za Opłatę Wdrożeniową po `service_contracts.contract.signed` (termin 7 dni).
- **REQ-002** — Faktura za każdy okres po `service_contracts.billing_period.started`, idempotentnie po `billingPeriodId`, kwota netto + VAT 23%.
- **REQ-003** — Faktura trafia do KSeF i do Klienta e-mailem (PDF + numer KSeF); status wysyłki widoczny w backendzie i portalu.
- **REQ-004** — Rejestracja płatności (ręczna i import wyciągu mBank) z dopasowaniem po tytule/kwocie; eventy `*.paid` dla SNW-02.
- **REQ-005** — Windykacja: przypomnienie po terminie, wezwanie po 14 dniach (termin ≥ 7 dni + uprzedzenie o zawieszeniu), propozycja zawieszenia, propozycja rozwiązania po 30 dniach — każde działanie prawne zatwierdza człowiek.
- **REQ-006** — Przywrócenie Witryny: zadanie dla obsługi z terminem 2 Dni Robocze od zaksięgowania zaległości.
- **REQ-007** — Faktury za Usługi Dodatkowe z zaakceptowanych wycen (SNW-03).
- **REQ-008** — Klient w portalu widzi faktury, statusy płatności i zaległości.

## Non-goals

- Płatności online kartą/BLIK (możliwe później przez `payment_gateways`).
- Automatyczne naliczanie odsetek i rekompensaty (faza 1: informacja dla właściciela).
- Księgowość, JPK, deklaracje VAT — robi biuro rachunkowe.
- Automatyczne zawieszanie Witryny na Cloudflare (faza 1: zadanie ręczne).

## Proposed Solution

Warstwa `subscription_billing` z interfejsem `InvoiceProvider` w DI. Implementacja wybrana w Q-001:

| Opcja | Opis | Plusy | Minusy |
|---|---|---|---|
| **A (rekomendowana do rozważenia)** | zewnętrzny system fakturowy z API i KSeF; `subscription_billing` przechowuje link, numer, status | KSeF, numeracja, PDF, księgowość po stronie sprawdzonego systemu; najmniej kodu | abonament systemu, zależność od API |
| B | moduł `sales` Open Mercato (`SalesInvoice`) + integracja KSeF | wszystko w jednym systemie | `sales` wymusza Quote → Order → Invoice; abonament wymagałby zamówienia na każdy okres; dużo kodu |
| C | własna encja `Invoice` + moduł społecznościowy `@fastwhitecat/integration-ksef-direct` | pełna kontrola | odpowiedzialność za zgodność faktur z prawem po naszej stronie; ryzyko |

### Design Decisions and Alternatives

| Decision | Rationale | Alternative considered | Why rejected / deferred |
|---|---|---|---|
| `InvoiceProvider` jako port w DI | zmiana dostawcy bez zmiany domeny | wywołania API inline | łamie zasady integracji |
| Idempotencja po `billingPeriodId` / `contractId` / `quoteId` | brak duplikatów faktur przy ponowieniach | deduplikacja po kwocie | niejednoznaczne |
| Windykacja jako propozycje akcji | ryzyko prawne i relacyjne | pełna automatyzacja | domain.md § 15 |

## Domain Vocabulary and Business Rules

| Term / invariant | Precise meaning or rule | Source of truth | Failure behavior |
|---|---|---|---|
| `issueDate` | `BillingPeriod.starts_on` (abonament) / data zawarcia (990 zł) | SNW-02 | — |
| `dueDate` | `issueDate + 7 dni kalendarzowych` | config | — |
| `daysOverdue` | dni kalendarzowe od `dueDate` do dziś (Europe/Warsaw) | sweep | — |
| etap windykacji | `reminder` (≥1 dzień), `formal_notice` (>14 dni), `suspension_proposed` (po bezskutecznym wezwaniu), `termination_proposed` (>30 dni + wezwanie 7-dniowe) | `DunningCase` | każdy krok zapisany z datą i treścią |
| VAT | 23% od netto, zaokrąglenie wg przepisów | provider / kalkulator | — |
| KSeF | faktura B2B wysłana do KSeF; numer KSeF zapisany | provider | błąd → `pending_ksef`, ponowienia, alert |

## Users, Permissions, and Scope

| Actor | Allowed outcomes | Scope rule | Required feature IDs |
|---|---|---|---|
| Właściciel | wszystko, zatwierdzanie akcji windykacyjnych | organizacja | `subscription_billing.view`, `.manage`, `.suspend_site` |
| Księgowość | podgląd, płatności | organizacja | `subscription_billing.view`, `.manage` |
| Klient (portal) | podgląd faktur i płatności | własne umowy | `portal.invoices.view` |

## Reuse and Ownership Map

| Capability | Reuse / extend / app-own | Existing module or new module | Integration seam | Why |
|---|---|---|---|---|
| Faktury | integracja (Q-001) | `subscription_billing` + provider | `InvoiceProvider` (DI) | KSeF i zgodność |
| Płatności i windykacja | app-own | `subscription_billing` | eventy | reguły z umowy |
| Import wyciągów | reuse | `data_sync` / `sync_excel` | import CSV | — |
| E-mail | reuse | `communication_channels` + `channel_smtp` (AhaSend) | — | — |
| Harmonogram | reuse | `scheduler` | sweep dzienny | — |

## Architecture and Data Flow

```text
service_contracts.contract.signed / billing_period.started / change_requests.quote.accepted
   ─► worker: create invoice (idempotent) ─► InvoiceProvider: wystaw + KSeF ─► e-mail (AhaSend) + portal
mBank (CSV) / ręczny wpis ─► Payment ─► dopasowanie ─► subscription_billing.*.paid ─► service_contracts
sweep dzienny ─► DunningCase: przypomnienie → wezwanie → propozycja zawieszenia/rozwiązania ─► właściciel zatwierdza
```

## User Journeys

### Journey J-001 — Faktura cykliczna
1. W dniu `start(k)` sweep emituje start okresu; worker wystawia fakturę u providera.
2. Faktura trafia do KSeF; Klient dostaje e-mail z PDF; portal pokazuje „do zapłaty do …”.
3. Błąd providera → status `pending`, ponowienia, alert po 3 próbach.

### Journey J-002 — Zaległość
1. Dzień po terminie: przypomnienie e-mail.
2. 15. dzień zaległości: system przygotowuje wezwanie (termin 7 dni, data i skutki zawieszenia); właściciel zatwierdza wysyłkę.
3. Po bezskutecznym terminie: propozycja zawieszenia → właściciel zatwierdza → zadanie dla obsługi (Cloudflare).
4. Wpłata → zadanie przywrócenia w 2 Dni Robocze.
5. > 30 dni i bezskuteczne wezwanie 7-dniowe → propozycja rozwiązania (SNW-02 J-004).

## UI and Interaction Contracts

| Surface / route | Purpose and primary actions | Data source / mutations | Closest installed reference | Canonical shell / components | Required states | Requirement IDs |
|---|---|---|---|---|---|---|
| `/backend/billing/invoices` | lista faktur, status KSeF/e-mail, ponowienie | `GET /api/subscription_billing/invoices` | `sales` invoices list | `DataTable` | loading, empty, error | REQ-001..003 |
| `/backend/billing/payments` | płatności, import CSV, dopasowanie | API + import | `sync_excel` import | `DataTable`, import dialog | progress, error | REQ-004 |
| `/backend/billing/dunning` | sprawy windykacyjne, akcje do zatwierdzenia | komendy | `customers` todos | `DataTable`, dialog potwierdzenia | conflict, permission denied | REQ-005, REQ-006 |
| portal `/portal/invoices` | faktury i zaległości | portal API | portal pages | portal shell | loading, empty, error | REQ-008 |

## Data Models

### `Invoice`

| Field | Type / nullability | Scope / index | Sensitive / encrypted | Lifecycle and validation |
|---|---|---|---|---|
| `id`, scope cols, timestamps | standard | — | no | — |
| `source_type` / `source_id` | enum `setup_fee`/`billing_period`/`extra_work` + UUID | unique `(source_type, source_id)` | no | idempotencja |
| `contract_id` | UUID | index | no | — |
| `number` / `ksef_number` | text nullable | index | no | od providera |
| `issue_date` / `due_date` | date | index | no | — |
| `net_amount` / `vat_amount` / `gross_amount` | integer (grosze) | — | no | — |
| `status` | enum `pending`/`issued`/`sent`/`paid`/`partially_paid`/`overdue`/`cancelled` | index | no | — |
| `provider_ref` / `pdf_attachment_id` | text / UUID | — | no | — |

`Payment`, `DunningCase`, `DunningStep` — pola do uzupełnienia w `om-spec-writing` (kwota w groszach,
data księgowania, tytuł, dopasowanie do faktur, etap, treść wysłanego pisma, kto zatwierdził, kiedy).

## API, Command, and Error Contracts

| Method / command | Path / ID | Auth and feature gate | Input | Success response / event | Errors and concurrency | Requirement IDs |
|---|---|---|---|---|---|---|
| `GET` | `/api/subscription_billing/invoices` | `.view` / portal | filtry | lista | 403 | REQ-001..003, REQ-008 |
| command | `subscription_billing.invoice.retry_send` | `.manage` | `{ invoiceId }` | ponowienie | 409 jeśli `sent` | REQ-003 |
| `POST` | `/api/subscription_billing/payments` | `.manage` | kwota, data, tytuł, faktury | `subscription_billing.payment.recorded` (+ `*.paid`) | 400/409 | REQ-004 |
| command | `subscription_billing.dunning.approve_step` | `.manage` / `.suspend_site` | `{ caseId, step }` | wysyłka pisma / zadanie | 403/409 | REQ-005 |

## Events, Jobs, Notifications, and Cross-Module Flows

| Trigger | Producer | Consumer | Side effect | Retry / idempotency / audit behavior |
|---|---|---|---|---|
| `service_contracts.contract.signed` | SNW-02 | worker | faktura 990 zł | po `contractId` |
| `service_contracts.billing_period.started` | SNW-02 | worker | faktura okresu | po `billingPeriodId` |
| `change_requests.quote.accepted` | SNW-03 | worker | faktura Usługi Dodatkowej | po `quoteId` |
| `subscription_billing.setup_fee.paid` | billing | SNW-02 | odblokowanie startu prac | — |
| `subscription_billing.billing_period.paid` | billing | SNW-02 | sprawdzenie Warunku praw | — |
| sweep windykacji (dzienny) | scheduler | worker | etapy `DunningCase` | idempotentne po `(invoiceId, step)` |

## Security, Privacy, and Compliance

- Numer rachunku i dane sprzedawcy wyłącznie z konfiguracji.
- Klucze providera/KSeF w `integrations` (credentials), nigdy w logach.
- Wezwania i zawieszenie: zawsze zatwierdzenie człowieka; zapis pełnej treści i dat (dowód).
- Portal: tylko faktury umów powiązanych z `CustomerUser`.

## Integration Coverage

| Test ID | Level | Setup / fixture | Actions | Assertions | Requirement IDs |
|---|---|---|---|---|---|
| TEST-001 | integration | umowa podpisana, provider fake | event signed ×2 | jedna faktura 990 zł, due +7 dni | REQ-001 |
| TEST-002 | integration | okres k=3, provider fake | event started ×2 | jedna faktura, kwota ze snapshotu, VAT 23% | REQ-002 |
| TEST-003 | integration | provider zwraca błąd KSeF | wystawienie | `pending`, ponowienia, alert | REQ-003 |
| TEST-004 | integration | faktura po terminie | sweep dni 1, 15, 22, 31 | właściwe etapy i propozycje, brak auto-zawieszenia | REQ-005 |
| TEST-005 | integration | płatność za ostatni okres | record payment | event `*.paid`, SNW-02 ustawia Warunek | REQ-004 |
| TEST-006 | security | inny klient | portal invoices | 403/404 | REQ-008 |

## Implementation Phases

### Phase 1 — Płatności i windykacja bez providera (faktury ręczne)
- **Depends on:** SNW-02 Phase 2 · **Outcome:** rejestr należności z okresów, płatności, sweep windykacji, portal · **Requirements closed:** REQ-004, REQ-005, REQ-006, REQ-008 · **Tests:** TEST-004..006 · **Exit gate:** właściciel widzi zaległości i zatwierdza wezwanie.

### Phase 2 — Wystawianie faktur i KSeF (po Q-001)
- **Depends on:** Phase 1, decyzja Q-001 · **Requirements closed:** REQ-001, REQ-002, REQ-003 · **Tests:** TEST-001..003 · **Exit gate:** faktura testowa w środowisku testowym KSeF/providera.

### Phase 3 — Import wyciągów mBank i Usługi Dodatkowe
- **Depends on:** Phase 2, SNW-03 Phase 3 · **Requirements closed:** REQ-004 (import), REQ-007 · **Exit gate:** import wyciągu dopasowuje ≥ 90% wpłat automatycznie.

## Requirement Traceability

| Requirement | Journey / surface | Data/API/event contracts | Phase | Tests | Acceptance criterion |
|---|---|---|---|---|---|
| REQ-001 | J-001 | `Invoice(setup_fee)` | 2 | TEST-001 | AC-001 |
| REQ-002 | J-001 | `Invoice(billing_period)` | 2 | TEST-002 | AC-002 |
| REQ-003 | J-001 | provider, e-mail | 2 | TEST-003 | AC-003 |
| REQ-004 | payments | `Payment`, events | 1, 3 | TEST-005 | AC-004 |
| REQ-005 | J-002 | `DunningCase` | 1 | TEST-004 | AC-005 |
| REQ-006 | J-002 | zadanie przywrócenia | 1 | TEST-004 | AC-006 |
| REQ-007 | — | `Invoice(extra_work)` | 3 | — | AC-007 |
| REQ-008 | portal | portal API | 1 | TEST-006 | AC-008 |

## Rollout, Migration, and Rollback

Do czasu Phase 2 faktury wystawiane jak dziś (ręcznie), a system tylko śledzi należności i płatności.
Rollback: wyłączenie workerów faktur; dane pozostają.

## Risks and Tradeoffs

| Risk / tradeoff | Impact | Mitigation / detection | Residual risk |
|---|---|---|---|
| duplikat faktury | korekta, zła relacja z klientem | unikalny `(source_type, source_id)` | niski |
| awaria KSeF/providera | faktura nie wystawiona w terminie | ponowienia, alert, tryb awaryjny KSeF wg przepisów | średni |
| błędne dopasowanie wpłaty | niesłuszna windykacja | windykacja tylko po zatwierdzeniu człowieka | niski |

## Acceptance Criteria

- [ ] **AC-001** — Faktura 990 zł powstaje raz na umowę z terminem 7 dni.
- [ ] **AC-002** — Każdy okres ma dokładnie jedną fakturę w dniu `start(k)` z kwotą ze snapshotu.
- [ ] **AC-003** — Faktura ma numer KSeF i została wysłana e-mailem; błędy są ponawiane i widoczne.
- [ ] **AC-004** — Zarejestrowana płatność emituje event zużywany przez SNW-02.
- [ ] **AC-005** — Etapy windykacji zgodne z § 6 ust. 9 i § 7 ust. 3; brak automatycznego zawieszenia/rozwiązania.
- [ ] **AC-006** — Po zapłacie zaległości powstaje zadanie przywrócenia z terminem 2 Dni Robocze.
- [ ] **AC-007** — Zaakceptowana wycena Usługi Dodatkowej trafia na fakturę.
- [ ] **AC-008** — Klient widzi tylko swoje faktury.
- [ ] Every listed backend surface matches its recorded Open Mercato reference and uses the canonical shell/components, shared API helpers, semantic tokens, and complete loading, empty, error, conflict, keyboard, accessibility, responsive, light-mode, and dark-mode states.
- [ ] Every affected API and UI path has self-contained integration coverage and the configured validation gate passes.

## Final Compliance Report

| Check | Status | Evidence / resolution |
|---|---|---|
| Applicable `AGENTS.md` files and routed guides/skills reviewed | fail | `sales`, `integrations`, `om-integration-builder` do przeczytania |
| Data models, APIs, events, UI, and tests are internally consistent | fail | pola encji pomocniczych |
| Every workflow completes end to end without a catch-all integration phase | pass | fazy |
| Platform-native reuse and extension points were chosen before custom code | fail | zależy od Q-001 |
| UI contracts identify references, canonical components, and theme/state coverage | fail | layouty |
| Every phase has dependencies, bounded slices, tests, value, and an observable exit gate | pass | fazy |

Verdict: `Blocked — Q-001, Q-002, layouty, pola encji pomocniczych`

## Open Questions

| ID | Question | Owner | Blocking? | Resolution / decision date |
|---|---|---|---|---|
| Q-001 | Opcja A, B czy C wystawiania faktur i KSeF? Jeśli A — który system (i czy ma API do faktur cyklicznych)? | właściciel + księgowa | yes | pending |
| Q-002 | Format wyciągu z mBanku (CSV/MT940) i czy klienci płacą z tytułem zawierającym numer faktury? | właściciel | yes (Phase 3) | pending |
| Q-003 | Treść wzorów: przypomnienie, wezwanie do zapłaty, uprzedzenie o zawieszeniu (do akceptacji prawnika). | właściciel | yes (Phase 1) | pending |
| Q-004 | Czy faktura za okres ma być wystawiana w dniu `start(k)` czy kilka dni wcześniej (z terminem liczonym od `start(k)`)? | właściciel + księgowa | no | pending |

## Changelog

| Date | Change |
|---|---|
| 2026-10-31 | Initial draft |
