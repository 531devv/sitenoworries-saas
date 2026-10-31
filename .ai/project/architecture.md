# Site No Worries — architektura aplikacji

> Ten plik opisuje **decyzje specyficzne dla tej aplikacji**. Konwencje frameworka (encje, `makeCrudRoute`,
> `CrudForm`, ACL, eventy, migracje) są w `AGENTS.md` i `.ai/guides/` wygenerowanych przez
> `create-mercato-app` — **nie powielaj ich tutaj** i nie łam ich na podstawie tego pliku.

## 1. Zasada nadrzędna: najpierw reuse

Zanim napiszesz encję, sprawdź, czy dany rekord nie należy do zainstalowanego modułu. Kolejność:
1. zainstalowany moduł Open Mercato (konfiguracja, custom fields, `ce.ts`),
2. rozszerzenie przez UMES (extension entity w `data/extensions.ts`, enricher, widget, interceptor),
3. własny moduł aplikacji w `src/modules/<id>/` — tylko dla niezmienników, których nie ma nigdzie indziej.

## 2. Mapa odpowiedzialności

| Obszar | Właściciel | Typ | Uwagi |
|---|---|---|---|
| Firmy i osoby klientów, kontakty | `customers` (Companies, People) | reuse | NIP, REGON, PKD, `consumerRights` jako custom fields na Company/Person (`ce.ts` modułu aplikacji). |
| Lejek sprzedaży (lead → oferta → wygrana) | `customers` (Deals + pipeline) | reuse | Etapy pipeline konfigurowane w `setup.ts` modułu `lead_intakes`. |
| Przyjmowanie leadów z formularza sitenoworries.com | `lead_intakes` | **app** | Publiczny endpoint z rate-limit + idempotencja; tworzy Person/Company + Deal. |
| Umowy, Pakiety, etapy realizacji, Tury Zmian, okresy rozliczeniowe, Okres Dalszy, prawa autorskie, rozliczenia zakończenia | `service_contracts` | **app** | Serce domeny. Jedyny właściciel algorytmu okresów (domain.md § 7). |
| Zgłoszenia: Zbiorcze Zmiany, Błędy, wyceny Usług Dodatkowych, SLA | `change_requests` | **app** | Szkice zmian przez agenta AI z obowiązkową akceptacją człowieka. |
| Faktury abonamentowe, płatności, przypomnienia, windykacja | `subscription_billing` | **app** + integracja | Wystawianie faktur i KSeF — decyzja otwarta (spec 04, Q-001). |
| Konta klientów w portalu, logowanie | `customer_accounts` + `portal` | reuse | Portal: umowa, faktury, zgłoszenia, wybór wariantu. `CustomerUser` linkuje do Company/Person. |
| Zadania cykliczne (generowanie okresów, faktur, przypomnień, SLA sweep) | `scheduler` (`@open-mercato/scheduler`) | reuse | Rejestracja w `setup.ts` przez `schedulerService.register(...)`; joby idempotentne (wzorzec: `warranty_claims/setup.ts`). |
| Kolejki i workery | `queue` | reuse | Nigdy własnych kolejek. |
| Powiadomienia w panelu | `notifications` | reuse | Np. „SLA zgłoszenia mija za 2 h”, „faktura po terminie”. |
| E-maile do klientów | `communication_channels` + `channel_smtp` (AhaSend SMTP) | reuse + konfiguracja | Patrz `integrations.md`. |
| Załączniki (materiały klienta, PDF umów, faktury) | `attachments` | reuse | |
| Asystent i agenci AI | `ai_assistant` | reuse | `ai-agents.ts` / `ai-tools.ts` w `change_requests`, polityka `confirm-required`. |
| Historia zmian, audyt | `audit_logs` | reuse | |
| Dokumenty sprzedażowe (oferta/zamówienie/faktura) | `sales` | **decyzja otwarta** | `sales` wymusza przepływ Quote → Order → Invoice; patrz spec 04. |

## 3. Moduły aplikacji (`src/modules/`)

Nazwy: liczba mnoga, snake_case (konwencja frameworka).

### `lead_intakes`
- Encja `LeadSubmission` (surowe zgłoszenie z formularza, zgoda, źródło UTM, `dedupeKey`), status przetworzenia.
- Subscriber `lead_intakes.submission.received` → tworzy/łączy Person + Company, zakłada Deal na pierwszym etapie.
- Nie przechowuje danych kontaktowych dłużej niż potrzeba po utworzeniu rekordu w `customers`.

### `service_contracts`
- Encje: `ServiceContract`, `ContractPriceSnapshot` (lub pola snapshotu), `DeliveryMilestone`,
  `RevisionRound`, `BillingPeriod`, `ExtendedTermDecision`, `ContractTermination`, `RightsTransfer`.
- Linki do innych modułów **tylko przez ID + snapshot** (`companyId`, `personId`, `dealId`) — bez relacji ORM
  między modułami.
- Serwisy w DI: `businessCalendarService` (dni/godziny robocze), `billingPeriodService` (algorytm okresów),
  `terminationSettlementService` (kalkulator z domain.md § 12). Wszystkie czyste i w 100% pokryte testami.
- Eventy (przykłady): `service_contracts.contract.signed`, `service_contracts.contract.launched`,
  `service_contracts.billing_period.started`, `service_contracts.initial_term.reminder_due`,
  `service_contracts.contract.terminated`, `service_contracts.rights_transfer.condition_met`.

### `change_requests`
- Encje: `ChangeRequest` (`kind`: `content_batch` | `bug` | `extra_work`), `ChangeDraft` (szkic zmiany,
  w tym od AI), `WorkLog` (czas pracy — do limitu 1 h), `ExtraWorkQuote` (wycena + akceptacja Klienta).
- Reguły limitu i uprawnień czyta z `service_contracts` przez serwis/DI lub snapshot, nie przez relację ORM.
- Agent AI: przygotowuje `ChangeDraft`; publikacja = osobna komenda wymagająca feature
  `change_requests.publish` i zatwierdzenia na karcie (`prepareMutation`).

### `subscription_billing`
- Encje: `Invoice` (albo link do faktury w systemie zewnętrznym), `Payment`, `DunningCase`, `DunningStep`.
- Reaguje na `service_contracts.billing_period.started` → wystawia fakturę (idempotentnie po `billingPeriodId`).
- Windykacja przygotowuje akcje (wezwanie, zawieszenie, rozwiązanie) do zatwierdzenia przez człowieka.

## 4. Przepływ end-to-end

```text
Formularz WWW ──► lead_intakes ──► customers (Person/Company + Deal)
                                     │ Deal wygrany
                                     ▼
                         service_contracts: umowa (draft → signed)
                                     │ faktura 990 zł (subscription_billing) ──► płatność
                                     ▼
                         realizacja: brief → projekt (2 warianty, 2 tury) → wdrożenie (3 tury) → odbiór
                                     │ launchDate
                                     ▼
                  scheduler ─► billing_period.started ─► subscription_billing: faktura + KSeF + e-mail
                                     │                              │ brak płatności
                                     │                              ▼
                                     │                    dunning: przypomnienie → wezwanie → (człowiek) zawieszenie
                                     ▼
                   portal klienta ─► change_requests (Zbiorcza Zmiana / Błąd) ─► AI szkic ─► człowiek publikuje
                                     │
                   T−60 / T−30 dni ─► przypomnienia o wyborze wariantu ─► Okres Dalszy (A/B) albo C (przekazanie praw)
```

## 5. Konfiguracja (nie w kodzie)

Cennik, stawka godzinowa, terminy (7 dni, 5/10 Dni Roboczych, 18/16 h), limity (1 h, liczba Tur Zmian),
progi windykacji (14/7/30 dni) → `configs` modułu `service_contracts` z wartościami domyślnymi z
`domain.md` i snapshotem na umowie. Zmiana konfiguracji dotyczy tylko **nowych** umów.

## 6. Uprawnienia (feature IDs — propozycja)

| Feature | Kto |
|---|---|
| `service_contracts.view`, `service_contracts.manage` | właściciel, obsługa |
| `service_contracts.terminate` | tylko właściciel |
| `change_requests.view`, `change_requests.manage` | obsługa |
| `change_requests.publish` | osoba weryfikująca zmiany AI |
| `subscription_billing.view`, `subscription_billing.manage` | właściciel, księgowość |
| `subscription_billing.suspend_site` | tylko właściciel |
| portal: `portal.service_contracts.view`, `portal.change_requests.create`, `portal.invoices.view` | `CustomerUser` (przez `requireCustomerFeatures`) |

Przed implementacją sprawdź konwencję nazw feature w `.ai/guides/` i w module `customers` — ta tabela to
propozycja, nie kontrakt.
