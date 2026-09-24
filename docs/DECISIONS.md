# Decyzje architektoniczne (ADR)

Log decyzji, które trwale kształtują system — zwłaszcza te dotykające zasad
nienaruszalnych z `CLAUDE.md` (model księgowy, schemat, bezpieczeństwo danych).
Nowy wpis przy każdej decyzji, która zmienia zachowanie systemu lub rozstrzyga
punkt z `docs/OPEN-QUESTIONS.md`. Nie edytuj istniejących wpisów wstecz — decyzja
odwrócona dostaje nowy wpis ze statusem `zastąpiona przez ADR-000X`.

---

## Szablon

```markdown
## ADR-000X: <tytuł decyzji>

- **Data:** RRRR-MM-DD
- **Status:** proponowana / przyjęta / zastąpiona przez ADR-000Y / odrzucona

### Kontekst

<Jaki problem lub pytanie wymagało rozstrzygnięcia. Jakie opcje rozważano.>

### Decyzja

<Co dokładnie zostało postanowione.>

### Konsekwencje

<Co to oznacza dla schematu, kodu, procesu. Jakie ograniczenia lub dług
wprowadza. Co trzeba zrobić w innych miejscach w efekcie tej decyzji.>
```

---

## ADR-0001: Zdarzenia bez ruchu wartości nie są operacjami

- **Data:** 2026-09-24
- **Status:** przyjęta
- **Rozstrzyga:** OPEN-QUESTIONS pkt 1

### Kontekst

SPEC 0.1 §2 zasada 2 wymaga, by każda operacja miała co najmniej dwa zapisy, a
jednocześnie katalog zawierał `OTC-OPEN` („brak zapisów, tylko nagłówek + kurs”) oraz
`SH-OPEN`/`SH-CLOSE`, które niczego nie księgują. Jeśli inwariant ma być twardy w bazie,
te operacje nie mogą istnieć w `operations`. Dodatkowo §4.3 liczy rozrachunek „dla tej
transakcji”, a konto `RECV/<kontrahent>` jest per kontrahent, nie per transakcja — nic
nie wiązało nóg OTC z uzgodnieniem.

### Decyzja

- Inwariant „≥2 zapisy” jest twardy: odroczony constraint trigger na `operations`.
- Uzgodnienie OTC to wiersz w nowej tabeli `otc_deals` (`id, entity_id, counterparty_id,
  agreed_at, base_asset_id, base_amount, quote_asset_id, quote_amount, note, created_by,
  created_at`), tylko do dopisywania, tworzony przez RPC `otc_deal_open`. Kurs wynika z
  `quote_amount / base_amount` — nie jest przechowywany osobno.
- `OTC-CASH` i `OTC-CRYPTO` mają obowiązkowe `operations.deal_id`. Saldo transakcji to
  suma zapisów na koncie `recv` z operacji o danym `deal_id`; `v_open_otc` grupuje po
  `deal_id`. Status „otwarta/zamknięta” nie jest polem.
- Otwarcie/zamknięcie zmiany to RPC `shift_open`/`shift_close` na `shifts`. Zamknięcie
  jest jednorazowym przejściem `closed_at`/`closed_by` z `null` na wartość (trigger).

### Konsekwencje

- Kody `OTC-OPEN`, `SH-OPEN`, `SH-CLOSE` znikają z katalogu operacji.
- `otc_deals` podlega tej samej regule tylko-dopisywania co księga; zmiana warunków to
  nowa transakcja, nie edycja.
- `shifts` jest jedyną tabelą rdzenia z kontrolowanym UPDATE (jedno pole, raz).

---

## ADR-0002: `operations.source` tylko dla kanałów, które mogą księgować

- **Data:** 2026-09-24
- **Status:** przyjęta
- **Rozstrzyga:** OPEN-QUESTIONS pkt 2

### Kontekst

CLAUDE.md zasada 8 i SPEC §6 mówią, że blockchain, giełda, bank i Hyllet nigdy nie
tworzą zapisów w księdze. SPEC 0.1 §3.2 dopuszczał jednak `operations.source` =
`'chain' | 'exchange' | 'hyllet' | 'bank'`. Hash transakcji on-chain lądował w
`external_id`, który jest objęty `unique (source, external_id)` — przez to noga
`OTC-CRYPTO` i `OTC-FEE` z tej samej transakcji on-chain nie mogłyby współistnieć.

### Decyzja

- `operations.source` ∈ {`ui`, `kantor_logic`}. Pozostałe wartości zostają wyłącznie w
  `external_balances.source`.
- Nowa kolumna `operations.tx_hash` (bez unique) — dowód wpisany przez człowieka,
  obowiązkowy dla `OTC-CRYPTO` i `TR-WALLET`.
- `external_id` + `unique (source, external_id)` służy wyłącznie idempotencji importu.

### Konsekwencje

- Operacja krypto zawsze wchodzi przez człowieka (`ui`), z hashem jako dowodem.
  Integracja blockchain może później podpowiadać hash lub alarmować o rozbieżności,
  ale nie tworzy operacji.
- Brak unikalności `tx_hash` oznacza, że dwukrotne wpisanie tego samego transferu nie
  jest blokowane przez bazę — wykryje je uzgodnienie z `external_balances`.

---

## ADR-0003: Bilans per spółka i `legal_basis` tylko przy ruchu między spółkami

- **Data:** 2026-09-24
- **Status:** przyjęta (właściciel projektu bez preferencji — przyjęto wariant
  rekomendowany)
- **Rozstrzyga:** OPEN-QUESTIONS pkt 3

### Kontekst

SPEC 0.1 §2 zasada 6: „każda operacja ma podmiot i tytuł prawny”. Schemat ma
`legal_basis text null`, a CLAUDE.md zasada 6 wymaga go tylko przy ruchach między
podmiotami. Do tego bilans per (operacja, aktywo) pozwalał zaksięgować `−KW/PLN` (KRTX)
i `+KZ/PLN` (WRTX) jako zwykły `TR-MOVE` — suma zero, żadnego rozrachunku między
spółkami, żadnego tytułu prawnego.

### Decyzja

- `accounts.entity_id` zawsze wskazuje spółkę własną — „czyje to księgi”.
- Constraint trigger bilansu działa per **(operacja, spółka, aktywo)**.
- Operacja, której zapisy dotykają ksiąg więcej niż jednej spółki, musi mieć
  `op_type = 'TR-INTERCO'` i niepusty `legal_basis`. W każdej innej operacji wszystkie
  konta mają `entity_id = operations.entity_id`.
- Przy operacjach jednej spółki `legal_basis` jest opcjonalny.

### Konsekwencje

- `TR-INTERCO` musi przejść przez konta `interco` po obu stronach, bo inaczej żadna ze
  spółek się nie zbilansuje — „przełożenie z szuflady do szuflady” jest niemożliwe
  konstrukcyjnie, nie tylko regulaminowo.
- Zmienia brzmienie zasady 2 i 6 w SPEC §2 i w CLAUDE.md (obie zaostrzone).

---

## ADR-0004: Klienci detaliczni w osobnej tabeli `customers`

- **Data:** 2026-09-24
- **Status:** przyjęta
- **Rozstrzyga:** OPEN-QUESTIONS pkt 4

### Kontekst

`operations.customer_id` i SPEC §9 (AML) odwoływały się do klienta detalicznego, którego
nie było w modelu. Nie było też jasne, czy klienci mają żyć w `entities`.

### Decyzja

- `entities` — strony, z którymi prowadzimy rozrachunki (mają konta): spółki własne,
  emitent, sieć, kontrahenci OTC.
- `customers` — klienci identyfikowani na potrzeby AML, bez kont księgowych.
  `operations.customer_id` → `customers(id)`.
- Zakres danych identyfikacyjnych i widoczność między spółkami ustala MLRO
  (OPEN-QUESTIONS pkt 15).

### Konsekwencje

- Tabela `customers` powstaje w osobnej migracji po stanowisku MLRO; migracja rdzenia
  (słowniki, konta, operacje, zapisy) na nią nie czeka.
- Kontrahent OTC rozliczany cyklicznie jest w `entities`; jednorazowy klient kasy — w
  `customers`.

---

## ADR-0005: Adresy portfeli w `location_addresses`

- **Data:** 2026-09-24
- **Status:** przyjęta
- **Rozstrzyga:** OPEN-QUESTIONS pkt 5

### Kontekst

SPEC 0.1 dawał lokalizacji `wallet` pojedyncze `address, chain`, a `seed.yaml` opisuje
`WALLET_HOT` z adresami w dwóch sieciach.

### Decyzja

- Nowa tabela `location_addresses (id, location_id, chain, address)`,
  `unique (chain, address)`. Pola `address`, `chain` znikają z `locations`.
- Konto `holding` w lokalizacji `wallet` na aktywie z siecią X wymaga adresu tej
  lokalizacji w sieci X.

### Konsekwencje

- Kształt `seed.yaml` (lista `addresses`) zostaje bez zmian.
- Integracja blockchain mapuje adres → lokalizację przez tę tabelę.

---

## ADR-0006: Jedno miejsce należy do jednej spółki; KZ = WRTX

- **Data:** 2026-09-24
- **Status:** przyjęta
- **Rozstrzyga:** OPEN-QUESTIONS pkt 6

### Kontekst

SPEC 0.1 i `seed.yaml` dopuszczały kasę KZ „KRTX czy WRTX, albo obie”, a `locations`
ma jedno `entity_id`. Ślepe liczenie jednej szuflady nie przypisze różnicy do spółki.
Właściciel projektu: KZ pracuje wyłącznie na rachunek WRTX.

### Decyzja

- `locations.entity_id` NOT NULL; konto `holding` ma `entity_id = locations.entity_id`.
- KZ należy do WRTX.
- Fizycznie wspólny schowek dwóch spółek zapisuje się jako dwie lokalizacje z osobnym
  liczeniem.

### Konsekwencje

- Przynależność SEJF, WALLET_HOT i EXCHANGE_OKX (w `seed.yaml` nadal `TODO`) trzeba
  podać jako jedną spółkę albo rozbić na dwie lokalizacje.

---

## ADR-0007: Tożsamość kont i konta techniczne

- **Data:** 2026-09-24
- **Status:** przyjęta
- **Rozstrzyga:** OPEN-QUESTIONS pkt 7

### Kontekst

SPEC 0.1 wymagał `accounts.location_id` i klucza `(location_id, asset_id, entity_id)`,
a jednocześnie wymieniał konta techniczne bez fizycznego miejsca (`MARGIN`, `OPENING`,
`RECV/<kontrahent>`, …). Klucz nie rozróżniał wielu kontrahentów ani spółek, a
`locations.kind` zawierał `transit`/`external` jako namiastkę.

### Decyzja

- `accounts = id, entity_id, asset_id, kind, location_id null, counterparty_id null`,
  `unique nulls not distinct (entity_id, asset_id, kind, location_id, counterparty_id)`.
- `kind` ∈ `holding`, `transit`, `recv`, `settle`, `interco`, `fx_position`, `margin`,
  `fees`, `diff`, `opening`.
- `location_id` obowiązkowe tylko dla `holding`, zakazane dla reszty.
- `counterparty_id` obowiązkowe dla `recv` (rola `otc_counterparty`), `settle`
  (`emoney_issuer` / `network`), `interco` (spółka własna ≠ `entity_id`); zakazane dla
  reszty.
- Rozrachunek to jedno konto per (spółka, aktywo, kontrahent); znak salda mówi, kto jest
  winien. Dawny podział real/receivable/payable/pnl/suspense/equity zostaje w SPEC
  wyłącznie jako klasyfikacja raportowa.
- `locations.kind` traci `transit` i `external` — to już nie są miejsca.

### Konsekwencje

- RLS po `location_id` sam ukrywa przed kasjerem konta wynikowe (§7), bo nie mają
  lokalizacji.
- Zapis `SETTLE/STABILLON` w SPEC oznacza `kind = settle` + `counterparty = STABILLON`.
- Wymaga Postgresa ≥ 15 (`nulls not distinct`) — Supabase to spełnia.

---

## ADR-0008: Formuły `EM-*` i konwencja `FX_POSITION`

- **Data:** 2026-09-24
- **Status:** przyjęta
- **Rozstrzyga:** OPEN-QUESTIONS pkt 8

### Kontekst

SPEC 0.1 §4.2 wymieniał cztery zapisy `EM-ISSUE`, ale nie podawał aktywów ani
arytmetyki; w zapisie `+ MARGIN` suma się nie zeruje, a „`EM-REDEEM` odwrotnie”
zamieniłby marżę w koszt. Podobnie „różnica → `FX_POSITION`” w §4.1 nie bilansuje się
per aktywo. Właściciel projektu: marża netto (`G = N + S + M`), zapisy w walucie
gotówki; klient może dostać e-pieniądz w innej walucie niż wpłacona gotówka.

### Decyzja

Oznaczenia: `C` — waluta gotówki, `E` — waluta bazowa e-pieniądza
(`assets.underlying_asset_id`), `G` — gotówka, `N` — nominał e-pieniądza, `S` — udział
sieci (w `C`), `M` — marża (w `C`, zawsze reszta). Konwencja: `+` debet, `−` kredyt.

- `C = E`: ISSUE `KH/C +G`, `SETTLE/STABILLON −N`, `SETTLE/NETWORK −S`, `MARGIN −M`,
  `M = G − N − S`. REDEEM `KH/C −G`, `SETTLE/STABILLON +N`, `SETTLE/NETWORK −S`,
  `MARGIN −M`, `M = N − G − S`.
- `C ≠ E`: RPC dostaje jawny kurs `r`, `X = round(N · r, scale(C))`.
  ISSUE — w `C`: `KH +G`, `SETTLE/NETWORK −S`, `MARGIN −M`, `FX_POSITION −X`; w `E`:
  `FX_POSITION +N`, `SETTLE/STABILLON −N`; `M = G − S − X`.
  REDEEM — w `C`: `KH −G`, `SETTLE/NETWORK −S`, `MARGIN −M`, `FX_POSITION +X`; w `E`:
  `FX_POSITION −N`, `SETTLE/STABILLON +N`; `M = X − G − S`.
- `M = 0` → brak zapisu `MARGIN`; `S = 0` → brak zapisu `SETTLE/NETWORK`;
  `M < 0` → RPC odrzuca operację.
- Aktywa `emoney` nie mają kont.
- `FX_POSITION` ogólnie: w każdej wymianie A↔B druga noga w każdym aktywie idzie na
  `fx_position` tego aktywa (KL-BUY, KL-SELL, TR-TRADE, EM-* z przewalutowaniem).

Przykłady (suma per aktywo = 0):

| Przypadek | Zapisy | Suma |
|---|---|---|
| ISSUE PLN→PLNdt, G=1000, N=970, S=10 | KH +1000, STABILLON −970, NETWORK −10, MARGIN −20 | PLN 0 |
| REDEEM PLNdt→PLN, N=1000, G=970, S=10 | KH −970, STABILLON +1000, NETWORK −10, MARGIN −20 | PLN 0 |
| ISSUE PLN→EURdt, G=4300, N=1000, r=4,25, S=10 | PLN: KH +4300, NETWORK −10, MARGIN −40, FX −4250; EUR: FX +1000, STABILLON −1000 | PLN 0, EUR 0 |
| REDEEM EURdt→PLN, N=1000, G=4200, r=4,25, S=10 | PLN: KH −4200, NETWORK −10, MARGIN −40, FX +4250; EUR: FX −1000, STABILLON +1000 | PLN 0, EUR 0 |

### Konsekwencje

- Prefinansowanie i rozliczenie po fakcie używają tych samych zapisów — różni je tylko
  saldo startowe `SETTLE/STABILLON`.
- Różnica zaokrąglenia kursu zawsze trafia w marżę, więc bilans nie wymaga korekt.
- Założenia do potwierdzenia (OPEN-QUESTIONS, sekcja „później”): waluta `S` przy
  przewalutowaniu, źródło kursu `r`, dopuszczalność `M < 0`.

---

## ADR-0009: Brakujące pola słowników

- **Data:** 2026-09-24
- **Status:** przyjęta
- **Rozstrzyga:** OPEN-QUESTIONS pkt 9

### Kontekst

`seed.yaml` używał `entities.role` i `assets.issuer`, których nie było w schemacie SPEC.
ADR-0007 potrzebuje ról do walidacji kontrahentów, a ADR-0008 — waluty bazowej
e-pieniądza.

### Decyzja

- `entities.role` ∈ `emoney_issuer` | `network` | `otc_counterparty` | `null`.
- `assets.issuer_entity_id` i `assets.underlying_asset_id` — obowiązkowe wtedy i tylko
  wtedy, gdy `kind = emoney`.

### Konsekwencje

- `seed.yaml` dostaje `underlying` przy PLNdt i EURdt.
