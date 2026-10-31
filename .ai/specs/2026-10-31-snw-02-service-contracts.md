# SNW-02 — Umowy i abonamenty (`service_contracts`)

**Date**: 2026-10-31
**Status**: Draft

> Reguły biznesowe: `.ai/project/domain.md` (§ 2–7, 10–13). Ta specyfikacja ich nie powtarza — odwołuje się
> do nich. Zależności: SNW-01 (Deal wygrany → umowa) jest opcjonalna; SNW-04 konsumuje eventy okresów.

## TLDR

Moduł `service_contracts` prowadzi umowę Klienta od podpisania przez realizację (projekt, wdrożenie, Tury
Zmian, odbiór) do Dnia Uruchomienia, generuje okresy rozliczeniowe Okresu Początkowego i Dalszego, pilnuje
przypomnień 60/30 dni i wyboru wariantu A/B/C, Warunku przejścia praw autorskich oraz liczy rozliczenia przy
wcześniejszym zakończeniu. Reużywa `customers` (Company/Person/Deal), `scheduler`, `notifications`,
`attachments`, `audit_logs`, portal.

## Problem Statement

Dziś umowy żyją w plikach .docx i w pamięci właściciela. Terminy (7 dni na zapłatę, 5/10 Dni Roboczych,
4 miesiące na uruchomienie, przypomnienia 60/30 dni, 7/14 dni na prawa autorskie) łatwo przeoczyć, a każde
przeoczenie ma skutek prawny lub finansowy (np. utrata prawa do automatycznego przejścia na wariant A,
spór o termin odstąpienia konsumenta).

## Overview and Success Measures

- **Primary outcome:** 100% aktywnych umów w systemie z poprawnym kalendarzem okresów i terminów; zero
  przeoczonych przypomnień 60/30 dni.
- **Leading indicators:** liczba umów z kompletnym snapshotem cen i PKD; liczba alertów terminów obsłużonych przed czasem.
- **Baseline:** unknown — pomiar po migracji pierwszych umów.
- **Market / product reference:** moduły subskrypcji w systemach billingowych (anchor date billing) — przyjmujemy
  kotwicę dnia miesiąca; odrzucamy rozliczenia proporcjonalne (umowa ich nie przewiduje).

## Goals

- **REQ-001** — Właściciel zakłada umowę dla Company/Person z Pakietem, formą podpisu, statusem konsumenckim i snapshotem cen.
- **REQ-002** — System blokuje start prac do zaksięgowania Opłaty Wdrożeniowej oraz (dla konsumenta) do 14 dni lub `earlyStartRequestedAt`.
- **REQ-003** — System prowadzi etapy realizacji i Tury Zmian z terminami w Dniach Roboczych i licznikiem opóźnień Klienta.
- **REQ-004** — Po odbiorze system ustala Dzień Uruchomienia i generuje okresy rozliczeniowe zgodnie z algorytmem z domain.md § 7.
- **REQ-005** — System wysyła przypomnienia 60 i 30 dni przed końcem Okresu Początkowego i obsługuje wybór wariantu A/B/C (także z portalu).
- **REQ-006** — System wykrywa spełnienie Warunku i pilnuje terminów 7 dni (kod) i 14 dni (umowa przeniesienia praw przy formie dokumentowej).
- **REQ-007** — Kalkulator rozliczeń zakończenia umowy dla scenariuszy z domain.md § 12, z zatwierdzeniem przez człowieka.
- **REQ-008** — Klient widzi w portalu status umowy, okresy, terminy i może wybrać wariant Okresu Dalszego.

## Non-goals

- Generowanie i podpisywanie dokumentu umowy (.docx/PDF) — faza późniejsza; teraz załącznik w `attachments`.
- Wystawianie faktur i windykacja (SNW-04).
- Obsługa zgłoszeń (SNW-03).
- Automatyczna waloryzacja CPI — tylko przypomnienie dla właściciela.

## Proposed Solution

Moduł aplikacji `service_contracts` z encjami domenowymi, trzema czystymi serwisami w DI
(`businessCalendarService`, `billingPeriodService`, `terminationSettlementService`), komendami z undo,
`makeCrudRoute` dla CRUD, eventami domenowymi oraz zadaniami schedulera (dzienny sweep terminów).

### Design Decisions and Alternatives

| Decision | Rationale | Alternative considered | Why rejected / deferred |
|---|---|---|---|
| `BillingPeriod` jako materializowane wiersze | audyt, idempotencja faktur po `billingPeriodId`, łatwy podgląd w portalu | liczenie okresów w locie | brak stabilnego ID dla faktury i przypomnień |
| Generowanie okresów „rolling” (bieżący + następny) przez sweep | Okres Dalszy jest bezterminowy | wygenerowanie 12 z góry i potem ręcznie | ręczne kroki = ryzyko przeoczenia |
| Snapshot cen na umowie | zmiana cennika nie zmienia zawartych umów | odczyt cennika na bieżąco | narusza umowę |
| Umowa linkuje `companyId`/`personId`/`dealId` | zakaz relacji ORM między modułami | relacja ORM do `customers` | łamie zasady frameworka |
| Akcje prawne (rozwiązanie, wariant A bez przypomnień) jako propozycje do zatwierdzenia | ryzyko prawne | pełna automatyzacja | domain.md § 15 |

## Domain Vocabulary and Business Rules

Pełny słownik: `.ai/project/domain.md` § 2. Niezmienniki implementacyjne:

| Term / invariant | Precise meaning or rule | Source of truth | Failure behavior |
|---|---|---|---|
| `anchorDay` | dzień miesiąca `launchDate`; nie zmienia się nigdy po ustaleniu | `ServiceContract.launchDate` | zmiana `launchDate` po wygenerowaniu okresów → odrzuć (409) bez komendy korekty |
| `BillingPeriod.index` | 0..11 Okres Początkowy, ≥12 Okres Dalszy; unikalny `(contractId, index)` | `billingPeriodService` | duplikat → no-op (idempotencja) |
| `initialTermEnd` | `start(12) − 1 dzień` | `billingPeriodService` | — |
| `reminder60At` / `reminder30At` | `initialTermEnd + 1 − 60/30 dni`; zapis `sentAt` | sweep | niewysłane → blokada auto-wariantu A + alert |
| `rightsConditionMetAt` | późniejsza z dat: `start(12)` (dzień po końcu Okresu Początkowego) albo data zaksięgowania ostatniej z należności: 990 zł + 12 Abonamentów; odsetki pomijamy | zdarzenie z SNW-04 | brak płatności → brak przejścia |
| `consumerRights` | `true` ⇒ blokada startu prac (REQ-002) | umowa | próba startu → 422 z komunikatem |
| `unpaidImplementationPart` | wzór z domain.md § 12 | `terminationSettlementService` | wynik ujemny → 0 + log błędu |

## Users, Permissions, and Scope

| Actor | Allowed outcomes | Scope rule | Required feature IDs |
|---|---|---|---|
| Właściciel | wszystko, w tym rozwiązanie i zatwierdzenie rozliczeń | organizacja | `service_contracts.view`, `.manage`, `.terminate` |
| Obsługa | prowadzenie realizacji, Tur Zmian | organizacja | `service_contracts.view`, `.manage` |
| Klient (portal) | podgląd własnej umowy, wybór wariantu | tylko umowy z `companyId`/`personId` powiązanym z `CustomerUser` | `portal.service_contracts.view`, `portal.service_contracts.choose_variant` |

`tenantId`/`organizationId` wyłącznie z kontekstu żądania; dla portalu z sesji `CustomerUser`. Brak scope → fail closed.

## Reuse and Ownership Map

| Capability | Reuse / extend / app-own | Existing module or new module | Integration seam | Why |
|---|---|---|---|---|
| Klient (firma/osoba) | reuse | `customers` | `companyId`/`personId` + snapshot nazwy/NIP | źródło prawdy o kliencie |
| NIP, PKD, `consumerRights` | extend | `customers` custom fields (`ce.ts` w `service_contracts`) | custom fields | dane klienta, nie umowy |
| Umowa, okresy, terminy | app-own | `service_contracts` | — | brak odpowiednika w OM |
| Zadania cykliczne | reuse | `scheduler` | `schedulerService.register` w `setup.ts` | standard frameworka |
| Powiadomienia | reuse | `notifications` | `notifications.ts` | — |
| Pliki umowy, materiały | reuse | `attachments` | ID załącznika | — |

## Architecture and Data Flow

```text
backend UI / portal ──► API (makeCrudRoute / komendy) ──► ServiceContract, DeliveryMilestone, RevisionRound
scheduler (codziennie 06:00 Europe/Warsaw) ──► worker: sweep terminów ──► BillingPeriod (rolling), przypomnienia
                                                            └──► eventy service_contracts.* ──► SNW-04 / notifications / e-mail
```

- **Module boundaries:** wszystkie niezmienniki umowy w jednym module (transakcyjna spójność okresów i statusu).
- **Extension points:** custom fields na Company; widget „Umowy” na karcie firmy w `customers` (injection).
- **Alternatives considered:** przechowywanie umowy jako Deal z custom fields — odrzucone (brak okresów i terminów).
- **Compatibility:** nowe kontrakty; brak zmian w zainstalowanych modułach.

## User Journeys

### Journey J-001 — Zawarcie umowy
1. Właściciel na karcie firmy lub wygranego Deal klika „Utwórz umowę”.
2. Wybiera Pakiet, formę podpisu, status konsumencki, PKD (z CEIDG), datę zawarcia; system zapisuje snapshot cen.
3. Status `signed`; event `service_contracts.contract.signed` (SNW-04 wystawia fakturę 990 zł).
4. Błąd: brak PKD dla JDG → ostrzeżenie (nie blokada); konflikt edycji → 409 i pasek konfliktu.

### Journey J-002 — Realizacja do odbioru
1. Po zapłacie 990 zł (event z SNW-04) i spełnieniu warunku konsumenckiego obsługa uruchamia realizację.
2. System liczy terminy kamieni milowych w Dniach Roboczych; Tury Zmian rejestrowane z licznikiem (2 + 3).
3. Każda wysyłka wersji zapisuje `silenceNoticeSentAt`; brak odpowiedzi w terminie → propozycja akceptacji/odbioru do zatwierdzenia.
4. Odbiór → `launchDate` → status `launched` → wygenerowanie okresów k=0 i k=1.

### Journey J-003 — Koniec Okresu Początkowego
1. Sweep wysyła przypomnienie T−60 i T−30 (e-mail + portal) z ceną wariantów, sposobem rezygnacji i skutkiem braku odpowiedzi.
2. Klient wybiera wariant w portalu albo właściciel wpisuje decyzję z e-maila.
3. Brak decyzji i oba przypomnienia wysłane → po końcu Okresu Początkowego propozycja przejścia na A (zatwierdza właściciel).
4. Wariant C → status `ended`, uruchomienie przekazania praw/kodu (REQ-006).

### Journey J-004 — Wcześniejsze zakończenie
1. Właściciel wybiera scenariusz z domain.md § 12; kalkulator pokazuje należności, cenę wykupu praw i daty (14/30 dni).
2. Właściciel zatwierdza; status `terminated`; event dla SNW-04 (korekty/należności) i przypomnienia o wyłączeniu Witryny.

## UI and Interaction Contracts

| Surface / route | Purpose and primary actions | Data source / mutations | Closest installed reference | Canonical shell / components | Required states | Requirement IDs |
|---|---|---|---|---|---|---|
| `/backend/service-contracts` | lista umów, filtry statusu/Pakietu/terminów | `GET /api/service_contracts/contracts` | `customers` people list | `Page`, `DataTable` | loading, empty, error | REQ-001 |
| `/backend/service-contracts/create` | nowa umowa | `POST /api/service_contracts/contracts` | `customers` create | `CrudForm` | validation, conflict | REQ-001 |
| `/backend/service-contracts/[id]` | szczegóły: realizacja, okresy, terminy, Okres Dalszy, prawa, zakończenie | GET + komendy | `customers` detail (tabs) | detail sections, `DataTable` | loading, error, conflict, permission denied | REQ-002..007 |
| portal `/portal/contract` | status umowy, okresy, wybór wariantu | portal API | portal pages z `customer_accounts` | portal shell | loading, empty, error | REQ-005, REQ-008 |

Szczegóły layoutu: do uzupełnienia w trakcie `om-spec-writing` po obejrzeniu stron referencyjnych.

## Data Models

### `ServiceContract`

| Field | Type / nullability | Scope / index | Sensitive / encrypted | Lifecycle and validation |
|---|---|---|---|---|
| `id` | UUID | PK | no | immutable |
| `tenant_id` / `organization_id` | UUID | scope index | no | z kontekstu |
| `number` | text | unique per org | no | numeracja z konfiguracji |
| `company_id` / `person_id` / `deal_id` | UUID nullable | index | no | ID z `customers`, bez relacji ORM |
| `customer_snapshot` | jsonb | — | tak (NIP, adres) | nazwa, NIP, adres, PKD w dniu zawarcia |
| `package` | enum `starter`/`cms` | — | no | — |
| `price_snapshot` | jsonb | — | no | setupFee, monthlyFee, wariant A/B, stawka h, cennik etapów |
| `signed_at` | date | — | no | data zawarcia |
| `signature_form` | enum `qualified`/`documentary` | — | no | — |
| `consumer_rights` | bool | — | no | — |
| `professional_statement` | enum | — | no | `professional`/`non_professional`/`none` |
| `withdrawal_notice_sent_at` / `early_start_requested_at` | timestamptz nullable | — | no | konsument |
| `status` / `delivery_stage` | enum | index | no | domain.md § 4 |
| `launch_date` | date nullable | — | no | po ustaleniu niezmienne |
| `extended_variant` | enum `a`/`b` nullable | — | no | — |
| `client_delay_days` | int | — | no | ≥ 0 |
| `created_at`, `updated_at`, `deleted_at`, `is_active` | standard | — | no | optimistic lock na `updated_at` |

### `BillingPeriod`

| Field | Type / nullability | Scope / index | Sensitive / encrypted | Lifecycle and validation |
|---|---|---|---|---|
| `id`, scope cols | UUID | PK, scope | no | — |
| `contract_id` | UUID | unique `(contract_id, index)` | no | — |
| `index` | int | — | no | 0..11 początkowy, ≥12 dalszy |
| `starts_on` / `ends_on` / `due_on` | date | index `starts_on` | no | z `billingPeriodService` |
| `amount_net` | integer (grosze) | — | no | ze snapshotu i wariantu |
| `term` | enum `initial`/`extended` | — | no | — |
| `includes_implementation_installment` | bool | — | no | `true` dla index 0..11 (100 zł) |

Pozostałe encje (`DeliveryMilestone`, `RevisionRound`, `ExtendedTermDecision`, `ContractTermination`,
`RightsTransfer`) — szczegóły pól do uzupełnienia w `om-spec-writing`; każda ze standardowymi kolumnami.

## API, Command, and Error Contracts

| Method / command | Path / ID | Auth and feature gate | Input | Success response / event | Errors and concurrency | Requirement IDs |
|---|---|---|---|---|---|---|
| `GET`/`POST`/`PUT` | `/api/service_contracts/contracts` | auth + `.view`/`.manage` | zod w `data/validators.ts` | `service_contracts.contract.created/updated` | 400/403/409 | REQ-001 |
| command | `service_contracts.contract.start_delivery` | `.manage` | `{ contractId }` | `service_contracts.contract.delivery_started` | 422 (brak 990 zł / blokada konsumencka) | REQ-002 |
| command | `service_contracts.contract.launch` | `.manage` | `{ contractId, launchDate }` | `service_contracts.contract.launched` | 409 jeśli już ustalone | REQ-004 |
| command | `service_contracts.extended_term.choose` | `.manage` lub portal | `{ contractId, variant }` | `service_contracts.extended_term.chosen` | 422 po terminie (wymaga właściciela) | REQ-005 |
| command | `service_contracts.contract.terminate` | `.terminate` | `{ scenario, effectiveDate }` | `service_contracts.contract.terminated` | 422 niezgodny scenariusz | REQ-007 |
| `GET` | `/api/service_contracts/contracts/[id]/settlement-preview` | `.terminate` | `{ scenario, date }` | wynik kalkulatora | 400 | REQ-007 |

## Events, Jobs, Notifications, and Cross-Module Flows

| Trigger | Producer | Consumer | Side effect | Retry / idempotency / audit behavior |
|---|---|---|---|---|
| `service_contracts.contract.signed` | service_contracts | subscription_billing | faktura 990 zł | idempotencja po `contractId` |
| `subscription_billing.setup_fee.paid` | subscription_billing | service_contracts | odblokowanie startu | idempotentne ustawienie flagi |
| `service_contracts.billing_period.started` | sweep | subscription_billing | faktura okresu | po `billingPeriodId` |
| `subscription_billing.billing_period.paid` | subscription_billing | service_contracts | sprawdzenie Warunku | — |
| sweep dzienny (scheduler, cron `0 6 * * *` Europe/Warsaw) | scheduler | worker `service_contracts` | okresy rolling, przypomnienia 60/30, terminy realizacji, 4 miesiące, Warunek, 7/14 dni praw | stabilne ID harmonogramu per organizacja (wzorzec `warranty_claims`) |

## Security, Privacy, and Compliance

- **Authorization:** feature guards; portal przez `requireCustomerFeatures` i dopasowanie `CustomerUser` → `companyId`/`personId`.
- **Tenant isolation:** każdy odczyt i sweep filtrowany `organization_id`; sweep iteruje per organizacja.
- **Sensitive data:** `customer_snapshot` przez helpery szyfrowania (`findWithDecryption`).
- **Abuse and failure modes:** portal nie może wybrać wariantu po terminie bez właściciela; zmiana `launchDate` po wygenerowaniu okresów zablokowana.

## Integration Coverage

| Test ID | Level | Setup / fixture | Actions | Assertions | Requirement IDs |
|---|---|---|---|---|---|
| TEST-001 | unit | — | `billingPeriodService` dla kotwic 1, 15, 28, 29, 30, 31; lata przestępne | daty start/end/due zgodne z domain.md § 7; okres 13 kontynuuje kotwicę | REQ-004 |
| TEST-002 | unit | — | `businessCalendarService`: piątek 16:30 + 18 h, święta (24.12, Boże Ciało), zmiana czasu | poprawne terminy | REQ-003 |
| TEST-003 | unit | — | `terminationSettlementService` — każdy wiersz domain.md § 12, w tym min. 3 000 zł | kwoty co do grosza | REQ-007 |
| TEST-004 | integration | org, firma, umowa konsumencka | start prac w dniu 3 bez `earlyStartRequestedAt` | 422, status bez zmian | REQ-002 |
| TEST-005 | integration | umowa po uruchomieniu | sweep T−60, T−30, koniec okresu | 2 przypomnienia, propozycja wariantu A | REQ-005 |
| TEST-006 | security | druga organizacja / inny CustomerUser | odczyt cudzej umowy w API i portalu | 403/404, brak wycieku | REQ-008 |

## Implementation Phases

### Phase 1 — Serwisy domenowe i model danych
- **Depends on:** none
- **Outcome:** `businessCalendarService`, `billingPeriodService`, `terminationSettlementService` z testami; encje i migracje.
- **Why this order / value delivered:** cała reszta zależy od poprawnych dat i kwot.
- **Deliverables:** `di.ts`, `data/entities.ts`, `data/validators.ts`, migracja, testy TEST-001..003.
- **Requirements closed:** — (fundament REQ-003, REQ-004, REQ-007)
- **Tests:** TEST-001, TEST-002, TEST-003
- **Validation:** `yarn generate`, `yarn typecheck`, `yarn test`
- **Exit gate:** 100% przypadków tabelarycznych zielonych.

### Phase 2 — CRUD umowy i realizacja
- **Depends on:** Phase 1
- **Outcome:** umowy w backendzie, start prac z blokadami, kamienie milowe, Tury Zmian, uruchomienie.
- **Requirements closed:** REQ-001, REQ-002, REQ-003, REQ-004
- **Tests:** TEST-004
- **Exit gate:** pełny przebieg J-001 i J-002 na danych testowych.

### Phase 3 — Sweep, Okres Dalszy, prawa, zakończenie, portal
- **Depends on:** Phase 2 (+ SNW-04 Phase 1 dla eventów płatności)
- **Outcome:** przypomnienia, wybór wariantu, Warunek, kalkulator zakończenia, strona portalu.
- **Requirements closed:** REQ-005, REQ-006, REQ-007, REQ-008
- **Tests:** TEST-005, TEST-006
- **Exit gate:** J-003 i J-004 end-to-end, portal w jasnym/ciemnym motywie i na wąskim ekranie.

## Requirement Traceability

| Requirement | Journey / surface | Data/API/event contracts | Phase | Tests | Acceptance criterion |
|---|---|---|---|---|---|
| REQ-001 | J-001, `/backend/service-contracts/create` | `ServiceContract`, POST contracts | 2 | TEST-006 | AC-001 |
| REQ-002 | J-002 | `start_delivery` | 2 | TEST-004 | AC-002 |
| REQ-003 | J-002 | `DeliveryMilestone`, `RevisionRound` | 1–2 | TEST-002 | AC-003 |
| REQ-004 | J-002 | `launch`, `BillingPeriod` | 1–2 | TEST-001 | AC-004 |
| REQ-005 | J-003, portal | `extended_term.choose`, sweep | 3 | TEST-005 | AC-005 |
| REQ-006 | J-003 | `RightsTransfer`, sweep | 3 | TEST-005 | AC-006 |
| REQ-007 | J-004 | settlement-preview, terminate | 1, 3 | TEST-003 | AC-007 |
| REQ-008 | portal `/portal/contract` | portal API | 3 | TEST-006 | AC-008 |

## Rollout, Migration, and Rollback

Migracje generowane `yarn db:generate`, stosowane po akceptacji właściciela. Import istniejących umów: CLI
`yarn mercato service_contracts import` (CSV) — w fazie 2, z trybem `--dry-run`. Rollback: wyłączenie modułu
w `src/modules.ts`; dane pozostają.

## Risks and Tradeoffs

| Risk / tradeoff | Impact | Mitigation / detection | Residual risk |
|---|---|---|---|
| Błąd w algorytmie okresów | złe faktury, spór z klientem | TEST-001, jeden serwis, podgląd okresów przed pierwszą fakturą | niski |
| Sweep nie uruchomił się | przeoczone przypomnienia | idempotentny sweep nadrabia zaległe dni; alert, gdy ostatni run > 26 h | niski |
| Nieaktualna lista świąt | złe terminy SLA | konfiguracja świąt + test roczny | średni |
| Błędna klasyfikacja konsumenta | nieważne postanowienia, nieopłacona praca | PKD obowiązkowe dla JDG, domyślnie `true` w razie wątpliwości | średni |

## Acceptance Criteria

- [ ] **AC-001** — Właściciel zakłada umowę ze snapshotem cen; zmiana cennika nie zmienia istniejącej umowy.
- [ ] **AC-002** — Start prac jest odrzucany bez 990 zł oraz dla konsumenta bez zgody w pierwszych 14 dniach.
- [ ] **AC-003** — Terminy realizacji uwzględniają Dni Robocze i opóźnienia Klienta.
- [ ] **AC-004** — Dla uruchomienia 31.01 okresy zaczynają się 31.01, 28/29.02, 31.03, 30.04; okres 13 kontynuuje kotwicę.
- [ ] **AC-005** — Przypomnienia 60/30 dni wysłane i zapisane; auto-wariant A tylko gdy oba wysłane, zawsze z zatwierdzeniem.
- [ ] **AC-006** — Warunek wykrywany po zapłacie wszystkich należności; alerty 7 i 14 dni działają.
- [ ] **AC-007** — Kalkulator zgadza się co do grosza z tabelą domain.md § 12.
- [ ] **AC-008** — Klient w portalu widzi tylko swoje umowy.
- [ ] Every listed backend surface matches its recorded Open Mercato reference and uses the canonical shell/components, shared API helpers, semantic tokens, and complete loading, empty, error, conflict, keyboard, accessibility, responsive, light-mode, and dark-mode states.
- [ ] Every affected API and UI path has self-contained integration coverage and the configured validation gate passes.

## Final Compliance Report

| Check | Status | Evidence / resolution |
|---|---|---|
| Applicable `AGENTS.md` files and routed guides/skills reviewed | fail | do wykonania w `om-spec-writing` |
| Data models, APIs, events, UI, and tests are internally consistent | fail | wymaga dopisania pól encji pomocniczych |
| Every workflow completes end to end without a catch-all integration phase | pass | fazy 1–3 |
| Platform-native reuse and extension points were chosen before custom code | pass | Reuse map |
| UI contracts identify references, canonical components, and theme/state coverage | fail | layouty do uzupełnienia |
| Every phase has dependencies, bounded slices, tests, value, and an observable exit gate | pass | Implementation Phases |

Verdict: `Blocked — Q-001, Q-002, layouty UI, pola encji pomocniczych`

## Open Questions

| ID | Question | Owner | Blocking? | Resolution / decision date |
|---|---|---|---|---|
| Q-001 | Numeracja umów (format, np. `SNW/2026/001`)? | właściciel | yes | pending |
| Q-002 | Czy klient ma wybierać wariant w portalu samodzielnie, czy tylko zgłaszać wybór do zatwierdzenia? | właściciel | yes | pending |
| Q-003 | Czy importujemy istniejące umowy przed startem (ile, w jakim formacie)? | właściciel | no | pending |
| Q-004 | Godzina sweepu i czy weekendy też wysyłają przypomnienia e-mail? | właściciel | no | pending |

## Changelog

| Date | Change |
|---|---|
| 2026-10-31 | Initial draft na podstawie umowy v7 i cennika sitenoworries.com |
