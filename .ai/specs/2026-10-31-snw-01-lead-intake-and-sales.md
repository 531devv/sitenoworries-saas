# SNW-01 — Leady i sprzedaż (`lead_intakes` + `customers`)

**Date**: 2026-10-31
**Status**: Draft

> Reguły biznesowe: `.ai/project/domain.md` (§ 1, 3, 13, 14). Integracja formularza: `.ai/project/integrations.md`.

## TLDR

Zapytania z formularza na sitenoworries.com trafiają do Open Mercato jako Person/Company + Deal w lejku
„Strony WWW”. Właściciel prowadzi konsultację, wysyła ofertę (Pakiet Starter/Z CMS), a wygrany Deal tworzy
szkic umowy (SNW-02). Reużywamy `customers` (People, Companies, Deals, Activities, Todos) — nowy jest tylko
mały moduł `lead_intakes` do przyjęcia zgłoszeń z WWW.

## Problem Statement

Zapytania przychodzą e-mailem i giną; brak źródła (kampania), brak przypomnień o follow-upie i brak danych
o koszcie pozyskania klienta (CAC, założenie biznesowe: ok. 600–700 zł).

## Overview and Success Measures

- **Primary outcome:** każde zgłoszenie z formularza w CRM w < 1 min, z przypisanym etapem i zadaniem follow-up.
- **Leading indicators:** czas do pierwszego kontaktu, konwersja lead → umowa, CAC per źródło UTM.
- **Baseline:** unknown — pomiar od uruchomienia.
- **Market / product reference:** typowy lejek B2B w CRM; adoptujemy Deals z `customers`, odrzucamy osobny moduł „leads”.

## Goals

- **REQ-001** — Publiczny endpoint przyjmuje zgłoszenie z formularza (imię, firma, e-mail, telefon, wiadomość, zgoda RODO, UTM) i zwraca 202.
- **REQ-002** — Zgłoszenie tworzy lub łączy Person/Company (deduplikacja po e-mailu i NIP) i zakłada Deal na etapie „Nowe zapytanie” z zadaniem follow-up.
- **REQ-003** — Lejek „Strony WWW” z etapami: Nowe zapytanie → Konsultacja umówiona → Oferta wysłana → Negocjacje → Wygrany / Przegrany.
- **REQ-004** — Na Company: NIP, REGON, PKD, forma prawna, `consumerRights` (custom fields) i akcja „Utwórz umowę” na wygranym Deal (SNW-02).
- **REQ-005** — Raport źródeł: liczba leadów, konwersja i suma kosztów pozyskania per źródło/miesiąc.

## Non-goals

- Automatyczne pobieranie danych z CEIDG/GUS (faza późniejsza).
- Generowanie dokumentu oferty (na start: szablon e-mail).
- Marketing automation / newsletter.

## Proposed Solution

`lead_intakes`: encja `LeadSubmission`, publiczna trasa POST z weryfikacją HMAC/tokenu, rate-limit per IP,
idempotencja po `submissionId` z formularza. Worker przetwarza zgłoszenie i wywołuje komendy modułu
`customers` (nie zapisuje bezpośrednio do jego tabel). Pipeline i custom fields seedowane w `setup.ts`.

### Design Decisions and Alternatives

| Decision | Rationale | Alternative considered | Why rejected / deferred |
|---|---|---|---|
| Deals z `customers` jako lejek | gotowe UI, AI asystent, aktywności | własna encja Lead | duplikacja CRM (zakazane) |
| Asynchroniczne przetwarzanie (202 + worker) | formularz nie czeka na CRM | synchroniczny zapis | wolniejszy formularz, brak ponowień |
| Custom fields dla NIP/PKD | dane należą do klienta | pola na umowie | dane klienta powielone w każdej umowie |

## Domain Vocabulary and Business Rules

| Term / invariant | Precise meaning or rule | Source of truth | Failure behavior |
|---|---|---|---|
| `dedupeKey` | lower(e-mail) + NIP (jeśli podany) | `lead_intakes` | duplikat → dopisanie aktywności do istniejącego Deal |
| zgoda RODO | treść klauzuli + wersja + timestamp | `LeadSubmission` | brak zgody → 422, nic nie zapisujemy |
| retencja | `LeadSubmission` usuwane/anonimizowane 30 dni po przetworzeniu | config | sweep |
| CAC | koszt kampanii / liczba umów z danego źródła | ręczny wpis kosztów | brak kosztu → „n/d” |

## Users, Permissions, and Scope

| Actor | Allowed outcomes | Scope rule | Required feature IDs |
|---|---|---|---|
| Formularz WWW (system) | utworzenie zgłoszenia | stała organizacja z konfiguracji endpointu | token/HMAC (bez sesji) |
| Właściciel / obsługa | praca na Deals, raport | organizacja | feature'y `customers.*`, `lead_intakes.view` |

Organizacja dla zgłoszeń publicznych ustalana z konfiguracji tokenu (nie z payloadu).

## Reuse and Ownership Map

| Capability | Reuse / extend / app-own | Existing module or new module | Integration seam | Why |
|---|---|---|---|---|
| Osoby, firmy, deale, zadania | reuse | `customers` | komendy `customers` przez DI | źródło prawdy CRM |
| NIP/PKD/konsument | extend | custom fields (`ce.ts`) | `cf_*` | — |
| Surowe zgłoszenie WWW | app-own | `lead_intakes` | event `lead_intakes.submission.received` | audyt i ponowienia |

## Architecture and Data Flow

```text
sitenoworries.com (Astro) ─POST /api/lead_intakes/submissions (HMAC)─► LeadSubmission (status: received)
        └─► queue worker ─► customers: upsert Person/Company ─► Deal „Nowe zapytanie” + Todo „Oddzwoń w 24 h”
                         └─► notifications: „Nowe zapytanie”
```

## User Journeys

### Journey J-001 — Nowe zapytanie
1. Klient wysyła formularz; strona dostaje 202.
2. W < 1 min w CRM pojawia się Deal z zadaniem i powiadomieniem.
3. Duplikat → aktywność w istniejącym Deal.
4. Awaria workera → ponowienia; po 3 nieudanych → powiadomienie dla właściciela.

### Journey J-002 — Wygrana
1. Właściciel przesuwa Deal na „Wygrany”.
2. Akcja „Utwórz umowę” otwiera formularz SNW-02 z danymi firmy i Pakietem.

## UI and Interaction Contracts

| Surface / route | Purpose and primary actions | Data source / mutations | Closest installed reference | Canonical shell / components | Required states | Requirement IDs |
|---|---|---|---|---|---|---|
| `customers` Deals (istniejące) | lejek | installed | `customers` deals | installed | installed | REQ-003 |
| widget na Deal: „Utwórz umowę” | przejście do SNW-02 | link + prefill | `customers` deal detail injection | widget injection | permission denied | REQ-004 |
| `/backend/lead-intakes` | lista surowych zgłoszeń, ponowienie | `GET /api/lead_intakes/submissions` | `customers` list | `DataTable` | loading, empty, error | REQ-001 |
| `/backend/lead-intakes/report` | raport źródeł | agregaty | `dashboards` widgets | dashboard widgets | empty, error | REQ-005 |

## Data Models

### `LeadSubmission`

| Field | Type / nullability | Scope / index | Sensitive / encrypted | Lifecycle and validation |
|---|---|---|---|---|
| `id`, scope cols, timestamps | standard | — | no | — |
| `external_id` | text | unique per org | no | idempotencja |
| `payload` | jsonb | — | **tak** | imię, e-mail, telefon, firma, wiadomość |
| `consent` | jsonb | — | no | klauzula, wersja, czas |
| `utm` | jsonb | index `utm_source` | no | — |
| `status` | enum `received`/`processed`/`duplicate`/`failed` | index | no | — |
| `person_id` / `company_id` / `deal_id` | UUID nullable | — | no | ID z `customers` |

## API, Command, and Error Contracts

| Method / command | Path / ID | Auth and feature gate | Input | Success response / event | Errors and concurrency | Requirement IDs |
|---|---|---|---|---|---|---|
| `POST` | `/api/lead_intakes/submissions` | HMAC/token, rate-limit | zod: pola formularza + zgoda + UTM | 202 + `lead_intakes.submission.received` | 400/401/422/429 | REQ-001 |
| `GET` | `/api/lead_intakes/submissions` | auth + `lead_intakes.view` | filtry | lista | 403 | REQ-001 |
| command | `lead_intakes.submission.retry` | `lead_intakes.manage` | `{ id }` | ponowny event | 409 jeśli `processed` | REQ-002 |

## Events, Jobs, Notifications, and Cross-Module Flows

| Trigger | Producer | Consumer | Side effect | Retry / idempotency / audit behavior |
|---|---|---|---|---|
| `lead_intakes.submission.received` | API | worker | upsert CRM + Deal + Todo | idempotencja po `external_id` |
| sweep retencji (dzienny) | scheduler | worker | anonimizacja po 30 dniach | — |

## Security, Privacy, and Compliance

- Endpoint publiczny: HMAC z sekretem współdzielonym ze stroną Astro, rate-limit per IP i per e-mail, limit rozmiaru.
- `payload` szyfrowany; brak logowania treści wiadomości.
- Informacja RODO na stronie musi wskazywać Site No Worries jako administratora danych z formularza.

## Integration Coverage

| Test ID | Level | Setup / fixture | Actions | Assertions | Requirement IDs |
|---|---|---|---|---|---|
| TEST-001 | integration | org + token | POST poprawne zgłoszenie | 202, Deal + Todo po przetworzeniu | REQ-001, REQ-002 |
| TEST-002 | integration | istniejąca osoba | POST duplikat | brak nowego Deal, nowa aktywność | REQ-002 |
| TEST-003 | security | zły podpis / brak zgody / 100 żądań | POST | 401 / 422 / 429 | REQ-001 |

## Implementation Phases

### Phase 1 — Lejek i custom fields
- **Depends on:** none · **Outcome:** pipeline + pola NIP/PKD/konsument w `customers` · **Requirements closed:** REQ-003, REQ-004 (bez akcji umowy) · **Tests:** ręczny smoke + test seedów · **Exit gate:** właściciel prowadzi lead ręcznie w CRM.

### Phase 2 — Przyjmowanie zgłoszeń z WWW
- **Depends on:** Phase 1 · **Outcome:** formularz → CRM · **Requirements closed:** REQ-001, REQ-002 · **Tests:** TEST-001..003 · **Exit gate:** zgłoszenie testowe z produkcyjnej strony widoczne w CRM.

### Phase 3 — Raport źródeł i akcja „Utwórz umowę”
- **Depends on:** Phase 2, SNW-02 Phase 2 · **Requirements closed:** REQ-004, REQ-005 · **Exit gate:** raport za miesiąc z danymi.

## Requirement Traceability

| Requirement | Journey / surface | Data/API/event contracts | Phase | Tests | Acceptance criterion |
|---|---|---|---|---|---|
| REQ-001 | J-001 | POST submissions | 2 | TEST-001, TEST-003 | AC-001 |
| REQ-002 | J-001 | worker | 2 | TEST-001, TEST-002 | AC-002 |
| REQ-003 | Deals | setup.ts | 1 | smoke | AC-003 |
| REQ-004 | J-002 | ce.ts, widget | 1, 3 | smoke | AC-004 |
| REQ-005 | report | agregaty | 3 | — | AC-005 |

## Rollout, Migration, and Rollback

Formularz na stronie wysyła równolegle e-mail (jak dziś) i POST do API przez 2 tygodnie; potem e-mail wyłączony.
Rollback: wyłączenie POST na stronie.

## Risks and Tradeoffs

| Risk / tradeoff | Impact | Mitigation / detection | Residual risk |
|---|---|---|---|
| spam formularza | śmieci w CRM | HMAC + rate-limit + honeypot po stronie Astro | niski |
| awaria API | utrata leadu | równoległy e-mail w okresie przejściowym, ponowienia | niski |

## Acceptance Criteria

- [ ] **AC-001** — Zgłoszenie z formularza zwraca 202 i jest zapisane z zgodą.
- [ ] **AC-002** — W < 1 min powstaje Deal z zadaniem; duplikat nie tworzy nowego Deal.
- [ ] **AC-003** — Lejek ma 6 etapów w podanej kolejności.
- [ ] **AC-004** — Firma ma pola NIP/REGON/PKD/konsument; wygrany Deal pozwala utworzyć umowę.
- [ ] **AC-005** — Raport pokazuje leady i konwersję per źródło.
- [ ] Every listed backend surface matches its recorded Open Mercato reference and uses the canonical shell/components, shared API helpers, semantic tokens, and complete loading, empty, error, conflict, keyboard, accessibility, responsive, light-mode, and dark-mode states.
- [ ] Every affected API and UI path has self-contained integration coverage and the configured validation gate passes.

## Final Compliance Report

| Check | Status | Evidence / resolution |
|---|---|---|
| Applicable `AGENTS.md` files and routed guides/skills reviewed | fail | do wykonania w `om-spec-writing` |
| Data models, APIs, events, UI, and tests are internally consistent | pass | tabele powyżej |
| Every workflow completes end to end without a catch-all integration phase | pass | fazy 1–3 |
| Platform-native reuse and extension points were chosen before custom code | pass | Reuse map |
| UI contracts identify references, canonical components, and theme/state coverage | fail | layout raportu do uzupełnienia |
| Every phase has dependencies, bounded slices, tests, value, and an observable exit gate | pass | fazy |

Verdict: `Blocked — Q-001, Q-002, layout raportu`

## Open Questions

| ID | Question | Owner | Blocking? | Resolution / decision date |
|---|---|---|---|---|
| Q-001 | Jakie pola ma dziś formularz na sitenoworries.com/kontakt i jaka jest treść klauzuli RODO? | właściciel | yes | pending |
| Q-002 | Skąd brać koszty kampanii do CAC (ręczny wpis miesięczny, import z Ads)? | właściciel | no | pending |
| Q-003 | Czy umówienie konsultacji ma iść przez kalendarz (`planner`) czy zewnętrzne narzędzie? | właściciel | no | pending |

## Changelog

| Date | Change |
|---|---|
| 2026-10-31 | Initial draft |
