# Site No Worries — domena i reguły biznesowe

> **Źródło prawdy:** umowa „Umowa o wykonanie i utrzymanie witryny internetowej” w wersji v7
> (2026-10-30) oraz cennik na https://sitenoworries.com/cennik. Paragrafy (§) poniżej odnoszą się
> do tej umowy. Jeśli kod, specyfikacja albo prośba w czacie są sprzeczne z tym plikiem — zatrzymaj
> się i zapytaj. Nie zgaduj reguł prawnych ani kwot.
>
> **Notacja:** samo „§ N” oznacza paragraf umowy; „domain.md § N” w specyfikacjach oznacza sekcję N tego pliku.
>
> Kwoty są **netto w PLN**. Wykonawca jest czynnym podatnikiem VAT → na fakturze doliczamy VAT 23%.
> Pieniądze przechowuj w groszach (integer) albo w typie dziesiętnym; nigdy w `float`.

## 1. Strony i kontekst

| Pole | Wartość |
|---|---|
| Wykonawca | Marek Klatt, JDG „Marek Klatt Flashnite”, marka **Site No Worries** |
| Adres | ul. Chmielna 2 lok. 31, 00-020 Warszawa |
| NIP / REGON | 5223297028 / 528519012 |
| E-mail / telefon | kontakt@sitenoworries.com / +48 576 109 697 (pon.–pt. 9:00–17:00) |
| Rachunek | mBank S.A., 70 1140 2004 0000 3702 8480 8511 |
| Status VAT | czynny podatnik VAT (KSeF obowiązkowy dla faktur B2B) |
| Klienci | firmy (spółki i JDG), głównie małe firmy usługowe; **brak klientów-konsumentów** |
| Panel klienta | https://sitenoworries.com/panel |

Aplikacja w tym repozytorium to **wewnętrzny system obsługi klientów** (CRM + umowy + abonamenty +
zgłoszenia + fakturowanie) oraz **portal klienta**. To nie jest system, który hostuje strony klientów —
strony stoją na Cloudflare Pages, kod w GitHub (patrz `integrations.md`).

## 2. Słownik (terminy z umowy → nazwy w kodzie)

| Termin z umowy | Nazwa w kodzie | Znaczenie |
|---|---|---|
| Umowa | `ServiceContract` | Jedna umowa = jeden Klient + jedna Witryna + jeden Pakiet. |
| Pakiet | `package`: `starter` \| `cms` | Starter (do 4 podstron) albo Z CMS (do 5 podstron + blog + CMS EmDash). |
| Opłata Wdrożeniowa | `setupFee` | Jednorazowo **990 zł**, płatna w 7 dni od zawarcia, **przed** rozpoczęciem prac (§ 6 ust. 3). Zaliczka, nie zadatek. |
| Abonament | `monthlyFee` | Miesięczna opłata za okres rozliczeniowy (§ 6 ust. 1, § 8). |
| Część Wdrożeniowa | `implementationPart` | **2 190 zł** = 990 zł Opłaty Wdrożeniowej + 100 zł zawarte w każdym z 12 Abonamentów (§ 6 ust. 2). |
| Wynagrodzenie | `initialTermTotal` | 990 + 12 × Abonament: Starter **3 978 zł**, Z CMS **5 178 zł**. |
| Dzień Uruchomienia | `launchDate` | Dzień udostępnienia ukończonej Witryny po odbiorze (§ 1 pkt 8). Kotwica całego cyklu rozliczeń. |
| Okres Początkowy | `initialTerm` | 12 miesięcy od Dnia Uruchomienia = 12 okresów rozliczeniowych. |
| Okres Dalszy | `extendedTerm` | Po Okresie Początkowym, wariant A lub B, czas nieoznaczony (§ 8). |
| Okres rozliczeniowy | `BillingPeriod` | Miesięczny okres liczony od Dnia Uruchomienia (§ 6 ust. 4). |
| Zbiorcza Zmiana | `ChangeRequest` (`kind: 'content_batch'`) | Raz w **miesiącu kalendarzowym**, max **1 h** pracy, tylko treści w istniejącym układzie. |
| Tura Zmian | `RevisionRound` | Jeden kompletny zestaw uwag w jednej wiadomości/zgłoszeniu w trakcie realizacji. |
| Błąd | `ChangeRequest` (`kind: 'bug'`) | Niezgodność z Projektem/Pakietem, która uniemożliwia lub istotnie utrudnia korzystanie (§ 1 pkt 2). |
| Usługa Dodatkowa | `ExtraWork` | Praca poza Pakietem; **150 zł/h** po akceptacji wyceny przez Klienta (§ 6 ust. 10). |
| Czas Reakcji | `reactionDeadline` | Starter **18**, Z CMS **16** Godzin Roboczych od prawidłowego zgłoszenia (§ 5 ust. 6). Nie jest czasem naprawy. |
| Klient na Prawach Konsumenta | `consumerRights: true` | JDG, dla której umowa nie ma charakteru zawodowego (§ 1 pkt 11, § 15 ust. 1). |
| Warunek | `rightsTransferCondition` | Upływ Okresu Początkowego **i** zapłata 990 zł + wszystkich 12 Abonamentów (§ 9 ust. 4). |
| Ważny Powód | `terminationReason` | Podstawa rozwiązania ze skutkiem natychmiastowym (§ 7 ust. 3 i 4). |

## 3. Cennik (netto, miesięcznie)

| | Starter | Z CMS |
|---|---|---|
| Opłata Wdrożeniowa (jednorazowo) | 990 zł | 990 zł |
| Abonament w Okresie Początkowym | 249 zł | 349 zł |
| Wariant A — kontynuacja Pakietu (Okres Dalszy) | 149 zł | 249 zł |
| Wariant B — sam hosting (Okres Dalszy) | 50 zł | 150 zł |
| Wariant C — rezygnacja | 0 zł | 0 zł |
| Usługi Dodatkowe | 150 zł/h | 150 zł/h |

Cennik etapów — rozliczenie przy zakończeniu umowy **przed** Dniem Uruchomienia (Zał. 1 pkt A):

| Etap osiągnięty | Należność łącznie |
|---|---|
| Projekt przedstawiony do akceptacji | 990 zł |
| Projekt zaakceptowany | 1 500 zł |
| Witryna wdrożona do przeglądu | 2 190 zł (pełna Część Wdrożeniowa) |

Ceny i kwoty trzymaj w **konfiguracji z wersjonowaniem** (np. `PriceList` z `validFrom`), nie w kodzie.
Umowa zapisuje snapshot cen z dnia zawarcia — zmiana cennika nie zmienia zawartych umów.

## 4. Cykl życia umowy

```text
lead ──► offer_sent ──► signed ──► setup_fee_paid ──► in_progress ──► launched (Okres Początkowy, 12 okresów)
                │            │                              │                 │
                ▼            ▼                              ▼                 ▼
             lost      withdrawn (konsument, 14 dni)   terminated_before_launch   extended_a / extended_b (Okres Dalszy)
                                                                               │            │
                                                                               ▼            ▼
                                                                          ended_c       terminated / ended
```

Stany `ServiceContract.status`: `draft`, `signed`, `in_progress`, `launched`, `extended`, `ended`,
`terminated`, `withdrawn`. Etap realizacji (`deliveryStage`): `briefing`, `materials_pending`,
`design_in_review`, `design_accepted`, `build_in_review`, `accepted` (odbiór).

## 5. Terminy realizacji (§ 3)

| Zdarzenie | Termin | Kto |
|---|---|---|
| Zapłata Opłaty Wdrożeniowej | 7 dni kalendarzowych od zawarcia umowy | Klient |
| Start prac | po zaksięgowaniu 990 zł; rozmowa briefingowa 15–30 min | Wykonawca |
| Materiały i inspiracje | 7 dni od rozmowy | Klient |
| Projekt strony głównej w **2 wariantach** | 5 Dni Roboczych od briefu i kompletu materiałów | Wykonawca |
| Uwagi do projektu | 2 Tury Zmian, każda 5 Dni Roboczych od otrzymania wersji | Klient |
| Wdrożenie Witryny | 10 Dni Roboczych od akceptacji projektu i kompletu materiałów | Wykonawca |
| Uwagi po wdrożeniu | 3 Tury Zmian, 5 Dni Roboczych na uwagi | Klient |
| Wprowadzenie poprawek | 5 Dni Roboczych | Wykonawca |

- Milczenie Klienta = akceptacja / odbiór **tylko jeśli** w wiadomości wysłano wyraźne pouczenie o skutku
  milczenia. System musi zapisać, że pouczenie zostało wysłane (`silenceNoticeSentAt`).
- Opóźnienia Klienta **przesuwają** terminy Wykonawcy (§ 3 ust. 7) — licz pauzy (`clientDelayDays`).
- Dodatkowe Tury Zmian ponad limit, zmiana koncepcji po akceptacji, zmiana zakresu → Usługa Dodatkowa.
- Brak współpracy Klienta > **30 dni** + wezwanie z terminem **14 dni** → Wykonawca może rozwiązać (§ 3 ust. 8).
- Dzień Uruchomienia nie nastąpił w **4 miesiące** od zawarcia z winy Klienta → Wykonawca może rozwiązać (§ 3 ust. 9).
- Opóźnienie Wykonawcy > **30 dni** + dodatkowe **14 dni** po wezwaniu → Klient może rozwiązać (§ 7 ust. 4).
- Brak współpracy Klienta przy odbiorze/domenie mimo wezwania z terminem ≥ **14 dni** → Dniem Uruchomienia
  jest dzień bezskutecznego upływu terminu (§ 1 pkt 8). Brak podpięcia domeny nie blokuje uruchomienia.

## 6. Kalendarz: dni robocze i godziny robocze

- **Dzień Roboczy** = pon.–pt. z wyłączeniem dni ustawowo wolnych w Polsce.
- **Godziny Robocze** = 9:00–17:00 w Dniu Roboczym, strefa **Europe/Warsaw** (8 h dziennie).
- Dni wolne (sprawdź aktualność ustawy przed wdrożeniem): 1.01, 6.01, Wielkanoc (niedziela), Poniedziałek
  Wielkanocny, 1.05, 3.05, Zielone Świątki (niedziela), Boże Ciało, 15.08, 1.11, 11.11, **24.12 (od 2025)**,
  25.12, 26.12. Daty ruchome licz z daty Wielkanocy.
- Kalendarz to jeden współdzielony serwis (`businessCalendarService` w DI) z testami jednostkowymi na
  granicach (piątek 16:30, święta, zmiana czasu). **Nigdy** nie licz dni roboczych inline.
- Wszystkie daty biznesowe (Dzień Uruchomienia, okresy, terminy płatności) to **daty kalendarzowe bez
  godziny** w strefie Europe/Warsaw. Znaczniki zdarzeń zapisuj w UTC.

## 7. Okresy rozliczeniowe i faktury (§ 6 ust. 4, § 8 ust. 3)

**Algorytm (jedyna poprawna implementacja):**

```text
anchorDay = dzień miesiąca z launchDate (1..31)
start(k)  = data w miesiącu (launchMonth + k), dzień = min(anchorDay, liczba dni tego miesiąca)   k = 0,1,2,...
end(k)    = start(k + 1) − 1 dzień
dueDate(k)= start(k) + 7 dni kalendarzowych
```

- Licz **zawsze od `anchorDay`**, nie od poprzedniego okresu. Uruchomienie 31.01 → 28/29.02 → **31.03** → 30.04.
- Okres Początkowy = okresy k = 0..11. Koniec Okresu Początkowego = `end(11)` = `start(12) − 1 dzień`.
- **Okres Dalszy kontynuuje ten sam cykl** (k = 12, 13, …) — ten sam dzień miesiąca, ta sama kotwica.
  Zmienia się wyłącznie kwota (wariant A/B). Nie ma okresów niepełnych ani rozliczeń proporcjonalnych.
- Abonament płatny **z góry**, w terminie 7 dni od rozpoczęcia okresu. Fakturę wystawiamy w dniu `start(k)`.
- Opłata Wdrożeniowa: osobna faktura w dniu zawarcia, termin 7 dni.
- Przykład: uruchomienie 15.01.2027 → faktury 15. każdego miesiąca; 12. okres 15.12.2027–14.01.2028;
  pierwszy okres Okresu Dalszego od 15.01.2028 (np. 149 zł w wariancie A dla Startera).
- Zawieszenie strony **nie przerywa** biegu okresów i nie zwalnia z płatności (§ 6 ust. 9).

## 8. Zgłoszenia: Zbiorcze Zmiany, Błędy, Usługi Dodatkowe

- **Zbiorcza Zmiana** (§ 1 pkt 30, § 4 ust. 7): jedna na **miesiąc kalendarzowy** (nie okres rozliczeniowy),
  jedna wiadomość e-mail lub jedno zgłoszenie w Panelu Klienta, max **1 h** pracy. Zakres: teksty, zdjęcia,
  dane kontaktowe, ceny w istniejącym układzie. **Nie** obejmuje nowych podstron, zmian struktury, nowych funkcji.
- Niewykorzystane Zbiorcze Zmiany **nie kumulują się**.
- Termin wykonania: 5 Dni Roboczych od otrzymania **kompletnego** zestawu zmian.
- Przekroczenie 1 h → Usługa Dodatkowa (150 zł/h) **wyłącznie po akceptacji wyceny** przez Klienta
  w formie dokumentowej. Bez akceptacji nie wykonujemy pracy ponad limit.
- Zmiany może przygotowywać AI (Claude), ale **każda zmiana przed publikacją jest weryfikowana przez
  człowieka** (§ 4 ust. 7). System nie może publikować zmiany bez zatwierdzenia przez użytkownika z uprawnieniem.
- Uprawnienia do Zbiorczych Zmian: Okres Początkowy (oba Pakiety) i Wariant A. **Wariant B: brak** Zbiorczych
  Zmian i opieki — każda praca to Usługa Dodatkowa. Pakiet Z CMS: Klient edytuje też sam w CMS.
- **Błąd**: zgłaszany e-mailem lub przez Panel; Czas Reakcji 18 h (Starter) / 16 h (Z CMS) Godzin Roboczych
  od prawidłowego zgłoszenia do rozpoczęcia diagnozy i potwierdzenia przyjęcia. Błędy nie zużywają Tur Zmian
  ani limitu Zbiorczej Zmiany. Błąd blokujący całą Witrynę = priorytet + informacja o szacowanym terminie.
- Usunięcie skutków zmian Klienta w CMS i zastąpienie komponentu zewnętrznego to Usługi Dodatkowe.

## 9. Płatności, windykacja, zawieszenie (§ 6 ust. 8–9, § 7 ust. 3)

| Krok | Warunek | Działanie |
|---|---|---|
| Przypomnienie | termin płatności minął | e-mail przypominający (miękki) |
| Wezwanie do zapłaty | opóźnienie > 14 dni | wezwanie w formie dokumentowej z terminem **≥ 7 dni** + uprzedzenie o dacie i skutkach zawieszenia |
| Zawieszenie | wezwanie bezskuteczne | Wykonawca **może** zawiesić utrzymanie i publiczną dostępność Witryny |
| Przywrócenie | zaksięgowanie zaległości | Witryna dostępna w ciągu **2 Dni Roboczych** |
| Rozwiązanie | zaległość > **30 dni** + wezwanie z 7-dniowym terminem | Wykonawca **może** rozwiązać umowę (§ 7 ust. 3 lit. a) |

- Zawieszenie i rozwiązanie to **decyzje człowieka**. System przygotowuje akcję i dokumenty; nie wykonuje
  ich automatycznie.
- Odsetki: transakcje handlowe → odsetki ustawowe za opóźnienie w transakcjach handlowych + rekompensata
  (ustawa z 8.03.2013); w pozostałych przypadkach art. 481 KC. Faza 1: tylko informacyjnie, bez naliczania.
- Odsetki **nie blokują** przejścia praw autorskich (§ 9 ust. 4).

## 10. Okres Dalszy (§ 8)

- Przypomnienia e-mail **60** i **30** dni przed końcem Okresu Początkowego, z ceną, sposobem rezygnacji
  i skutkiem braku odpowiedzi (wymóg: wyraźnie wskazane).
- Klient wybiera wariant najpóźniej **30 dni** przed końcem Okresu Początkowego.
- Brak wyboru mimo obu przypomnień → **Wariant A**, chyba że Klient przed końcem Okresu Początkowego
  oświadczył e-mailem, że kończy współpracę. Jeżeli któreś przypomnienie **nie zostało wysłane**, nie wolno
  automatycznie przejść na wariant A — eskaluj do człowieka.
- Wariant C: przekazanie kodu i praw bez opłat; na wniosek hosting bez opłat przez 14 dni po końcu
  Okresu Początkowego na migrację.
- Wypowiedzenie w Okresie Dalszym: Klient **1 miesiąc**, Wykonawca **3 miesiące**, ze skutkiem na **koniec
  okresu rozliczeniowego**, forma dokumentowa.
- Waloryzacja: raz w roku o średnioroczny wskaźnik CPI (GUS) za poprzedni rok, tylko gdy dodatni,
  z 30-dniowym wyprzedzeniem; Klient może wypowiedzieć na dzień wejścia zmiany. Decyzja człowieka.

## 11. Prawa autorskie i kod (§ 9)

- Do spełnienia Warunku prawa ma Wykonawca; Klient ma licencję na czas umowy.
- **Warunek** = upłynął Okres Początkowy **i** zapłacono 990 zł + 12 Abonamentów. Odsetki nie blokują.
- Wydanie kodu źródłowego + eksport treści CMS: **7 dni** od spełnienia Warunku.
- Jeśli umowy **nie** podpisano w Formie Kwalifikowanej (kwalifikowany podpis albo papier), w ciągu **14 dni**
  od Warunku trzeba podpisać osobną umowę przeniesienia praw (§ 9 ust. 12). System pilnuje tego terminu.
- Pole `signatureForm`: `qualified` | `documentary`. Podpis zaufany = `documentary`.

## 12. Zakończenie umowy przed czasem — rozliczenia (§ 7, § 10)

`unpaidImplementationPart = 2190 − 990 (jeśli zapłacone) − 100 × liczbaZapłaconychAbonamentówOkresuPoczątkowego`

| Scenariusz | Klient płaci | Prawa i kod |
|---|---|---|
| Rozwiązanie przez Wykonawcę (§ 7 ust. 3) lub przez Klienta bez winy Wykonawcy | Abonamenty wymagalne do dnia rozwiązania + `unpaidImplementationPart` (wymagalne 14 dni od rozwiązania) | Wykup: `unpaidImplementationPart` + **500 zł**; jeśli `unpaidImplementationPart` już zapłacone → 500 zł. Wyłączenie Witryny po 14 dniach na eksport. |
| Rozwiązanie przez Klienta z winy Wykonawcy (§ 7 ust. 4) | proporcjonalnie do wykonanych świadczeń; `unpaidImplementationPart` **nie** jest wymagalne | Wykup: `unpaidImplementationPart`, **bez 500 zł**; 30 dni na migrację; nie wyłączamy do zawarcia umowy przeniesienia. |
| Przed Dniem Uruchomienia (np. § 3 ust. 8–9) | cennik etapów − 990 zł; nadwyżka 990 zł → zwrot w 14 dni | — |
| Przed Dniem Uruchomienia z powodu opóźnienia Wykonawcy | nic; zwrot 990 zł w 14 dni | — |
| Sprzeciw wobec przeniesienia firmy na spółkę (§ 10 ust. 3) | jak § 7 ust. 4 (bez `unpaidImplementationPart`), do końca bieżącego okresu | Wykup: `max(3000 − sumaZapłacona, unpaidImplementationPart)`; łącznie min. **3 000 zł** |

Kalkulator rozliczeń to jedna czysta funkcja z testami tabelarycznymi dla każdego wiersza. Wynik zawsze
zatwierdza człowiek.

## 13. Klient na Prawach Konsumenta (JDG)

- Ustalany przy podpisaniu: PKD z CEIDG (zapisz kody!) + dobrowolne oświadczenie Klienta (`professionalStatement`:
  `professional` | `non_professional` | `none`). Domyślnie dla JDG spoza branży IT/marketing przyjmij `true`.
- Spółki prawa handlowego **nigdy** nie mają praw konsumenta.
- Pouczenie o odstąpieniu (Zał. 3) wysłane **przed** podpisaniem — zapisz `withdrawalNoticeSentAt`. Brak
  pouczenia wydłuża termin odstąpienia o 12 miesięcy.
- Prawo odstąpienia: **14 dni od zawarcia umowy**.
- **Twarda blokada:** dla `consumerRights = true` nie wolno rozpocząć prac w ciągu 14 dni, jeśli Klient nie
  zaznaczył żądania wcześniejszego rozpoczęcia (Zał. 1 pkt B, `earlyStartRequestedAt`).
- Wobec takiego Klienta nie stosujemy ograniczenia odpowiedzialności z § 12 ust. 2 ani wyłączenia rękojmi.

## 14. Dane osobowe i retencja

- Wykonawca jest procesorem danych użytkowników Witryn klientów (Zał. 2). Podprocesorzy: Cloudflare (hosting,
  baza CMS), GitHub (kod, kopie), AhaSend (e-maile z formularzy, EOG), Anthropic/Claude (przygotowanie zmian
  treści, **bez danych z formularzy**).
- Do AI **nie wysyłaj** danych z formularzy kontaktowych Witryn ani danych osobowych spoza treści strony.
- Kopie zapasowe: tylko **najnowsza** codzienna kopia danych CMS (nadpisywana). Po zakończeniu umowy: 14 dni
  na eksport, potem usunięcie danych i ostatniej kopii; usuń też repozytorium klienta po przekazaniu kodu
  (historia gita zawiera dawne wersje treści).
- Naruszenie ochrony danych: informacja do Klienta w ciągu [24] h od stwierdzenia (wartość w nawiasie
  do potwierdzenia przez właściciela).
- Poufność: przez czas umowy + 24 miesiące.

## 15. Czego system NIE robi bez człowieka

Podpisanie/zmiana umowy, rozwiązanie, zawieszenie Witryny, wysłanie wezwania do zapłaty, publikacja zmiany
przygotowanej przez AI, wykonanie Usługi Dodatkowej bez akceptacji wyceny, waloryzacja cen, przejście na
wariant A gdy nie wysłano obu przypomnień, wydanie kodu/praw, usunięcie danych klienta.
