# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

---

# Site No Worries — system obsługi klientów (projekt)

Aplikacja Open Mercato do obsługi klientów **Site No Worries** (strony WWW w abonamencie): leady, umowy,
abonamenty, zgłoszenia zmian i błędów, fakturowanie i windykacja, portal klienta.

Reguły frameworka z `AGENTS.md` mają pierwszeństwo w sprawach technicznych. Ten blok dodaje wiedzę
o domenie — nie zastępuje routingu, skilli ani bramek walidacji z `AGENTS.md`.

## Zanim zaczniesz

1. Przejdź routing z `AGENTS.md` (Three-Axis Context Assembler).
2. Przeczytaj `.ai/project/domain.md`, jeśli zadanie dotyczy kwot, dat, terminów, umowy, faktur
   lub zgłoszeń. To jedyne źródło reguł biznesowych.
3. Przeczytaj `.ai/project/architecture.md`, jeśli zadanie dodaje encję, moduł, event lub uprawnienie.
4. Przeczytaj `.ai/project/integrations.md`, jeśli zadanie dotyka AhaSend, Cloudflare, GitHub, Claude,
   KSeF, mBank lub formularza na sitenoworries.com.
5. Sprawdź specyfikacje w `.ai/specs/2026-10-31-snw-*.md`. Implementuj tylko spec ze statusem
   `Ready for implementation` i tylko bieżącą fazę.

## Zawsze

- Kwoty netto w PLN jako liczby całkowite w groszach (lub decimal); VAT 23% doliczany na fakturze.
- Daty biznesowe jako daty kalendarzowe w strefie `Europe/Warsaw`; znaczniki czasu w UTC.
- Okresy rozliczeniowe licz wyłącznie przez `billingPeriodService` (kotwica = dzień miesiąca z Dnia
  Uruchomienia, clamp do końca miesiąca, Okres Dalszy kontynuuje ten sam cykl).
- Dni i godziny robocze licz wyłącznie przez `businessCalendarService` (pon.–pt., 9–17, polskie święta).
- Ceny, terminy i limity czytaj z konfiguracji i snapshotu na umowie — nigdy nie wpisuj ich na sztywno.
- Joby schedulera i workery idempotentne (klucz biznesowy, np. `billingPeriodId`).
- Teksty dla użytkowników po polsku przez i18n (`pl` jako język domyślny), nigdy na sztywno.
- Każda reguła z `domain.md` dotycząca pieniędzy lub terminów ma test jednostkowy z przypadkami granicznymi.

## Zapytaj najpierw

- Zmiana kwot, terminów, limitów lub reguł z `domain.md` (to zmiana umowy, nie kodu).
- Wybór sposobu wystawiania faktur / integracji KSeF (spec 04, Q-001).
- Użycie prawdziwych kluczy: AhaSend, Cloudflare, GitHub, Anthropic, KSeF, mBank.
- Każda automatyzacja, która zawiesza Witrynę, rozwiązuje umowę, wysyła wezwanie do zapłaty
  lub publikuje zmianę bez zatwierdzenia człowieka.

## Nigdy

- Nie publikuj zmiany przygotowanej przez AI bez zatwierdzenia przez użytkownika z `change_requests.publish`.
- Nie wysyłaj do modelu AI danych z formularzy kontaktowych Witryn klientów ani danych spoza treści strony.
- Nie zaczynaj prac (status `in_progress`) dla Klienta na Prawach Konsumenta w ciągu 14 dni od zawarcia,
  jeśli nie ma `earlyStartRequestedAt`.
- Nie przechodź automatycznie na Wariant A, jeśli nie wysłano obu przypomnień (60 i 30 dni).
- Nie licz okresów od „poprzedniego okresu + 1 miesiąc” — zawsze od kotwicy.
- Nie wpisuj danych firmy, numeru rachunku ani kluczy w kod — tylko konfiguracja / env.

## Słowniczek (skrót — pełny w `domain.md`)

Pakiet `starter` (249 zł) / `cms` (349 zł) · Opłata Wdrożeniowa 990 zł · Część Wdrożeniowa 2 190 zł ·
Okres Początkowy 12 okresów · Wariant A 149/249 zł, B 50/150 zł, C rezygnacja · Zbiorcza Zmiana 1× w miesiącu
kalendarzowym, max 1 h · Usługa Dodatkowa 150 zł/h · Czas Reakcji 18 h / 16 h roboczych.
