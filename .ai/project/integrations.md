# Site No Worries — integracje zewnętrzne

> Klucze i hasła wyłącznie w zmiennych środowiskowych / `integrations` (credentials service). Nigdy w repo,
> nigdy w logach, nigdy w promptach do AI. Przed użyciem prawdziwych kluczy zapytaj właściciela.

| Usługa | Do czego | Jak w Open Mercato | Uwagi prawne |
|---|---|---|---|
| **AhaSend** (AhaSend B.V., Amsterdam) | e-maile transakcyjne: faktury, przypomnienia, powiadomienia o zgłoszeniach | `channel_smtp` (AhaSend udostępnia SMTP) albo dedykowany pakiet providera, jeśli SMTP nie wystarczy | dane w EOG; DPA podpisany przez właściciela |
| **Cloudflare** (Pages, D1/R2, DNS) | hosting Witryn klientów i bazy CMS EmDash | faza 1: brak integracji (operacje ręczne); później pakiet providera: status deploymentu, zawieszenie/przywrócenie Witryny | DPF + SCC |
| **GitHub** | repozytoria Witryn klientów, kopie zapasowe | faza 1: link do repo na umowie; później: publikacja zatwierdzonej zmiany jako commit/PR | DPF + SCC; usunąć repo po zakończeniu umowy |
| **Anthropic (Claude)** | przygotowanie szkiców Zbiorczych Zmian | `ai_assistant` (provider/model w konfiguracji) | tylko konto komercyjne (API/Team) z DPA; **bez danych z formularzy** |
| **KSeF** (Ministerstwo Finansów) | e-faktury B2B | decyzja otwarta (spec 04): zewnętrzny system fakturowy z KSeF **albo** moduł społecznościowy `@fastwhitecat/integration-ksef-direct` (zweryfikuj wersję i licencję) | obowiązek od 1.04.2026; kary od 1.01.2027 |
| **mBank** | wpływy od klientów | faza 1: ręczne oznaczanie płatności lub import wyciągu (CSV/MT940) przez `data_sync`/`sync_excel` | numer rachunku tylko z konfiguracji |
| **CEIDG** (biznes.gov.pl) | weryfikacja JDG, PKD | faza 1: ręcznie; później API CEIDG (wymaga tokenu) | zapis PKD = dowód statusu konsumenckiego |
| **sitenoworries.com** (Astro) | formularz kontaktowy → leady | `lead_intakes`: publiczny endpoint POST z tokenem/HMAC i rate-limit | zgoda RODO zapisana z treścią klauzuli |

## Zasady dla każdej integracji

1. Provider w osobnym pakiecie/module integracji (skill `om-integration-builder`), health check, credentials
   przez `integrations`.
2. Wywołania zewnętrzne tylko w workerach kolejki, idempotentne (klucz: np. `billingPeriodId`,
   `changeRequestId`), z ponowieniami i logowaniem błędów przez `reportError`.
3. Awaria integracji **nie** blokuje zapisu domenowego — status `pending_sync` + ponowienie + powiadomienie.
4. Webhooki przychodzące przez moduł `webhooks` (weryfikacja podpisu).
