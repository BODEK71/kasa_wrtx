# Księga skarbcowa trzech kas

System ewidencji gotówki, e-pieniądza i krypto dla trzech kas w jednej lokalizacji:
kantor walutowy (KRTX), kasa Hyllet (model hyllet.cash), kasa wymian OTC (WRTX/KRTX).

Pełna specyfikacja: @docs/SPEC.md
Plan budowy i kolejność kroków: `docs/PLAN.md`
Dane startowe: `docs/seed.yaml`

## Czym to jest

To jest księga, nie aplikacja CRUD. Zapis raz wprowadzony jest dowodem w postępowaniu
kontrolnym. Kod, który „na szybko” poprawia dane, niszczy jedyną wartość tego systemu.

## Zasady nienaruszalne

Złamanie którejkolwiek z nich to błąd krytyczny, nawet jeśli testy przechodzą.

1. **Saldo nie istnieje jako pole.** Stan to zawsze suma zapisów. Nie dodawaj kolumny
   `balance`, nie cache'uj salda bez wyraźnej decyzji w SPEC.
2. **Podwójny zapis per spółka i aktywo.** Dla każdej operacji, każdej spółki
   (`accounts.entity_id`) i każdego aktywa suma zapisów = 0. Pilnuje tego constraint
   trigger w bazie. Nie wyłączaj go, nie obchodź.
3. **Tylko dopisywanie.** Żadnego UPDATE ani DELETE na `operations` i `postings`.
   Korekta to zawsze storno: nowa operacja z `reverses_id` i `reversal_reason`.
4. **Kwoty jako `numeric` i `string`.** Nigdy `float`, nigdy `number` w JS, nigdy
   `parseFloat`. W TypeScripcie `decimal.js`, na granicy API string. To dotyczy też
   sum, procentów, marż i kursów.
5. **Klient nie pisze do księgi.** Żadnego `.insert()` na `operations`/`postings` z frontu.
   Jedyna droga to funkcje RPC `security definer`.
6. **Każdy ruch między podmiotami ma tytuł prawny.** Operacja dotykająca ksiąg dwóch
   spółek jest możliwa tylko jako `TR-INTERCO`, a `TR-INTERCO` bez `legal_basis` ma się
   nie dać zapisać.
7. **Import jest idempotentny.** `unique (source, external_id)`. Dwukrotny import tego
   samego pliku nie zmienia sald.
8. **Integracje zewnętrzne nie księgują.** Blockchain, giełda, bank i Hyllet zasilają
   wyłącznie `external_balances` i alerty uzgodnieniowe.

## Konwencje

- Migracje: wyłącznie przez `supabase migration new <nazwa>`, nigdy ręczna edycja
  zastosowanej migracji. Zmiana schematu = nowa migracja.
- Nazwy w bazie po angielsku, `snake_case`. Kody operacji z katalogu w SPEC sekcja 4.
- Komentarze i UI po polsku. Etykiety w UI biorą się ze słownika, nie z kodu operacji.
- Każda nowa funkcja RPC dostaje test pgTAP w tej samej zmianie.
- Kwoty w UI formatuje jedna funkcja, zgodnie z `assets.scale`. Nie formatuj lokalnie.
- Czas: `timestamptz`, zapisy w UTC, prezentacja w `Europe/Warsaw`.
- Sekrety w `.env.local` i w Vercel. Klucz `service_role` nigdy nie trafia do przeglądarki.
- Klucze API giełd tylko read-only. Klucz prywatny portfela nie ma prawa istnieć w repo
  ani w bazie — nawet w komentarzu, nawet zakomentowany.

## Jak pracujemy

- Zaczynamy od `/plan`. Przy zmianach dotykających schematu lub księgowania plan jest
  obowiązkowy i musi wskazać, które zasady nienaruszalne są w grze.
- Jeden krok planu = jedna sesja. Po zamknięciu kroku `/clear`.
- Nie dopisuj funkcji, o które nikt nie prosił. Zakres bierze się z `docs/PLAN.md`.
- Jeśli wymaganie w SPEC jest niejasne albo sprzeczne — pytaj, nie zgaduj. Zgadnięta
  reguła księgowa jest gorsza niż brak funkcji.
- Jeśli implementacja wymusza odstępstwo od SPEC, najpierw zmień SPEC i powiedz o tym.

## Czego nie robić

- Nie instaluj ORM-a. Logika pieniędzy jest w SQL i tam zostaje.
- Nie dodawaj „trybu administratora”, który pozwala edytować zapisy.
- Nie generuj danych przykładowych w migracjach produkcyjnych. Seed osobno.
- Nie ukrywaj błędu uzgodnienia. Lepiej zablokować zamknięcie dnia niż pokazać ładny zero.
