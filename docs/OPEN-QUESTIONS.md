# Otwarte pytania — krytyczny przegląd SPEC

Ten dokument tylko nazywa problemy znalezione w `docs/SPEC.md` (w konfrontacji
z `CLAUDE.md` i `docs/seed.yaml`) — nie proponuje rozwiązań. Podzielony na to,
co blokuje start prac nad schematem/migracjami, i to, co można rozstrzygnąć
później, w trakcie budowy.

## Blokujące

Rzeczy dotykające kształtu schematu bazy lub logiki RPC — bez rozstrzygnięcia
nie da się napisać pierwszej migracji ani pierwszej funkcji księgującej.

1. **Sprzeczność co do liczby zapisów na operację.** SPEC §2 zasada 2: „Każda
   operacja rozpisuje się na co najmniej dwa zapisy.” SPEC §4.3: `OTC-OPEN` —
   „brak zapisów, tylko nagłówek + kurs”. Prawdopodobnie tak samo `SH-OPEN` /
   `SH-CLOSE` (§4.4). Nie wiadomo, czy inwariant „≥2 postings” jest twardym
   ograniczeniem w bazie (i wtedy te operacje nie mogą istnieć w `operations`
   tak, jak opisano), czy jest warunkowy.

2. **Sprzeczność co do księgowania przez integracje zewnętrzne.** CLAUDE.md
   zasada 8 i SPEC §6: blockchain / giełda / bank / Hyllet „nigdy nie tworzą
   zapisów w księdze”, zasilają wyłącznie `external_balances`. Jednocześnie
   `operations.source` (SPEC §3.2) ma wprost wartości `'chain' | 'exchange' |
   'hyllet' | 'bank'` jako źródła *operacji*, czyli zapisów księgowych — nie
   tylko sald zewnętrznych.

3. **Sprzeczność co do obowiązkowości `legal_basis`.** SPEC §2 zasada 6: „Każda
   operacja ma podmiot i tytuł prawny.” Schemat (§3.2) ma `legal_basis text
   null`, a CLAUDE.md zasada 6 mówi węziej: obowiązkowy tylko przy ruchach
   *między podmiotami* (przykład: `TR-INTERCO`). Nie wiadomo, czy reguła z §2
   dotyczy dosłownie każdej operacji, czy jest tam sformułowana zbyt szeroko.

4. **Brak tabeli `customers`.** `operations.customer_id` (§3.2) i §9 (AML,
   „wspólna kartoteka klienta”) odwołują się do bytu klienta detalicznego,
   którego nie ma w modelu danych (§3). Niejasne też, czy klienci mieliby żyć
   w `entities` (opisanych w SPEC jako „podmioty prawne”), czy w osobnej
   tabeli.

5. **Model portfeli wieloadresowych vs schemat lokalizacji.** SPEC §3.1:
   lokalizacja typu `wallet` ma pojedyncze pola `address, chain`. `seed.yaml`:
   `WALLET_HOT` ma listę `addresses` z dwoma różnymi chainami (`tron`,
   `ethereum`) pod jedną lokalizacją. Kształty się wykluczają.

6. **Niejasna relacja `locations.entity_id` vs `accounts.entity_id`.** SPEC §1
   (tabela kas) i `seed.yaml` (komentarz przy `KZ`) mówią, że kasa `KZ` może
   obsługiwać dwa podmioty naraz („KRTX czy WRTX, albo obie — wtedy dwa konta
   na tej kasie”), ale `locations` (§3.1) ma jedno pole `entity_id` na
   lokalizację, podczas gdy `accounts` ma własne, niezależne `entity_id`. Nie
   wiadomo, co wtedy oznacza `entity_id` na samej lokalizacji i czy jest w
   ogóle potrzebne.

7. **`accounts.location_id` a konta techniczne.** Schemat `accounts` (§3.2)
   wymaga `location_id`. §3.2 wymienia listę „kont technicznych” (`TRANSIT`,
   `FX_POSITION`, `MARGIN`, `FEES`, `DIFF`, `OPENING`, `SETTLE/STABILLON`,
   `SETTLE/NETWORK`, `INTERCO/<podmiot>`, `RECV/<kontrahent>`), z których
   część (PnL, rozrachunki, bilans otwarcia) nie ma naturalnego fizycznego
   miejsca. Nie wiadomo, czy każde z nich wymaga syntetycznej lokalizacji,
   czy `location_id` jest w praktyce nullable dla kont innych niż `real`.
   Powiązane: jak klucz unikalny `(location_id, asset_id, entity_id)` ma
   rozróżniać wielu kontrahentów pod `RECV/<kontrahent>` albo wiele spółek
   pod `INTERCO/<podmiot>`.

8. **Niepełna formuła księgowania `EM-ISSUE` / `EM-REDEEM`.** SPEC §4.2
   wymienia cztery zapisy (`KH/PLN`, `SETTLE/STABILLON`, `MARGIN`,
   `SETTLE/NETWORK`), ale nie mówi, w jakim aktywie księgowane są te trzy
   ostatnie (PLN? PLNdt?) ani jaka jest zależność arytmetyczna między kwotą
   gotówki, nominałem e-pieniądza, marżą i udziałem sieci, która ma dać sumę
   zero per aktywo (§2 zasada 2). To centralna operacja systemu — bez tej
   formuły nie da się napisać RPC.

9. **Pola w `seed.yaml`, których nie ma w schemacie SPEC.** `entities` w
   `seed.yaml` mają pole `role` (np. `emoney_issuer`, `network`,
   `otc_counterparty`), `assets` mają pole `issuer`. Schemat w SPEC §3.1
   (`entities`: `id, code, name, nip, krs, is_internal`; `assets`: `id, code,
   kind, scale, chain, display_name, is_active`) nie przewiduje żadnego z
   tych pól.

## Do rozstrzygnięcia później

Nie blokują pierwszych kroków — albo są już świadomie odłożone przez sam SPEC
(§11) i `seed.yaml` (pola `TODO`), albo dotyczą szczegółów operacyjnych, które
można ustalić równolegle z pracami nad schematem.

10. Format i granularność eksportu z Kantor-Logic (SPEC §11 pkt 1,
    `kantor_logic.*` w seed.yaml) — wymaga obejrzenia realnej próbki.
11. Na której spółce jest kasa Hyllet (SPEC §11 pkt 2, `locations[KH].entity`
    w seed.yaml).
12. Pełna lista walut/tokenów i sieci (SPEC §11 pkt 3, `assets` w seed.yaml).
13. Czy e-pieniądz jest prefinansowany czy rozliczany po fakcie, oraz podstawa
    marży / udziału sieci (SPEC §11 pkt 4, `emoney_model.*` w seed.yaml) —
    powiązane z punktem blokującym 8, ale samą decyzję biznesową można podjąć
    równolegle z pracami nad schematem.
14. Progi alertów i limity kasowe, w tym brakujący próg ustawowy dla
    `THRESHOLD_AML`, którego nie ma nawet jako `TODO` w `limits` w seed.yaml
    (SPEC §11 pkt 5, §8).
15. Zakres wspólnej kartoteki klienta między KRTX a WRTX — wprost oddane do
    decyzji MLRO (SPEC §11 pkt 6, §9).
16. Kto/co jest `created_by` dla operacji tworzonych automatycznie (import,
    storna z `KL_ROW_VANISHED`) — SPEC wymaga `created_by` „bez wyjątków” (§2
    zasada 8), ale nie mówi, jaka wartość reprezentuje „system” jako autora.
17. `shift_id` dla operacji obejmujących dwie lokalizacje w różnych zmianach
    (np. `TR-MOVE` między dwiema kasami) — do której zmiany przypisać
    operację.
18. Odniesienie „sprawdzone w generatorze KP/KW” (SPEC §10, jsPDF) — nie
    wiadomo, czy to istniejący kod do przeniesienia, czy coś do zbudowania od
    zera.
19. Adresy portfeli i dane banków to w większości `TODO` w `seed.yaml` — to
    dane do uzupełnienia, nie problem modelu.
