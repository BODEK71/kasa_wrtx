# SPEC — księga skarbcowa trzech kas

Wersja 0.2 (robocza). Ten dokument jest źródłem prawdy dla implementacji.
Zmiana zachowania systemu zaczyna się od zmiany tego pliku, nie od zmiany kodu.

Historia zmian:
- 0.2 (2026-09-24) — rozstrzygnięte punkty blokujące z `docs/OPEN-QUESTIONS.md`,
  uzasadnienia w `docs/DECISIONS.md` (ADR-0001…ADR-0009).
- 0.1 — wersja wyjściowa.

---

## 1. Problem i zakres

W jednej lokalizacji działają trzy kasy:

| Kasa | Co robi | Podmiot |
|---|---|---|
| **KW** — kantor walutowy | kupno/sprzedaż walut obcych za PLN, wpis NBP | KRTX |
| **KH** — kasa Hyllet | przyjęcie gotówki → wydanie e-pieniądza (PLNdt/EURdt) w modelu hyllet.cash | *do uzupełnienia w seed.yaml* |
| **KZ** — kasa wymian zewnętrznych | transakcje OTC krypto z kontrahentami | WRTX (ADR-0006) |

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
2. **Podwójny zapis per spółka i aktywo.** Każda operacja rozpisuje się na co najmniej
   dwa zapisy. Dla każdej spółki (czyje księgi, `accounts.entity_id`) i każdego aktywa
   w obrębie operacji suma kwot ze znakiem wynosi zero. Zdarzenia bez ruchu wartości
   (uzgodnienie warunków OTC, otwarcie/zamknięcie zmiany) nie są operacjami
   (ADR-0001, ADR-0003).
3. **Tylko dopisywanie.** `operations` i `postings` nie podlegają UPDATE ani DELETE.
   Korekta = storno, czyli nowa operacja odwracająca, z referencją do oryginału i powodem.
4. **Kwoty tylko dziesiętne.** `numeric` w bazie, string na granicy API, `decimal.js` w JS.
   Typ `float`/`number` dla kwoty jest błędem krytycznym, nie stylistycznym.
5. **Zapisu nie tworzy klient.** Front nie ma prawa INSERT na `operations`/`postings`.
   Jedyna droga to funkcje RPC w Postgresie.
6. **Każda operacja ma podmiot; ruch między spółkami ma tytuł prawny.** Operacja, której
   zapisy dotykają ksiąg więcej niż jednej spółki własnej, jest możliwa wyłącznie jako
   `TR-INTERCO` z niepustym `legal_basis`. Ruch wartości między KRTX a WRTX to
   transakcja z dokumentem, nie przełożenie z szuflady do szuflady (ADR-0003).
7. **Idempotencja importu.** Ponowny import tego samego pliku nie tworzy nowych zapisów.
8. **Każdy zapis ma autora i czas serwera.** `created_by`, `created_at = now()`, bez wyjątków.

---

## 3. Model danych

### 3.1 Słowniki

**`entities`** — podmioty prawne, z którymi prowadzimy rozrachunki
`id, code, name, nip, krs, is_internal, role`
Przykłady: `KRTX`, `WRTX`, `STABILLON` (emitent), `HYLLET_NET` (sieć), kontrahenci OTC.
`is_internal = true` dla spółek własnych.
- `role`: `emoney_issuer` | `network` | `otc_counterparty` | `null` (spółki własne) (ADR-0009)
- Klienci detaliczni **nie** są w `entities` — patrz `customers` (ADR-0004).

**`assets`** — aktywa
`id, code, kind, scale, chain, issuer_entity_id, underlying_asset_id, display_name, is_active`
- `kind`: `fiat` | `emoney` | `crypto`
- `scale`: liczba miejsc po przecinku (PLN 2, BTC 8, ETH 18, USDT 6)
- `chain`: dla krypto obowiązkowe (`tron`, `ethereum`, `bsc`, …)
- `issuer_entity_id`, `underlying_asset_id`: obowiązkowe wtedy i tylko wtedy, gdy
  `kind = emoney` — emitent (`role = emoney_issuer`) i waluta fiat, z którą e-pieniądz
  jest 1:1 (PLNdt → PLN, EURdt → EUR) (ADR-0009)
- Klucz naturalny to `(code, chain)`. `USDT@tron` i `USDT@ethereum` to **dwa różne aktywa**.
  Zamiana jednego na drugie to operacja, nie to samo saldo.
- Aktywa `emoney` nie mają kont księgowych — e-pieniądz nie jest naszym aktywem, występuje
  tylko w `external_balances` i jako argument operacji `EM-*` (ADR-0008).

**`locations`** — miejsca
`id, code, name, kind, entity_id, is_active`
- `kind`: `till` (kasa) | `vault` (sejf) | `bank` | `wallet` | `exchange`
- `entity_id` (NOT NULL) mówi, czyje jest miejsce. Jedno miejsce należy do jednej spółki.
  Fizycznie wspólny schowek dwóch spółek to dwie lokalizacje z osobnym liczeniem (ADR-0006).
- Dla `wallet`: `watch_only = true`, adresy w `location_addresses`

**`location_addresses`** — adresy portfeli watch-only (ADR-0005)
`id, location_id, chain, address`
Klucz unikalny: `(chain, address)`. Jeden portfel może mieć adresy w wielu sieciach.
Konto `holding` w lokalizacji `wallet` na aktywie z siecią X wymaga adresu tej lokalizacji
w sieci X.

**`accounts`** — konta księgowe (ADR-0007)
`id, entity_id, asset_id, kind, location_id, counterparty_id`
Klucz unikalny: `unique nulls not distinct (entity_id, asset_id, kind, location_id, counterparty_id)`.
- `entity_id` — czyje księgi; zawsze spółka własna (`is_internal = true`)
- `location_id` — obowiązkowe dla `holding`, zakazane dla pozostałych rodzajów;
  dla `holding` `entity_id = locations.entity_id`
- `counterparty_id` — obowiązkowe dla `recv`, `settle`, `interco`, zakazane dla pozostałych

| `kind` | Zapis w tym dokumencie | `location_id` | `counterparty_id` | Klasa raportowa |
|---|---|---|---|---|
| `holding` | `KW/PLN`, `portfel/USDT`, `bank/PLN` | wymagane | — | środki |
| `transit` | `TRANSIT` — wartość w drodze (gotówka do banku, transfer niepotwierdzony, przekazanie między kasami) | — | — | środki w drodze |
| `recv` | `RECV/<kontrahent>` — rozrachunek OTC | — | `role = otc_counterparty` | rozrachunek |
| `settle` | `SETTLE/STABILLON`, `SETTLE/NETWORK` — rozliczenie z emitentem / udział sieci hyllet.cash | — | `role = emoney_issuer` / `network` | rozrachunek |
| `interco` | `INTERCO/<podmiot>` — rozrachunek między spółkami własnymi | — | spółka własna ≠ `entity_id` | rozrachunek |
| `fx_position` | `FX_POSITION` — pozycja wymiany | — | — | pozycja walutowa |
| `margin` | `MARGIN` | — | — | wynik |
| `fees` | `FEES` | — | — | wynik |
| `diff` | `DIFF` — manko/nadwyżka | — | — | rozliczenie różnic |
| `opening` | `OPENING` — bilans otwarcia | — | — | kapitał |

Rozrachunek ma jedno konto per (spółka, aktywo, kontrahent); znak salda mówi, kto komu jest
winien — nie ma osobnych kont należności i zobowiązań.

Konwencja znaków: `+` debet (środki, należności, koszty), `−` kredyt (zobowiązania,
przychody). Przychód na `margin` ma więc znak `−`.

Konwencja `FX_POSITION`: w wymianie aktywa A na aktywo B każde aktywo bilansuje się osobno —
druga noga w każdym aktywie trafia na `fx_position` tego aktywa (ADR-0008).

### 3.2 Rdzeń

**`operations`**
```
id                uuid pk
op_type           text        -- kod z katalogu, sekcja 4
occurred_at       timestamptz -- moment zdarzenia
created_at        timestamptz default now()
created_by        uuid        -- użytkownik
entity_id         uuid        -- spółka prowadząca operację
shift_id          uuid null   -- zmiana kasowa, jeśli dotyczy
counterparty_id   uuid null   -- kontrahent (entities)
customer_id       uuid null   -- klient detaliczny (customers)
deal_id           uuid null   -- transakcja OTC (otc_deals); obowiązkowe dla OTC-CASH, OTC-CRYPTO
legal_basis       text null   -- tytuł prawny; obowiązkowy dla TR-INTERCO
source            text        -- 'ui' | 'kantor_logic'
external_id       text null   -- nr dowodu / id zewnętrzne z importu
tx_hash           text null   -- hash transakcji on-chain; obowiązkowy dla OTC-CRYPTO, TR-WALLET
reverses_id       uuid null   -- storno: wskazanie operacji odwracanej
reversal_reason   text null
note              text null
```
`unique (source, external_id) where external_id is not null` — to jest gwarancja idempotencji.

`source` mówi, którym kanałem operacja weszła do księgi: człowiek przez RPC (`ui`) albo
import Kantor-Logic. Blockchain, giełda, Hyllet i bank nie są źródłami operacji — zasilają
wyłącznie `external_balances` (sekcja 6, ADR-0002). `tx_hash` jest dowodem wpisanym przez
człowieka, nie kluczem idempotencji: jedna transakcja on-chain może stać za kilkoma
operacjami (noga i opłata), więc nie jest unikalny.

**`postings`**
```
id            bigserial pk
operation_id  uuid not null
account_id    uuid not null
amount        numeric(38,18) not null  -- ze znakiem: + debet (wpływ środków) / − kredyt (wypływ, zobowiązanie, przychód)
seq           int not null
```
Ograniczenia:
- `amount <> 0`
- skala `amount` ≤ `assets.scale` konta (trigger)
- **constraint trigger DEFERRABLE INITIALLY DEFERRED** na `postings`: dla każdej operacji,
  każdej spółki (`accounts.entity_id`) i każdego aktywa `sum(amount) = 0`
- **constraint trigger DEFERRABLE INITIALLY DEFERRED** na `operations`: operacja ma co
  najmniej dwa zapisy; jeśli zapisy dotykają ksiąg więcej niż jednej spółki, to
  `op_type = 'TR-INTERCO'` i `legal_basis` niepusty; w pozostałych operacjach
  `accounts.entity_id = operations.entity_id` dla każdego zapisu (ADR-0003)
- brak polityk RLS na UPDATE/DELETE + `FORCE ROW LEVEL SECURITY`

**`otc_deals`** — uzgodnione warunki transakcji OTC (ADR-0001)
`id, entity_id, counterparty_id, agreed_at, base_asset_id, base_amount, quote_asset_id, quote_amount, note, created_by, created_at`
Tylko dopisywanie, jak `operations`. Kurs = `quote_amount / base_amount`. Nie ma zapisów
księgowych — wartość rusza się dopiero w operacjach `OTC-CASH` / `OTC-CRYPTO` z `deal_id`.
Stan „otwarta / zamknięta” nie jest polem: wynika z salda `recv` dla operacji z tym `deal_id`.

**`customers`** — klienci detaliczni (ADR-0004)
Osoby i firmy identyfikowane na potrzeby AML, bez kont księgowych. `operations.customer_id`
wskazuje tutaj. Zakres pól identyfikacyjnych i widoczność między spółkami ustala MLRO
(sekcja 9) — do tego czasu tabela nie wchodzi do migracji rdzenia.

**`shifts`** — zmiany kasowe
`id, location_id, opened_at, opened_by, closed_at, closed_by, status`
Otwarcie i zamknięcie zmiany to funkcje RPC (`shift_open`, `shift_close`), nie operacje
księgowe. Zamknięcie jest jednorazowe: `closed_at`/`closed_by` przechodzą z `null` na
wartość i nie zmieniają się później (trigger) (ADR-0001).

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
- `v_open_otc` — transakcje OTC (`otc_deals`) z niezerowym saldem `recv` liczonym po
  operacjach z danym `deal_id`
- `v_transit` — niezerowe salda `TRANSIT` (każde powyżej progu czasu to alert)
- `v_daily_kw` — dzień kasy walutowej z importu, do uzgodnienia z raportem Kantor-Logic

---

## 4. Katalog operacji

Każdy typ to osobna funkcja RPC. Argumenty są jawne, księgowanie jest w jednym miejscu.

### 4.1 Kantor walutowy (źródło: import, bez wpisu ręcznego)

| Kod | Zdarzenie | Zapisy |
|---|---|---|
| `KL-BUY` | kupno waluty od klienta | waluta: `+` KW/waluta, `−` `FX_POSITION`/waluta; PLN: `−` KW/PLN, `+` `FX_POSITION`/PLN |
| `KL-SELL` | sprzedaż waluty klientowi | waluta: `−` KW/waluta, `+` `FX_POSITION`/waluta; PLN: `+` KW/PLN, `−` `FX_POSITION`/PLN |
| `KL-CASHIN` | wpłata do kasy KW | `+` KW/aktywo, `−` `TRANSIT` |
| `KL-CASHOUT` | wypłata z kasy KW | `−` KW/aktywo, `+` `TRANSIT` |

`KL-CASHIN`/`KL-CASHOUT` **domykają** drugą stronę ruchu zapoczątkowanego w systemie
(sejf, inna kasa, bank). Dlatego kasa walutowa nigdy nie jest księgowana dwa razy.

### 4.2 Kasa Hyllet

| Kod | Zdarzenie | Zapisy |
|---|---|---|
| `EM-ISSUE` | klient wpłaca gotówkę, dostaje e-pieniądz | poniżej |
| `EM-REDEEM` | klient oddaje e-pieniądz, dostaje gotówkę | poniżej |
| `EM-SETTLE` | przelew do/od emitenta | `+/−` bank, `−/+` `SETTLE/STABILLON` |
| `EM-NETFEE` | rozliczenie z siecią | `+/−` bank, `−/+` `SETTLE/NETWORK` |

Oznaczenia (ADR-0008): `C` — waluta gotówki w kasie; `E` — waluta bazowa wydanego
e-pieniądza (`assets.underlying_asset_id`, np. EURdt → EUR); `G` — gotówka od/do klienta
w `C`; `N` — nominał e-pieniądza w `E`; `S` — udział sieci w `C`; `M` — marża w `C`,
zawsze wyliczana jako reszta. Wszystkie zapisy są w walutach fiat — e-pieniądz nie ma kont.

**Ta sama waluta (`C = E`)**, `M = G − N − S` przy wydaniu, `M = N − G − S` przy wykupie:

| Konto | `EM-ISSUE` | `EM-REDEEM` |
|---|---|---|
| KH/C | `+G` | `−G` |
| `SETTLE/STABILLON`/C | `−N` | `+N` |
| `SETTLE/NETWORK`/C | `−S` | `−S` |
| `MARGIN`/C | `−M` | `−M` |

**Przewalutowanie (`C ≠ E`)** — RPC dostaje jawnie kurs `r` (`E` → `C`);
`X = round(N · r, scale(C))`; `M = G − S − X` przy wydaniu, `M = X − G − S` przy wykupie.
Różnica zaokrąglenia trafia w `M`, więc suma zawsze wynosi zero:

| Konto | `EM-ISSUE` | `EM-REDEEM` |
|---|---|---|
| KH/C | `+G` | `−G` |
| `SETTLE/NETWORK`/C | `−S` | `−S` |
| `MARGIN`/C | `−M` | `−M` |
| `FX_POSITION`/C | `−X` | `+X` |
| `FX_POSITION`/E | `+N` | `−N` |
| `SETTLE/STABILLON`/E | `−N` | `+N` |

Reguły: `M = 0` — brak zapisu na `MARGIN`; `M < 0` — RPC odrzuca operację (zmiana tej
reguły wymaga ADR); `S = 0` — brak zapisu na `SETTLE/NETWORK`.

**Uwaga modelowa:** po stronie gotówki nie powstaje krypto ani e-pieniądz na naszym koncie.
Powstaje rozrachunek z emitentem i przychód z marży. Jeśli e-pieniądz jest prefinansowany
(patrz `seed.yaml`), `SETTLE/STABILLON` startuje z saldem dodatnim i `EM-ISSUE` je
konsumuje — wtedy alert „kończy się float” jest kluczowym sygnałem operacyjnym. Te same
zapisy obsługują model rozliczany po fakcie — wtedy saldo po prostu schodzi poniżej zera.

### 4.3 OTC

| Kod | Zdarzenie | Zapisy |
|---|---|---|
| `OTC-CASH` | noga gotówkowa (`deal_id` obowiązkowe) | `+/−` KZ/waluta, `−/+` `RECV/<kontrahent>` |
| `OTC-CRYPTO` | noga krypto (`deal_id` i `tx_hash` obowiązkowe) | `+/−` portfel/aktywo, `−/+` `RECV/<kontrahent>` |
| `OTC-FEE` | opłata sieciowa | `−` portfel/aktywo, `+` `FEES` |

Uzgodnienie warunków nie jest operacją — to wpis w `otc_deals` przez RPC `otc_deal_open`,
bez zapisów księgowych (ADR-0001).

Transakcja jest zamknięta, gdy suma zapisów na `RECV/<kontrahent>` z operacji o jej
`deal_id` jest zerowa we wszystkich aktywach. Dopóki nie jest — widać dokładnie, kto komu
i ile jest winien. To jest odpowiedź na pytanie „gdzie w tym momencie jest wartość”.

### 4.4 Skarbiec

| Kod | Zdarzenie |
|---|---|
| `TR-MOVE` | przesunięcie w ramach jednego podmiotu (kasa ↔ sejf ↔ kasa) przez `TRANSIT` |
| `TR-INTERCO` | przesunięcie między spółkami własnymi: dokument po obu stronach + `INTERCO`, wymaga `legal_basis`; jedyna operacja dotykająca ksiąg dwóch spółek, każda z nich bilansuje się osobno |
| `TR-BANK` | bank ↔ kasa/sejf przez `TRANSIT` |
| `TR-WALLET` | transfer między własnymi portfelami/giełdą, z `tx_hash` i opłatą |
| `TR-TRADE` | wymiana na giełdzie: A: `−` giełda/A, `+` `FX_POSITION`/A; B: `+` giełda/B, `−` `FX_POSITION`/B |
| `ADJ-DIFF` | manko/nadwyżka z liczenia: `+/−` kasa, `−/+` `DIFF`, wymaga wyjaśnienia |
| `ADJ-STORNO` | odwrócenie operacji, wymaga `reverses_id` i `reversal_reason` |
| `ADJ-OPENING` | bilans otwarcia: `+` konto, `−` `OPENING` |

Otwarcie i zamknięcie zmiany nie są operacjami księgowymi — to RPC `shift_open` /
`shift_close` na tabeli `shifts` (ADR-0001).

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

Do uzupełnienia w `docs/seed.yaml` — każde przed krokiem, który go wymaga
(pozycja „Wymaga” w `docs/PLAN.md`):

1. Próbka eksportu z Kantor-Logic za jeden dzień (eksport księgowy + raport dzienny),
   dane klientów mogą być wyczyszczone — liczy się struktura.
2. Na której spółce jest kasa Hyllet.
3. Lista walut oraz tokenów razem z sieciami.
4. Czy e-pieniądz jest prefinansowany, czy rozliczany z emitentem po fakcie.
5. Progi alertów i limity kasowe.
6. Zakres wspólnej kartoteki klienta między KRTX a WRTX — stanowisko MLRO.

Pełna, bieżąca lista otwartych i rozstrzygniętych pytań: `docs/OPEN-QUESTIONS.md`.
