# Dokumentacja dla Claude Code — Site No Worries (Open Mercato)

Pakiet plików do wgrania do repozytorium aplikacji Open Mercato, w której powstanie system obsługi
klientów Site No Worries. Claude Code czyta je automatycznie (`CLAUDE.md`) albo na żądanie (`.ai/…`).

## Zawartość

| Plik | Po co |
|---|---|
| `CLAUDE.md` | Wczytywany na starcie każdej sesji Claude Code. Importuje `AGENTS.md` frameworka i dodaje krótkie reguły projektu. |
| `.ai/project/domain.md` | **Źródło prawdy o biznesie**: słownik, cennik, terminy, algorytm okresów rozliczeniowych, windykacja, rozliczenia zakończenia umowy, zasady dla JDG na prawach konsumenta. Spisane z umowy v7. |
| `.ai/project/architecture.md` | Które moduły Open Mercato wykorzystujemy, jakie 4 moduły piszemy (`lead_intakes`, `service_contracts`, `change_requests`, `subscription_billing`), przepływ end-to-end, uprawnienia. |
| `.ai/project/integrations.md` | AhaSend, Cloudflare, GitHub, Claude, KSeF, mBank, CEIDG, formularz WWW — jak i z czym się łączymy. |
| `.ai/specs/2026-10-31-snw-01-lead-intake-and-sales.md` | Spec: leady z formularza → CRM → umowa. |
| `.ai/specs/2026-10-31-snw-02-service-contracts.md` | Spec: umowy, realizacja, okresy, Okres Dalszy, prawa autorskie, zakończenie. **Zacznij od tej.** |
| `.ai/specs/2026-10-31-snw-03-change-requests.md` | Spec: Zbiorcze Zmiany, Błędy, SLA, agent AI z akceptacją człowieka, Usługi Dodatkowe. |
| `.ai/specs/2026-10-31-snw-04-subscription-billing.md` | Spec: faktury, KSeF, płatności, windykacja. |

Specyfikacje mają format szablonu Open Mercato (`SPEC-000-template.md`) i status **Draft** — każda ma listę
otwartych pytań. Claude Code nie powinien ich implementować, dopóki nie zmienisz statusu na
`Ready for implementation` (tak działa skill `om-spec-writing` / `om-implement-spec`).

## Jak zacząć

1. Utwórz aplikację z agentem Claude Code (Node 24, Yarn 4, Docker):
   ```bash
   npx create-mercato-app sitenoworries-ops --preset crm --agents claude-code --init-git
   cd sitenoworries-ops
   docker compose up -d --wait
   yarn setup
   ```
   Preset `crm` nie zawiera modułów `sales`, `customer_accounts`, `portal` ani `scheduler` — dodaj je
   w `src/modules.ts` (albo wybierz preset `classic` i potem usuń zbędne moduły skillem `om-trim-unused-modules`).
2. Skopiuj zawartość tego pakietu do katalogu głównego aplikacji. `CLAUDE.md` zastępuje wygenerowany
   (który zawiera tylko `@AGENTS.md` — nasz zachowuje ten import na początku).
3. Uruchom Claude Code w katalogu aplikacji i zacznij od:
   > „Przeczytaj `.ai/project/domain.md` i spec `snw-02`. Przejdź z nią `om-spec-writing`: uzupełnij brakujące
   > sekcje i zadaj mi otwarte pytania. Nie pisz kodu.”
4. Po odpowiedziach i statusie `Ready for implementation`:
   > „Zaimplementuj Phase 1 ze spec `snw-02` przez `om-implement-spec`.”

Kolejność wdrożenia: **SNW-02 Phase 1–2 → SNW-04 Phase 1 → SNW-02 Phase 3 → SNW-01 → SNW-03 → SNW-04 Phase 2–3.**

## Do rozstrzygnięcia przez Ciebie (najważniejsze)

- **SNW-04 Q-001:** jak wystawiamy faktury i wysyłamy do KSeF — zewnętrzny system fakturowy z API (najmniej kodu),
  moduł `sales` Open Mercato, czy własne faktury + moduł KSeF. Warto zapytać księgową.
- **SNW-03 Q-001:** skąd agent AI czyta treść strony klienta (repo GitHub, eksport CMS EmDash).
- **SNW-01 Q-001:** pola formularza i klauzula RODO na sitenoworries.com/kontakt.
- **SNW-02 Q-001/Q-002:** format numeru umowy; czy klient sam wybiera wariant w portalu.

## Utrzymanie

- Zmieniasz umowę lub cennik → najpierw popraw `.ai/project/domain.md`, potem poproś Claude o zmianę kodu.
- `yarn mercato agentic:init --update-harness` nie nadpisuje zmienionego `CLAUDE.md`; nowa wersja z frameworka
  pojawi się obok jako plik `.incoming` do ręcznego scalenia.
- Wartości w nawiasach kwadratowych w `domain.md` (np. `[24] h`) są niepotwierdzone — uzupełnij je.
