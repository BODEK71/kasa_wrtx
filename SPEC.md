# SPEC — księga skarbcowa trzech kas

Wersja 0.1 (robocza). Ten dokument jest źródłem prawdy dla implementacji.
Zmiana zachowania systemu zaczyna się od zmiany tego pliku, nie od zmiany kodu.

---

## 1. Problem i zakres

W jednej lokalizacji działają trzy kasy:

| Kasa | Co robi | Podmiot |
|---|---|---|
| **KW** — kantor walutowy | kupno/sprzedaż walut obcych za PLN, wpis NBP | KRTX |
| **KH** — kasa Hyllet | przyjęcie gotówki → wydanie e-pieniądza (PLNdt/EURdt) w modelu hyllet.cash | *do uzupełnienia w seed.yaml* |
| **KZ** — kasa wymian zewnętrznych | transakcje OTC krypto z kontrahentami | WRTX / KRTX |

Cel systemu: w każdej chwili wiadomo **gdzie jest wartość, w jakim aktywie i czyja jest**.
Produktem ubocznym, równie ważnym, jest ślad audytowy: kontrola NBP, GIIF, due diligence
i badanie ksiąg mają dostać spójny, niemodyfikowalny zapis.

Poza zakresem v1: obsługa klienta detalicznego w kasie walutowej (zostaje w Kantor-Logic),
pełna księgowość (system eksportuje dane do biura rachunkowego, nie zastępuje FK),
raportowanie GIIF (zostaje w Kantor-Logic dla KW; dla KH i KZ system tylko flaguje progi).

---

## 2. Zasady nienaruszalne

Te reguły mają pierwszeństwo przed wygodą, wydajnością i terminem.

1. **Stan nigdy nie jest wpisywany.** Saldo dowolnego konta to zawsze suma zapisów.
   Nie istnieje kolumna „saldo”, którą można nadpisać.
2. **Podwójny zapis per aktywo.** Każda operacja rozpisuje się na co najmniej dwa zapisy.
   Dla każdego aktywa w obrębie operacji suma kwot ze znakiem wynosi zero.
3. **Tylko dopisywanie.** `operations` i `postings` nie podlegają UPDATE ani DELETE.
   Korekta = storno, czyli nowa operacja odwracająca, z referencją do oryginału i powodem.
4. **Kwoty tylko dziesiętne.** `numeric` w bazie, string na granicy API, `decimal.js` w JS.
   Typ `float`/`number` dla kwoty jest błędem krytycznym, nie stylistycznym.
5. **Zapisu nie tworzy klient.** Front nie ma prawa INSERT na `operations`/`postings`.
   Jedyna droga to funkcje RPC w Postgresie.
6. **Każda operacja ma podmiot i tytuł prawny.** Ruch wartości między KRTX a WRTX to
   transakcja z dokumentem, nie przełożenie z szuflady do szuflady.
7. **Idempotencja importu.** Ponowny import tego samego pliku nie tworzy nowych zapisów.
8. **Każdy zapis ma autora i czas serwera.** `created_by`, `created_at = now()`, bez wyjątków.

---

## 3. Model danych

### 3.1 Słowniki

**`entities`** — podmioty prawne
`id, code, name, nip, krs, is_internal`
Przykłady: `KRTX`, `WRTX`, `STABILLON` (emitent), `HYLLET_NET` (sieć), kontrahenci OTC.
`is_internal = true` dla spółek własnych.

**`assets`** — aktywa
`id, code, kind, scale, chain, display_name, is_active`
- `kind`: `fiat` | `emoney` | `crypto`
- `scale`: liczba miejsc po przecinku (PLN 2, BTC 8, ETH 18, USDT 6)
- `chain`: dla krypto obowiązkowe (`tron`, `ethereum`, `bsc`, …)
- Klucz naturalny to `(code, chain)`. `USDT@tron` i `USDT@ethereum` to **dwa różne aktywa**.
  Zamiana jednego na drugie to operacja, nie to samo saldo.

**`locations`** — miejsca
`id, code, name, kind, entity_id, is_active`
- `kind`: `till` (kasa) | `vault` (sejf) | `bank` | `wallet` | `exchange` | `transit` | `external`
- Dla `wallet`: `address`, `chain`, `watch_only = true`
- `entity_id` mówi, czyje jest miejsce

**`accounts`** — konta księgowe
`id, location_id, asset_id, entity_id, kind`
Klucz unikalny: `(location_id, asset_id, entity_id)`.
- `kind`: `real` (gotówka, saldo portfela) | `receivable` (należność) | `payable` (zobowiązanie)
  | `pnl` (marża, opłaty, różnice kursowe) | `suspense` (manko/nadwyżka) | `equity` (bilans otwarcia)

Konta techniczne, które muszą istnieć:
- `TRANSIT` — wartość w drodze (gotówka do banku, transfer niepotwierdzony, przekazanie między kasami)
- `RECV/<kontrahent>` — rozrachunek OTC
- `SETTLE/STABILLON` — rozliczenie z emitentem e-pieniądza
- `SETTLE/NETWORK` — udział sieci hyllet.cash
- `INTERCO/<podmiot>` — rozrachunek między spółkami własnymi
- `FX_POSITION` — pozycja wymiany (druga strona kupna/sprzedaży waluty)
- `MARGIN`, `FEES`, `DIFF` — marża, opłaty, manko/nadwyżka
- `OPENING` — bilans otwarcia

### 3.2 Rdzeń

**`operations`**
```
id                uuid pk
op_type           text        -- kod z katalogu, sekcja 4
occurred_at       timestamptz -- moment zdarzenia
created_at        timestamptz default now()
created_by        uuid        -- użytkownik
entity_id         uuid        -- podmiot prowadzący operację
shift_id          uuid null   -- zmiana kasowa, jeśli dotyczy
counterparty_id   uuid null   -- kontrahent (entities)
customer_id       uuid null   -- klient detaliczny
legal_basis       text null   -- tytuł prawny przy ruchach między podmiotami
source            text        -- 'ui' | 'kantor_logic' | 'chain' | 'exchange' | 'hyllet' | 'bank'
external_id       text null   -- nr dowodu / hash / id zewnętrzne
reverses_id       uuid null   -- storno: wskazanie operacji odwracanej
reversal_reason   text null
note              text null
```
`unique (source, external_id) where external_id is not null` — to jest gwarancja idempotencji.

**`postings`**
```
id            bigserial pk
operation_id  uuid not null
account_id    uuid not null
amount        numeric(38,18) not null  -- ze znakiem, + przychód / − rozchód
seq           int not null
```
Ograniczenia:
- `amount <> 0`
- skala `amount` ≤ `assets.scale` konta (trigger)
- **constraint trigger DEFERRABLE INITIALLY DEFERRED**: dla każdej operacji i każdego aktywa
  `sum(amount) = 0`
- brak polityk RLS na UPDATE/DELETE + `FORCE ROW LEVEL SECURITY`

**`shifts`** — zmiany kasowe
`id, location_id, opened_at, opened_by, closed_at, closed_by, status`

**`shift_counts`** — ślepe liczenie
`id, shift_id, asset_id, denomination, qty, counted_at, counted_by`
Kasjer wpisuje policzone nominały **zanim** zobaczy stan systemowy.
Po zapisaniu liczenia system pokazuje różnicę i wymusza operację `ADJ-DIFF` z wyjaśnieniem.

**`documents`** — KP, KW, dowody wewnętrzne
`id, operation_id, doc_type, entity_id, number, issued_at, pdf_path, payload jsonb`
Numeracja ciągła per `(entity_id, doc_type, rok)`, nadawana w bazie, bez luk.

**`external_balances`** — salda z zewnątrz
`id, location_id, asset_id, balance, as_of, source, raw jsonb`
Nigdy nie wpływa na księgę. Służy wyłącznie do uzgodnienia i alertów.

**`import_runs`** — log importów
`id, source, file_name, file_hash, period_from, period_to, rows_total, rows_new, rows_skipped, rows_conflict, status, started_at, finished_at, created_by`

**`rates`** — kursy do wyceny
`id, asset_id, quote_asset_id, rate, as_of, source`

**`alerts`**
`id, code, severity, subject_ref, message, raised_at, ack_at, ack_by`

### 3.3 Widoki

- `v_account_balances` — saldo per konto (suma zapisów)
- `v_position` — miejsce × aktywo × podmiot, z wyceną w PLN wg `rates`
- `v_open_otc` — transakcje OTC z niezerowym rozrachunkiem
- `v_transit` — niezerowe salda `TRANSIT` (każde powyżej progu czasu to alert)
- `v_daily_kw` — dzień kasy walutowej z importu, do uzgodnienia z raportem Kantor-Logic

---

## 4. Katalog operacji

Każdy typ to osobna funkcja RPC. Argumenty są jawne, księgowanie jest w jednym miejscu.

### 4.1 Kantor walutowy (źródło: import, bez wpisu ręcznego)

| Kod | Zdarzenie | Zapisy |
|---|---|---|
| `KL-BUY` | kupno waluty od klienta | `+` KW/waluta, `−` KW/PLN, różnica → `FX_POSITION` |
| `KL-SELL` | sprzedaż waluty klientowi | `−` KW/waluta, `+` KW/PLN, różnica → `FX_POSITION` |
| `KL-CASHIN` | wpłata do kasy KW | `+` KW/aktywo, `−` `TRANSIT` |
| `KL-CASHOUT` | wypłata z kasy KW | `−` KW/aktywo, `+` `TRANSIT` |

`KL-CASHIN`/`KL-CASHOUT` **domykają** drugą stronę ruchu zapoczątkowanego w systemie
(sejf, inna kasa, bank). Dlatego kasa walutowa nigdy nie jest księgowana dwa razy.

### 4.2 Kasa Hyllet

| Kod | Zdarzenie | Zapisy |
|---|---|---|
| `EM-ISSUE` | klient wpłaca gotówkę, dostaje e-pieniądz | `+` KH/PLN (gotówka); `−` `SETTLE/STABILLON` o nominał; `+` `MARGIN` o marżę; `−` `SETTLE/NETWORK` o udział sieci |
| `EM-REDEEM` | klient przychodzi po gotówkę za e-pieniądz | odwrotnie |
| `EM-SETTLE` | przelew do/od emitenta | `+/−` bank, `−/+` `SETTLE/STABILLON` |
| `EM-NETFEE` | rozliczenie z siecią | `+/−` bank, `−/+` `SETTLE/NETWORK` |

**Uwaga modelowa:** po stronie gotówki nie powstaje krypto. Powstaje zobowiązanie wobec
emitenta i przychód z marży. Jeśli e-pieniądz jest prefinansowany (patrz `seed.yaml`),
`SETTLE/STABILLON` startuje z saldem dodatnim i `EM-ISSUE` je konsumuje — wtedy alert
„kończy się float” jest kluczowym sygnałem operacyjnym.

### 4.3 OTC

| Kod | Zdarzenie | Zapisy |
|---|---|---|
| `OTC-OPEN` | uzgodnienie warunków | brak zapisów, tylko nagłówek + kurs |
| `OTC-CASH` | noga gotówkowa | `+/−` KZ/waluta, `−/+` `RECV/<kontrahent>` |
| `OTC-CRYPTO` | noga krypto (hash obowiązkowy) | `+/−` portfel/aktywo, `−/+` `RECV/<kontrahent>` |
| `OTC-FEE` | opłata sieciowa | `−` portfel/aktywo, `+` `FEES` |

Transakcja jest zamknięta, gdy `RECV/<kontrahent>` dla tej transakcji jest zerowy we
wszystkich aktywach. Dopóki nie jest — widać dokładnie, kto komu i ile jest winien.
To jest odpowiedź na pytanie „gdzie w tym momencie jest wartość”.

### 4.4 Skarbiec

| Kod | Zdarzenie |
|---|---|
| `TR-MOVE` | przesunięcie w ramach jednego podmiotu (kasa ↔ sejf ↔ kasa) przez `TRANSIT` |
| `TR-INTERCO` | przesunięcie między spółkami własnymi: dokument po obu stronach + `INTERCO`, wymaga `legal_basis` |
| `TR-BANK` | bank ↔ kasa/sejf przez `TRANSIT` |
| `TR-WALLET` | transfer między własnymi portfelami/giełdą, z hashem i opłatą |
| `TR-TRADE` | wymiana na giełdzie: `−` aktywo A, `+` aktywo B, różnica → `FX_POSITION` |
| `SH-OPEN` / `SH-CLOSE` | otwarcie i zamknięcie zmiany |
| `ADJ-DIFF` | manko/nadwyżka z liczenia: `+/−` kasa, `−/+` `DIFF`, wymaga wyjaśnienia |
| `ADJ-STORNO` | odwrócenie operacji, wymaga `reverses_id` i `reversal_reason` |
| `ADJ-OPENING` | bilans otwarcia: `+` konto, `−` `OPENING` |

---

## 5. Import z Kantor-Logic

Kantor-Logic działa na Windows/.NET z bazą MS SQL i ma moduł eksportu do księgowości
(Symfonia, SAP, Rewizor, Fakir, Microsoft Dynamics NAV, Comarch) obejmujący transakcje
oraz operacje kasowe i bankowe. Nie ma API. Model importu:

1. **Wejście**: plik eksportu za dzień lub okres. Parser wykrywa format po nagłówku.
2. **Klucz**: numer dowodu kupna/sprzedaży. Jeśli eksport nie schodzi do poziomu dowodu,
   kluczem jest `(data, stanowisko, nr raportu, lp)`, a ziarno spada do raportu dziennego —
   to decyzja do podjęcia po obejrzeniu próbki, nie do zgadnięcia.
3. **Idempotencja**: `unique (source='kantor_logic', external_id)`. Ponowny import pomija
   istniejące wiersze i raportuje je jako `skipped`.
4. **Wykrywanie usunięć**: dowód obecny w poprzednim imporcie tego samego okresu, a nieobecny
   w bieżącym, generuje `ADJ-STORNO` oraz alert `KL_ROW_VANISHED` o wysokiej wadze.
   Równolegle: w Kantor-Logic odebrać kasjerom prawo usuwania (program pozwala ustawić
   uprawnienia osobno dla każdego kasjera).
5. **Uzgodnienie**: po imporcie system porównuje stany walut z raportem dziennym
   Kantor-Logic i saldo `TRANSIT` musi być zerowe. Niezgodność blokuje zamknięcie dnia.
6. **Aktualność**: stan kasy walutowej jest aktualny na moment ostatniego importu.
   Minimum to eksport przy każdym zamknięciu zmiany.

**Ścieżka do podglądu na żywo** (poza v1, do rozmowy z INFO-LOGIC):
agent czytający bazę MS SQL wyłącznie do odczytu, albo strumień danych po ich stronie.
Kantor-Logic Manager odświeża stany po każdej transakcji, więc dane już teraz wychodzą
z programu na bieżąco. Firma robi też oprogramowanie na zamówienie.

---

## 6. Integracje odczytowe

Wszystkie zasilają wyłącznie `external_balances`. Żadna nie tworzy zapisów w księdze.

| Źródło | Co pobiera | Uwaga |
|---|---|---|
| Blockchain | salda adresów watch-only | tylko adresy, nigdy klucze prywatne |
| Giełda | salda kont | klucz API wyłącznie read-only, bez uprawnień wypłaty |
| Hyllet | saldo e-pieniądza, historia wydań | do uzgodnienia z `SETTLE/STABILLON` |
| Bank | wyciąg | do uzgodnienia z `TRANSIT` i kontami bankowymi |

Różnica między saldem zewnętrznym a księgowym powyżej progu → alert `RECON_MISMATCH`.
Alert nigdy nie „poprawia” księgi automatycznie. Poprawia ją człowiek, operacją.

---

## 7. Role i dostęp

| Rola | Może |
|---|---|
| `cashier` | operacje własnej kasy i własnej zmiany, liczenie, podgląd własnych stanów |
| `manager` | wszystkie kasy w lokalizacji, przesunięcia, zatwierdzanie różnic, storno |
| `accounting` | odczyt wszystkiego, eksport, brak prawa księgowania |
| `mlro` | odczyt wszystkiego + kartoteka klientów + rejestr progów |
| `admin` | słowniki, użytkownicy, konfiguracja; **nie** ma prawa modyfikacji zapisów |

RLS na poziomie wiersza po `location_id` i `entity_id`. Kasjer nie widzi zysku ani stanów
innych kas. Storno wymaga roli `manager` i powodu.

Uwierzytelnienie: hasło + TOTP obowiązkowo dla `manager`, `accounting`, `mlro`, `admin`.
Kasjer na tablecie: sesja stanowiskowa + PIN kasjera przy każdej operacji.

---

## 8. Alerty

| Kod | Warunek |
|---|---|
| `TRANSIT_STALE` | niezerowy `TRANSIT` dłużej niż X godzin |
| `RECON_MISMATCH` | saldo zewnętrzne ≠ księgowe powyżej progu |
| `LIMIT_LOW` | zapas aktywa w kasie poniżej minimum |
| `LIMIT_HIGH` | gotówka w kasie powyżej limitu (ryzyko i ubezpieczenie) |
| `OTC_UNSETTLED` | otwarta noga OTC dłużej niż Y godzin |
| `FLOAT_LOW` | `SETTLE/STABILLON` poniżej progu |
| `COUNT_DIFF` | różnica z liczenia powyżej progu |
| `KL_ROW_VANISHED` | dowód zniknął między importami |
| `KL_IMPORT_MISSING` | brak importu za zamkniętą zmianę |
| `THRESHOLD_AML` | transakcja lub seria powyżej progu ustawowego |

---

## 9. AML

System nie zastępuje procedur, ale musi je obsłużyć:

- wspólna kartoteka klienta z identyfikacją, żeby wyłapać rozbicie kwoty między okienka;
  **zakres wspólnej kartoteki przy dwóch spółkach uzgadnia MLRO** — to decyzja prawna,
  nie techniczna, i nie jest podejmowana w kodzie
- liczenie serii powiązanych transakcji w oknie czasowym, per klient i per podmiot
- rejestr transakcji ponadprogowych i podejrzanych z eksportem
- weryfikacja list sankcyjnych na etapie zakładania klienta
- pełna, niemodyfikowalna historia: kto, kiedy, co i dlaczego

---

## 10. Stos i wybory techniczne

| Warstwa | Wybór | Powód |
|---|---|---|
| Baza | Supabase Postgres | `numeric`, transakcje, RLS, triggery — logika pieniędzy siedzi w bazie |
| Migracje | Supabase CLI, `supabase/migrations/` | schemat w repo, `supabase db reset` odtwarza stan |
| API | funkcje Postgres (RPC), `security definer` | klient nie ma prawa pisać do księgi |
| Front | Next.js App Router + TypeScript | znajomy stos, Vercel |
| Liczby w JS | `decimal.js`, wartości jako `string` | `number` gubi precyzję przy 8–18 miejscach |
| PDF | jsPDF z osadzoną czcionką z polskimi znakami | sprawdzone w generatorze KP/KW |
| Testy | pgTAP na logice księgowej | księga bez testów to kosztowna hipoteza |

Supabase JS zwraca `numeric` jako string i **tak ma zostać**. Rzutowanie na `number`
w jakimkolwiek miejscu ścieżki pieniądza jest błędem do odrzucenia na przeglądzie.

---

## 11. Otwarte pytania

Do uzupełnienia w `docs/seed.yaml` przed krokiem 2 planu:

1. Próbka eksportu z Kantor-Logic za jeden dzień (eksport księgowy + raport dzienny),
   dane klientów mogą być wyczyszczone — liczy się struktura.
2. Na której spółce jest kasa Hyllet.
3. Lista walut oraz tokenów razem z sieciami.
4. Czy e-pieniądz jest prefinansowany, czy rozliczany z emitentem po fakcie.
5. Progi alertów i limity kasowe.
6. Zakres wspólnej kartoteki klienta między KRTX a WRTX — stanowisko MLRO.
