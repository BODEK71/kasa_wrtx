# Otwarte pytania — krytyczny przegląd SPEC

Ten dokument nazywa problemy znalezione w `docs/SPEC.md` (w konfrontacji
z `CLAUDE.md` i `docs/seed.yaml`). Rozstrzygnięcia trafiają do
`docs/DECISIONS.md` i do SPEC — tu zostaje tylko ślad, co i gdzie rozstrzygnięto.

## Blokujące

Brak. Wszystkie punkty blokujące z przeglądu SPEC 0.1 są rozstrzygnięte w SPEC 0.2.

## Rozstrzygnięte

| # | Problem (SPEC 0.1) | Rozstrzygnięcie |
|---|---|---|
| 1 | §2 wymaga ≥2 zapisów na operację, a `OTC-OPEN`, `SH-OPEN`, `SH-CLOSE` nie mają zapisów | ADR-0001 — `otc_deals` + `deal_id`; zmiany jako RPC na `shifts` |
| 2 | `operations.source` dopuszczał `chain`/`exchange`/`hyllet`/`bank` wbrew zasadzie 8 | ADR-0002 — `source` ∈ {`ui`, `kantor_logic`}, hash w `tx_hash` |
| 3 | §2.6 „każda operacja ma tytuł prawny” vs `legal_basis null` i CLAUDE.md zasada 6 | ADR-0003 — bilans per spółka; dwie spółki tylko przez `TR-INTERCO` z `legal_basis` |
| 4 | Brak tabeli `customers` mimo `operations.customer_id` i §9 | ADR-0004 — osobna tabela; pola ustala MLRO (pkt 15) |
| 5 | `wallet` z jednym `address, chain` vs lista adresów w `seed.yaml` | ADR-0005 — `location_addresses` |
| 6 | `locations.entity_id` vs `accounts.entity_id` przy KZ „obu spółek” | ADR-0006 — jedno miejsce = jedna spółka; KZ = WRTX |
| 7 | Konta techniczne bez lokalizacji; klucz kont nie rozróżnia kontrahentów | ADR-0007 — `kind` + `counterparty_id`, `location_id` tylko dla `holding` |
| 8 | Niepełna formuła `EM-ISSUE`/`EM-REDEEM` (i „różnica → FX_POSITION”) | ADR-0008 — formuły dla tej samej waluty i z przewalutowaniem |
| 9 | `role` i `issuer` w `seed.yaml` bez odpowiednika w schemacie | ADR-0009 — `entities.role`, `assets.issuer_entity_id`, `assets.underlying_asset_id` |

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
    marży / udziału sieci (SPEC §11 pkt 4, `emoney_model.*` w seed.yaml).
    Zapisy z ADR-0008 działają w obu modelach; decyzja wpływa na saldo startowe
    `SETTLE/STABILLON` i sens alertu `FLOAT_LOW`.
14. Progi alertów i limity kasowe, w tym brakujący próg ustawowy dla
    `THRESHOLD_AML`, którego nie ma nawet jako `TODO` w `limits` w seed.yaml
    (SPEC §11 pkt 5, §8).
15. Zakres wspólnej kartoteki klienta między KRTX a WRTX — wprost oddane do
    decyzji MLRO (SPEC §11 pkt 6, §9). Od tego zależą kolumny i RLS tabeli
    `customers` (ADR-0004).
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

### Nowe — wynikające z rozstrzygnięć SPEC 0.2

20. Waluta udziału sieci `S` przy przewalutowaniu — ADR-0008 przyjmuje walutę
    gotówki; do potwierdzenia z umową sieci hyllet.cash.
21. Skąd pochodzi kurs `r` przy `EM-*` z przewalutowaniem (tabela `rates`,
    kurs podawany przez Hyllet, kurs z tablicy kasy?).
22. Czy ujemna marża (`M < 0`) bywa dopuszczalna (promocja, kurs niekorzystny) —
    ADR-0008 przyjmuje odrzucenie operacji.
23. `external_id` storna generowanego przez import (`KL_ROW_VANISHED`): oryginał
    zajmuje `(kantor_logic, external_id)`, więc storno nie może użyć tego samego
    klucza.
24. Wykrywanie `TRANSIT_STALE` na zbiorczym koncie `transit` per (spółka,
    aktywo): kilka równoległych ruchów w drodze może się wzajemnie maskować, a
    „od kiedy saldo jest niezerowe” jest niejednoznaczne.
25. Przynależność SEJF, WALLET_HOT i EXCHANGE_OKX — po ADR-0006 każde miejsce
    musi mieć jedną spółkę albo zostać rozbite na dwie lokalizacje.
