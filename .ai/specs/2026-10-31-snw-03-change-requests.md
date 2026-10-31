# SNW-03 — Zgłoszenia zmian i błędów (`change_requests`)

**Date**: 2026-10-31
**Status**: Draft

> Reguły biznesowe: `.ai/project/domain.md` § 6 (kalendarz), § 8 (zgłoszenia), § 14 (dane i AI).
> Uprawnienia do zmian zależą od Pakietu i wariantu z SNW-02.

## TLDR

Klient zgłasza w portalu (lub e-mailem, wprowadzanym przez obsługę) Zbiorczą Zmianę treści albo Błąd.
System pilnuje limitu (1 Zbiorcza Zmiana na miesiąc kalendarzowy, max 1 h), terminów (5 Dni Roboczych
na zmianę, Czas Reakcji 18/16 Godzin Roboczych na Błąd), przygotowuje szkic zmiany przez agenta AI (Claude)
i wymaga zatwierdzenia przez człowieka przed publikacją. Praca ponad limit → wycena Usługi Dodatkowej
(150 zł/h) do akceptacji Klienta.

## Problem Statement

Zmiany przychodzą w rozproszonych e-mailach, trudno udowodnić wykorzystanie limitu, terminy SLA nie są
mierzone, a ręczne wprowadzanie zmian zjada marżę (założenie biznesowe: AI przygotowuje, człowiek tylko sprawdza).

## Overview and Success Measures

- **Primary outcome:** średni czas pracy człowieka na Zbiorczą Zmianę ≤ 10 min; 100% Błędów z dotrzymanym Czasem Reakcji.
- **Leading indicators:** odsetek szkiców AI zatwierdzonych bez poprawek; liczba zgłoszeń przekraczających 1 h.
- **Baseline:** unknown.
- **Market / product reference:** helpdesk z SLA w godzinach roboczych; adoptujemy kalendarz biznesowy, odrzucamy SLA 24/7.

## Goals

- **REQ-001** — Klient tworzy w portalu Zbiorczą Zmianę (opis + załączniki) lub Błąd; obsługa może wprowadzić zgłoszenie z e-maila.
- **REQ-002** — System egzekwuje uprawnienie i limit: 1 Zbiorcza Zmiana na miesiąc kalendarzowy dla umów w Okresie Początkowym i Wariancie A; brak dla Wariantu B; komunikat i ścieżka „Usługa Dodatkowa”.
- **REQ-003** — Terminy: Zbiorcza Zmiana 5 Dni Roboczych od kompletności; Błąd — Czas Reakcji 18 h (Starter) / 16 h (Z CMS) Godzin Roboczych; alerty przed upływem.
- **REQ-004** — Agent AI przygotowuje `ChangeDraft` (diff treści) na podstawie zgłoszenia; publikacja tylko po zatwierdzeniu przez użytkownika z `change_requests.publish`.
- **REQ-005** — Rejestr czasu pracy; przekroczenie 1 h zatrzymuje pracę i tworzy wycenę Usługi Dodatkowej do akceptacji Klienta w portalu.
- **REQ-006** — Historia zgłoszeń i zużycia limitu widoczna dla Klienta i obsługi.

## Non-goals

- Automatyczny deploy na Cloudflare/GitHub bez człowieka (nigdy).
- Faza 1: publikacja wykonywana ręcznie przez człowieka w repo/CMS; integracja z GitHub (commit/PR) w fazie 3.
- Czat na żywo z klientem.

## Proposed Solution

Moduł `change_requests` z encjami zgłoszeń, szkiców, logu pracy i wycen; agent AI zdefiniowany w
`ai-agents.ts` z narzędziami w `ai-tools.ts` (odczyt treści strony z repo/eksportu CMS, zapis szkicu),
polityka `confirm-required`; komenda `publish` wymaga feature i zatwierdzenia na karcie (`prepareMutation`).
Kalendarz z `businessCalendarService` (SNW-02 Phase 1).

### Design Decisions and Alternatives

| Decision | Rationale | Alternative considered | Why rejected / deferred |
|---|---|---|---|
| Limit liczony per miesiąc kalendarzowy | brzmienie § 1 pkt 30 umowy | per okres rozliczeniowy | niezgodne z umową |
| AI tylko tworzy szkic | § 4 ust. 7 umowy: weryfikacja przed publikacją | auto-publikacja | niezgodne z umową |
| Szkic = diff treści, nie surowy kod | łatwa weryfikacja przez człowieka | pełny commit | trudny przegląd |
| SLA pauzuje poza Godzinami Roboczymi | umowa definiuje Godziny Robocze | zegar 24/7 | niezgodne z umową |

## Domain Vocabulary and Business Rules

| Term / invariant | Precise meaning or rule | Source of truth | Failure behavior |
|---|---|---|---|
| `entitlement` | `content_batch` dozwolony gdy umowa `launched` (Okres Początkowy) lub `extended` z wariantem A | SNW-02 (snapshot przy tworzeniu) | brak → 422 + propozycja Usługi Dodatkowej |
| `monthKey` | `YYYY-MM` daty utworzenia w Europe/Warsaw | `ChangeRequest` | drugie zgłoszenie w miesiącu → 422 (obsługa może scalić z pierwszym, jeśli nierozpoczęte) |
| `completeAt` | moment, gdy zestaw zmian jest kompletny (po doprecyzowaniach) | obsługa | termin 5 Dni Roboczych liczony od `completeAt` |
| `reactionDeadline` | `businessCalendarService.addBusinessHours(receivedAt, 18|16)` | snapshot Pakietu | alert T−2 h, naruszenie zapisane |
| `minutesUsed` | suma `WorkLog` dla zgłoszenia | `WorkLog` | > 60 min → blokada dalszej pracy bez zaakceptowanej wyceny |
| zakres | tylko teksty, zdjęcia, dane kontaktowe, ceny w istniejącym układzie | domain.md § 8 | poza zakresem → klasyfikacja `extra_work` |

## Users, Permissions, and Scope

| Actor | Allowed outcomes | Scope rule | Required feature IDs |
|---|---|---|---|
| Klient (portal) | tworzenie zgłoszeń, podgląd, akceptacja wyceny | tylko własne umowy | `portal.change_requests.create`, `portal.change_requests.view` |
| Obsługa | triage, doprecyzowanie, log pracy, wycena | organizacja | `change_requests.view`, `change_requests.manage` |
| Weryfikator | zatwierdzenie i publikacja szkicu | organizacja | `change_requests.publish` |
| Agent AI | odczyt zgłoszenia i treści, zapis szkicu | zakres agenta + uprawnienia użytkownika wywołującego | `confirm-required` |

## Reuse and Ownership Map

| Capability | Reuse / extend / app-own | Existing module or new module | Integration seam | Why |
|---|---|---|---|---|
| Zgłoszenia i limit | app-own | `change_requests` | — | reguły z umowy |
| Kalendarz roboczy | reuse (app) | `service_contracts` → `businessCalendarService` | DI | jeden kalendarz |
| Agent AI | reuse | `ai_assistant` | `ai-agents.ts`, `ai-tools.ts`, `prepareMutation` | standard frameworka |
| Załączniki | reuse | `attachments` | ID | — |
| Powiadomienia | reuse | `notifications` + e-mail (AhaSend) | eventy | — |
| Wiadomości z klientem | reuse | `messages` (jeśli włączony) | link z zgłoszenia | rozważyć zamiast własnych komentarzy |

## Architecture and Data Flow

```text
portal / obsługa ─► POST change request ─► ChangeRequest (status: new)
   ─► triage (obsługa): kind, kompletność ─► completeAt ─► termin 5 DR / SLA
   ─► agent AI (na żądanie obsługi): czyta zgłoszenie + treść strony ─► ChangeDraft (pending_review)
   ─► weryfikator: zatwierdza / poprawia ─► publish (faza 1: ręcznie; faza 3: commit/PR GitHub) ─► done
   ─► WorkLog > 60 min ─► ExtraWorkQuote ─► akceptacja Klienta w portalu ─► kontynuacja jako extra_work
```

## User Journeys

### Journey J-001 — Zbiorcza Zmiana
1. Klient w portalu opisuje zmiany (np. nowe ceny), dołącza zdjęcia.
2. Obsługa oznacza kompletność; system liczy termin 5 Dni Roboczych.
3. Obsługa uruchamia agenta; szkic pokazuje różnice treści.
4. Weryfikator zatwierdza; zmiana publikowana; Klient dostaje powiadomienie.
5. Drugie zgłoszenie w tym samym miesiącu → komunikat o limicie i opcja Usługi Dodatkowej.

### Journey J-002 — Błąd
1. Klient zgłasza Błąd; system wylicza Czas Reakcji w Godzinach Roboczych.
2. Obsługa potwierdza przyjęcie (to zatrzymuje licznik reakcji) i diagnozuje.
3. Błąd blokujący całą Witrynę → priorytet + szacowany termin usunięcia w portalu.

### Journey J-003 — Przekroczenie limitu
1. Log pracy przekracza 60 min → praca wstrzymana.
2. Obsługa tworzy wycenę (h × 150 zł); Klient akceptuje lub odrzuca w portalu (forma dokumentowa: zapis kto i kiedy).
3. Akceptacja → kontynuacja i pozycja do faktury (SNW-04).

## UI and Interaction Contracts

| Surface / route | Purpose and primary actions | Data source / mutations | Closest installed reference | Canonical shell / components | Required states | Requirement IDs |
|---|---|---|---|---|---|---|
| `/backend/change-requests` | kolejka z SLA, filtry | `GET /api/change_requests/requests` | `customers` list | `DataTable` | loading, empty, error | REQ-003, REQ-006 |
| `/backend/change-requests/[id]` | triage, log pracy, szkic AI, publikacja | komendy | `customers` deal detail | detail sections, `<AiChat>` | conflict, permission denied | REQ-004, REQ-005 |
| portal `/portal/changes` | nowe zgłoszenie, historia, akceptacja wyceny | portal API | portal pages | portal shell, formularz | loading, error, limit reached | REQ-001, REQ-002, REQ-005 |

## Data Models

### `ChangeRequest`

| Field | Type / nullability | Scope / index | Sensitive / encrypted | Lifecycle and validation |
|---|---|---|---|---|
| `id`, scope cols, timestamps | standard | — | no | — |
| `contract_id` | UUID | index | no | ID z SNW-02 |
| `kind` | enum `content_batch`/`bug`/`extra_work` | index | no | — |
| `month_key` | text `YYYY-MM` | unique `(contract_id, month_key)` gdzie `kind = content_batch` i nieanulowane | no | limit |
| `status` | enum `new`/`needs_info`/`ready`/`in_progress`/`pending_review`/`done`/`rejected`/`cancelled` | index | no | — |
| `received_at` / `complete_at` / `acknowledged_at` | timestamptz | — | no | SLA |
| `due_at` / `reaction_deadline_at` | timestamptz | index | no | z kalendarza |
| `description` | text | — | może zawierać dane osobowe | nie przekazywać danych z formularzy do AI |
| `package_snapshot` | jsonb | — | no | Pakiet, wariant, SLA w chwili zgłoszenia |

`ChangeDraft`, `WorkLog`, `ExtraWorkQuote` — pola do uzupełnienia w `om-spec-writing` (standardowe kolumny,
link `change_request_id`, autor, czas, kwota w groszach dla wyceny, `accepted_by_customer_user_id`, `accepted_at`).

## API, Command, and Error Contracts

| Method / command | Path / ID | Auth and feature gate | Input | Success response / event | Errors and concurrency | Requirement IDs |
|---|---|---|---|---|---|---|
| `POST` | `/api/change_requests/requests` | portal lub `change_requests.manage` | kind, opis, załączniki | 201 + `change_requests.request.created` | 422 limit/uprawnienie | REQ-001, REQ-002 |
| command | `change_requests.request.mark_complete` | `.manage` | `{ id }` | `due_at` wyliczone | 409 | REQ-003 |
| command | `change_requests.request.acknowledge` | `.manage` | `{ id }` | zatrzymanie licznika reakcji | 409 | REQ-003 |
| AI tool | `change_requests.create_draft` | agent, `confirm-required` | `{ requestId, changes }` | `ChangeDraft` | odrzucenie przez użytkownika | REQ-004 |
| command | `change_requests.draft.publish` | `.publish` | `{ draftId }` | `change_requests.request.completed` | 403 bez feature | REQ-004 |
| command | `change_requests.quote.accept` | portal | `{ quoteId }` | `change_requests.quote.accepted` | 409 już rozstrzygnięta | REQ-005 |

## Events, Jobs, Notifications, and Cross-Module Flows

| Trigger | Producer | Consumer | Side effect | Retry / idempotency / audit behavior |
|---|---|---|---|---|
| `change_requests.request.created` | API | notifications, e-mail | powiadomienie obsługi, potwierdzenie dla Klienta | — |
| sweep SLA (co 15 min) | scheduler | worker | alerty T−2 h, zapis naruszeń | idempotentne po `requestId` + progu |
| `change_requests.quote.accepted` | portal | subscription_billing | pozycja do faktury Usługi Dodatkowej | po `quoteId` |
| `service_contracts.contract.terminated` / wariant B | SNW-02 | change_requests | blokada nowych `content_batch` | — |

## Security, Privacy, and Compliance

- Agent AI dostaje tylko: treść zgłoszenia, treści strony (repo/eksport CMS), nazwę firmy. **Nigdy** danych z formularzy kontaktowych Witryny.
- Model przez konto komercyjne Anthropic z DPA (Załącznik 2 umowy).
- Akceptacja wyceny w portalu = forma dokumentowa: zapis użytkownika, czasu, treści wyceny.

## Integration Coverage

| Test ID | Level | Setup / fixture | Actions | Assertions | Requirement IDs |
|---|---|---|---|---|---|
| TEST-001 | integration | umowa w Okresie Początkowym | 2× `content_batch` w tym samym miesiącu | drugie 422 | REQ-002 |
| TEST-002 | integration | umowa wariant B | `content_batch` | 422 + propozycja extra_work | REQ-002 |
| TEST-003 | unit | — | SLA: zgłoszenie pt 16:00, Starter 18 h | termin w środę 10:00 (pt 1 h + pon 8 h + wt 8 h + śr 1 h; tydzień bez świąt) | REQ-003 |
| TEST-004 | integration | szkic AI | publish bez `.publish` | 403; z feature → done | REQ-004 |
| TEST-005 | integration | log 70 min | dalszy log bez wyceny | 422; po akceptacji OK | REQ-005 |
| TEST-006 | security | inny klient | odczyt cudzych zgłoszeń w portalu | 403/404 | REQ-006 |

## Implementation Phases

### Phase 1 — Zgłoszenia, limit, SLA
- **Depends on:** SNW-02 Phase 2 · **Requirements closed:** REQ-001, REQ-002, REQ-003, REQ-006 · **Tests:** TEST-001..003, TEST-006 · **Exit gate:** klient zgłasza zmianę w portalu, obsługa widzi termin.

### Phase 2 — Agent AI i weryfikacja
- **Depends on:** Phase 1 · **Requirements closed:** REQ-004 · **Tests:** TEST-004 · **Exit gate:** szkic AI zatwierdzony i opublikowany ręcznie na stronie testowej.

### Phase 3 — Usługi Dodatkowe i publikacja przez GitHub
- **Depends on:** Phase 2, SNW-04 Phase 1 · **Requirements closed:** REQ-005 · **Tests:** TEST-005 · **Exit gate:** wycena zaakceptowana w portalu trafia do faktury; zatwierdzony szkic tworzy PR w repo klienta.

## Requirement Traceability

| Requirement | Journey / surface | Data/API/event contracts | Phase | Tests | Acceptance criterion |
|---|---|---|---|---|---|
| REQ-001 | J-001, portal | POST requests | 1 | TEST-006 | AC-001 |
| REQ-002 | J-001 | `month_key`, entitlement | 1 | TEST-001, TEST-002 | AC-002 |
| REQ-003 | J-002 | sweep SLA | 1 | TEST-003 | AC-003 |
| REQ-004 | J-001 | AI tool, publish | 2 | TEST-004 | AC-004 |
| REQ-005 | J-003 | quote | 3 | TEST-005 | AC-005 |
| REQ-006 | portal, backend list | GET | 1 | TEST-006 | AC-006 |

## Rollout, Migration, and Rollback

E-mail kontakt@sitenoworries.com pozostaje kanałem równoległym; obsługa wprowadza zgłoszenia z e-maila do systemu.
Rollback: wyłączenie strony portalu; dane zostają.

## Risks and Tradeoffs

| Risk / tradeoff | Impact | Mitigation / detection | Residual risk |
|---|---|---|---|
| AI wprowadza błędną treść | szkoda wizerunkowa klienta | obowiązkowa weryfikacja, diff, brak auto-publikacji | niski |
| spór o „kompletność” zgłoszenia | spór o termin | zapis historii doprecyzowań i `complete_at` | średni |
| koszty API AI | marża | limit tokenów per zgłoszenie w konfiguracji | niski |

## Acceptance Criteria

- [ ] **AC-001** — Klient tworzy zgłoszenie w portalu z załącznikami.
- [ ] **AC-002** — Drugie Zbiorcze Zgłoszenie w miesiącu i każde w wariancie B jest odrzucane z propozycją Usługi Dodatkowej.
- [ ] **AC-003** — Terminy liczone w Dniach/Godzinach Roboczych, alerty przed upływem.
- [ ] **AC-004** — Żadna zmiana nie jest publikowana bez zatwierdzenia użytkownika z `change_requests.publish`.
- [ ] **AC-005** — Praca > 60 min wymaga zaakceptowanej wyceny; akceptacja trafia do faktury.
- [ ] **AC-006** — Klient widzi wyłącznie swoje zgłoszenia i zużycie limitu.
- [ ] Every listed backend surface matches its recorded Open Mercato reference and uses the canonical shell/components, shared API helpers, semantic tokens, and complete loading, empty, error, conflict, keyboard, accessibility, responsive, light-mode, and dark-mode states.
- [ ] Every affected API and UI path has self-contained integration coverage and the configured validation gate passes.

## Final Compliance Report

| Check | Status | Evidence / resolution |
|---|---|---|
| Applicable `AGENTS.md` files and routed guides/skills reviewed | fail | do wykonania (`om-create-ai-agent`, `ai-assistant`) |
| Data models, APIs, events, UI, and tests are internally consistent | fail | pola encji pomocniczych |
| Every workflow completes end to end without a catch-all integration phase | pass | fazy |
| Platform-native reuse and extension points were chosen before custom code | pass | Reuse map |
| UI contracts identify references, canonical components, and theme/state coverage | fail | layouty |
| Every phase has dependencies, bounded slices, tests, value, and an observable exit gate | pass | fazy |

Verdict: `Blocked — Q-001, Q-002, layouty, pola encji pomocniczych`

## Open Questions

| ID | Question | Owner | Blocking? | Resolution / decision date |
|---|---|---|---|---|
| Q-001 | Skąd agent czyta aktualną treść strony: repozytorium GitHub (Astro), eksport CMS EmDash, czy oba? | właściciel | yes | pending |
| Q-002 | Czy zgłoszenia z e-maila mają być importowane automatycznie (`inbox_ops`/`channel_imap`) czy ręcznie? | właściciel | no | pending |
| Q-003 | Czy w Pakiecie Z CMS Klient może też zgłaszać Zbiorcze Zmiany (umowa: tak) — potwierdzić UX. | właściciel | no | pending |
| Q-004 | Czy wariant B ma w ogóle dostęp do portalu zgłoszeń (tylko Błędy płatne)? | właściciel | no | pending |

## Changelog

| Date | Change |
|---|---|
| 2026-10-31 | Initial draft |
