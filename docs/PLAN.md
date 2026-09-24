# Plan budowy

Kolejność kroków budowy systemu opisanego w `docs/SPEC.md` (wersja 0.2).
Jeden krok = jedna sesja: zaczynamy od `/plan`, kończymy commitem i `/clear`.
Zakres każdego kroku bierze się stąd — czego tu nie ma, tego nie budujemy.

Krok może ruszyć dopiero, gdy wszystkie punkty z jego pozycji „Wymaga” są
rozstrzygnięte w `docs/OPEN-QUESTIONS.md` lub uzupełnione w `docs/seed.yaml`.
Numery pytań (pkt N) odnoszą się do `docs/OPEN-QUESTIONS.md`.

Każdy krok, który dodaje funkcję RPC, dodaje w tej samej zmianie jej test pgTAP.
Każdy krok dotykający księgowania wskazuje w planie sesji zasady nienaruszalne w grze.

---

## Krok 0 — fundamenty dokumentacji ✅

Repozytorium, `.gitignore`, SPEC 0.2, `docs/DECISIONS.md` (ADR-0001…0009),
`docs/OPEN-QUESTIONS.md`, ten plan.

## Krok 1 — rdzeń księgi w bazie

**Cel:** schemat, który fizycznie nie pozwala złamać zasad nienaruszalnych.

**Zakres:**
- `supabase init`, konfiguracja lokalna, pgTAP.
- Migracje: `entities`, `assets`, `locations`, `location_addresses`, `accounts`,
  `operations`, `postings`, `otc_deals`, `shifts` (bez danych).
- Ograniczenia z SPEC §3: klucze naturalne, `accounts` wg ADR-0007, `amount <> 0`,
  skala `amount` ≤ `assets.scale`, constraint trigger bilansu per (operacja, spółka,
  aktywo), constraint trigger na `operations` (≥2 zapisy, reguła `TR-INTERCO` +
  `legal_basis`), `unique (source, external_id)`.
- Tylko dopisywanie: brak UPDATE/DELETE na `operations`, `postings`, `otc_deals`;
  jednorazowe zamknięcie w `shifts`; `FORCE ROW LEVEL SECURITY`; brak grantów INSERT
  dla ról klienckich.
- Widok `v_account_balances`.
- Testy pgTAP samych inwariantów: każdy z nich musi odrzucić złamanie (niezbilansowana
  operacja, operacja z jednym zapisem, dwie spółki bez `TR-INTERCO`, UPDATE/DELETE,
  za duża skala, konto `holding` bez lokalizacji itd.). Dane testowe tworzone w teście,
  nie w migracji.

**Wymaga:** nic — wszystkie punkty blokujące są rozstrzygnięte.

**Zasady w grze:** 1, 2, 3, 4, 5, 6, 7.

**Gotowe gdy:** `supabase db reset` przechodzi, `supabase test db` zielony.

## Krok 2 — dane startowe i bilans otwarcia

**Cel:** baza z prawdziwymi słownikami i stanem otwarcia.

**Zakres:**
- Generator `supabase/seed.sql` z `docs/seed.yaml` (seed osobno, nie w migracjach).
- Konta techniczne tworzone dla spółek własnych i aktywów z seeda.
- RPC `ADJ-OPENING`.
- Test: seed + bilans otwarcia dają oczekiwane salda w `v_account_balances`.

**Wymaga:** pkt 11 (spółka kasy KH), pkt 12 (lista aktywów), pkt 19 i 25 (banki,
adresy, przynależność SEJF / WALLET_HOT / EXCHANGE_OKX).

**Zasady w grze:** 1, 2, 3, 5. Klucze prywatne portfeli nie trafiają do seeda.

## Krok 3 — operacje skarbca i zmiany kasowe

**Cel:** ręczny obieg wartości w obrębie lokalizacji.

**Zakres:**
- RPC: `TR-MOVE`, `TR-BANK`, `TR-INTERCO`, `ADJ-STORNO`, `ADJ-DIFF`, `shift_open`,
  `shift_close`.
- `shift_counts` i ślepe liczenie (różnica widoczna dopiero po zapisaniu liczenia).
- `documents` z ciągłą numeracją per (spółka, typ, rok), bez PDF.
- Widok `v_transit`.

**Wymaga:** pkt 16 (autor operacji automatycznych — tylko jeśli storno ma być
wywoływane przez system), pkt 17 (`shift_id` przy ruchu między kasami).

**Zasady w grze:** 2, 3, 5, 6.

## Krok 4 — role, dostęp i uwierzytelnienie

**Cel:** każda rola widzi i może dokładnie to, co w SPEC §7.

**Zakres:** użytkownicy i role, polityki RLS po `location_id` i `entity_id`,
uprawnienia do RPC per rola, storno tylko dla `manager`, TOTP dla ról
nie-kasjerskich, sesja stanowiskowa + PIN kasjera. Testy pgTAP na widoczność
(kasjer nie widzi wyniku ani innych kas).

**Wymaga:** nic nowego.

**Zasady w grze:** 5.

## Krok 5 — szkielet frontu

**Cel:** pierwsze ekrany na RPC z kroków 2–4.

**Zakres:** Next.js App Router + TypeScript, klient Supabase bez `service_role` w
przeglądarce, jedna funkcja formatowania kwot wg `assets.scale` (`decimal.js`,
string), słownik etykiet UI po polsku, logowanie, podgląd sald własnej kasy,
formularze operacji skarbca i liczenia zmiany.

**Wymaga:** nic nowego.

**Zasady w grze:** 4, 5.

## Krok 6 — kasa Hyllet

**Cel:** `EM-ISSUE`, `EM-REDEEM`, `EM-SETTLE`, `EM-NETFEE` wg formuł ADR-0008.

**Zakres:** RPC z testami wszystkich wariantów (ta sama waluta, przewalutowanie,
`M = 0`, `M < 0` odrzucone, `S = 0`), ekran kasy KH.

**Wymaga:** pkt 11, pkt 13 (prefinansowanie, podstawa marży i udziału sieci),
pkt 20 (waluta `S`), pkt 21 (źródło kursu `r`), pkt 22 (ujemna marża).

**Zasady w grze:** 2, 4, 5.

## Krok 7 — OTC i krypto

**Cel:** transakcje OTC z rozrachunkiem per transakcja.

**Zakres:** `otc_deal_open`, `OTC-CASH`, `OTC-CRYPTO`, `OTC-FEE`, `TR-WALLET`,
`TR-TRADE`, widok `v_open_otc`, ekran kasy KZ.

**Wymaga:** pkt 12 (tokeny i sieci), kontrahenci OTC w `seed.yaml`.

**Zasady w grze:** 2, 4, 5.

## Krok 8 — import z Kantor-Logic

**Cel:** kasa walutowa zasilana wyłącznie importem, idempotentnie.

**Zakres:** `import_runs`, parser wykrywający format, `KL-BUY`, `KL-SELL`,
`KL-CASHIN`, `KL-CASHOUT`, wykrywanie usuniętych dowodów (`ADJ-STORNO` +
`KL_ROW_VANISHED`), widok `v_daily_kw`, blokada zamknięcia dnia przy niezgodności.
Test: dwukrotny import tego samego pliku nie zmienia sald.

**Wymaga:** pkt 10 (próbka eksportu w `docs/samples/`), pkt 16, pkt 23.

**Zasady w grze:** 2, 3, 7.

## Krok 9 — uzgodnienia i alerty

**Cel:** rozbieżności z rzeczywistością widoczne i blokujące, nigdy „poprawiane”.

**Zakres:** `external_balances`, odczyt sald blockchain (watch-only), giełdy
(klucz read-only), Hyllet i banku; `rates` i `v_position`; `alerts` z kodami z
SPEC §8.

**Wymaga:** pkt 14 (progi i limity), pkt 24 (`TRANSIT_STALE`).

**Zasady w grze:** 8.

## Krok 10 — AML

**Cel:** obsługa procedur z SPEC §9.

**Zakres:** tabela `customers` (ADR-0004), powiązanie z operacjami KH i KZ, serie
transakcji w oknie czasowym, rejestr ponadprogowych i podejrzanych z eksportem,
weryfikacja list sankcyjnych, alert `THRESHOLD_AML`.

**Wymaga:** pkt 14 (próg ustawowy), pkt 15 (stanowisko MLRO).

**Zasady w grze:** 3, 5.

## Krok 11 — dokumenty PDF i eksport

**Cel:** KP, KW i dowody wewnętrzne jako PDF; eksport dla biura rachunkowego.

**Zakres:** jsPDF z osadzoną czcionką z polskimi znakami, `documents.pdf_path`,
eksport dla roli `accounting`.

**Wymaga:** pkt 18 (istniejący generator KP/KW), format eksportu uzgodniony z biurem
rachunkowym.

**Zasady w grze:** 3, 4.
