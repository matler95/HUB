# Hub — specyfikacja wymagań (v1.0)

> Robocza nazwa produktu: **Hub**. Nazwa docelowa i domena: do ustalenia (patrz sekcja 14, potrzebne przed wdrożeniem deep linków mobilnych).

---

## 0. Instrukcja dla agenta (przeczytaj najpierw)

1. Ten dokument jest źródłem prawdy o wymaganiach. Gdy czegoś brakuje lub jest sprzeczne, sprawdź **sekcję 14 (decyzje otwarte)**. Jeśli tam też nie ma odpowiedzi, **zadaj pytanie zamiast zgadywać**.
2. **Terminologia:** używaj wyłącznie słownika z sekcji 2 w kodzie, nazwach tabel, kolumn, endpointów, komunikatach, komentarzach, commitach, testach i danych testowych. Nie wprowadzaj własnego słownictwa branżowego.
3. **Priorytety (w tej kolejności):** bezpieczeństwo i izolacja danych → poprawność → prostota UX → tempo dostarczania. Przy konflikcie wygrywa wyższy priorytet.
4. Każde wymaganie ma ID (np. `F7`, `S3`). Odwołuj się do ID w commitach, PR-ach i testach.
5. **Konwencje:** kod, identyfikatory, schemat bazy, komentarze i commity po angielsku. Interfejs użytkownika po polsku, ale przez system i18n (klucze tłumaczeń, żadnych literałów w komponentach), żeby dodanie języków było tanie.
6. **Nigdy nie używaj prawdziwych danych** w repozytorium, dev, testach ani fixture'ach. Dane testowe generuj syntetycznie (fikcyjne imiona i nazwiska, wygenerowane obrazy).
7. Zasady pracy, definicja ukończenia i lista kontrolna bezpieczeństwa: **sekcja 15**.
8. To ma być produkt komercyjny, nie prototyp. Nie upraszczaj w obszarach bezpieczeństwa, uprawnień i integralności danych, nawet jeśli to spowalnia dostarczenie.

---

## 1. Cel produktu

**Hub** to wielodostępna (multi-tenant) platforma SaaS do bezpiecznego przechowywania i współdzielenia plików (głównie zdjęć i dużych plików obrazowych) w organizacjach z wieloma lokalizacjami. Użytkownik ma **jedno konto** i może być członkiem wielu przestrzeni z różnymi rolami. Dostęp do przestrzeni jest nadawany i odbierany indywidualnie, a pliki można wgrywać bezpośrednio z telefonu.

### Problemy, które rozwiązujemy

| Problem obecnie | Odpowiedź produktu |
|---|---|
| Pliki rozproszone po mailach i folderach w chmurach ogólnego przeznaczenia | jedno miejsce, uporządkowane wg przestrzeni i spraw |
| Wiele loginów i haseł, bo każdy udostępnia z innego systemu | jedno konto, wiele członkostw, zaproszenie mailem |
| Opóźnienia i ociężałość istniejących narzędzi | podpisane URL-e, miniatury, stronicowanie, szybki UI |
| Brak wygodnego dostępu z telefonu | PWA (MVP), potem natywne aplikacje iOS i Android |
| Zdjęcia z iPhone'a trzeba najpierw zgrać na komputer | upload z aparatu i galerii bezpośrednio z telefonu |

### Model użycia i biznesu

- Pierwotnie na użytek własny kilku osób, ale **od początku projektowany jako produkt na sprzedaż** (skalowalność, multi-tenancy, limity planów, panel operatora).
- Model przychodowy docelowo **freemium** (płatności poza MVP, ale limity i liczniki zużycia w modelu danych od początku).
- Dystrybucją zajmuje się partner biznesowy, więc **onboarding musi być w pełni samoobsługowy**.
- Język startowy: **polski**. Rynek startowy: Polska.
- Użytkownicy są **nietechniczni**: produkt ma być maksymalnie intuicyjny.

---

## 2. Słownik domenowy (obowiązujący)

| Termin w kodzie (EN) | Termin w UI (PL) | Znaczenie |
|---|---|---|
| `User` | Użytkownik | globalne konto (jeden login, jedno hasło) |
| `Space` | Przestrzeń | granica izolacji danych; odpowiada jednej lokalizacji/jednostce organizacyjnej; ma właściciela, członków, plan i limity |
| `Membership` | Członkostwo | powiązanie `User` ↔ `Space` z rolą |
| `Role` | Rola | `owner`, `editor`, `viewer` |
| `Invitation` | Zaproszenie | zaproszenie e-mailem do `Space` z rolą |
| `Case` | Sprawa (teczka) | zbiór materiałów dotyczących jednej osoby/rekordu wewnątrz `Space` |
| `Subject` | Osoba | osoba, której dotyczy `Case`; **przechowujemy imię i nazwisko** |
| `Asset` | Plik | plik lub folder wewnątrz `Case` (drzewo, wersjonowanie) |
| `Derivative` | Podgląd | wygenerowana miniatura lub podgląd pliku |
| `AuditEvent` | Zdarzenie | niezmienialny wpis dziennika dostępu i zmian |
| `BillingAccount` | Konto rozliczeniowe | jednostka rozliczeniowa (plan, limity); jeden `User` może mieć wiele `Space` pod jednym kontem |
| `StorageProvider` | — | abstrakcja fizycznego magazynu plików |

Dane w `Case` i `Subject` traktuj jako **dane osobowe o podwyższonej wrażliwości** (podejście jak do danych szczególnej kategorii w RODO). Wszystkie decyzje projektowe w sekcji 8 wynikają z tego założenia.

---

## 3. Kluczowe scenariusze użytkowników

| ID | Scenariusz | Kryterium akceptacji |
|---|---|---|
| U1 | Użytkownik ma członkostwa w kilku `Space` i loguje się raz | widzi wyłącznie `Space`, których jest członkiem; może przełączać się między nimi bez ponownego logowania |
| U2 | Właściciel zaprasza współpracownika tylko do jednej `Space` | współpracownik widzi tylko tę `Space`; **nie ma możliwości ustalenia, że inne `Space` istnieją** (odpowiedź 404, nie 403) |
| U3 | Współpracownik z innej organizacji ma milion własnych folderów, a z właścicielem współdzieli jedną `Space` | właściciel widzi tylko współdzieloną `Space` tego użytkownika, nie jego pozostałe zasoby |
| U4 | Użytkownik robi serię zdjęć telefonem i wysyła je do wybranej `Case` | wybór `Case` raz, seria zdjęć, jedno dotknięcie „Wyślij”; zdjęcia pojawiają się u współpracowników w ciągu kilku sekund |
| U5 | Użytkownik wybiera wiele zdjęć z galerii telefonu (w tym HEIC z iPhone'a) i wysyła je | wszystkie wgrane, podgląd działa, oryginały zachowane |
| U6 | Właściciel odbiera dostęp współpracownikowi | od następnego żądania API dostęp nie działa; ważne podpisane URL-e wygasają w ciągu ≤ 2 minut |
| U7 | Użytkownik szuka osoby po imieniu i nazwisku | wyniki ograniczone do `Space`, do których ma dostęp; szybka odpowiedź |
| U8 | Współpracownik dowiaduje się o nowych plikach | powiadomienie bez danych osobowych w treści |

---

## 4. Model domenowy i uprawnienia

```
User (konto globalne)
 └─ Membership (rola) ──► Space (granica izolacji, plan/limity przez BillingAccount)
                            └─ Case (Subject: imię i nazwisko)
                                 └─ Asset (pliki i foldery, wersje, Derivative)
```

### Reguły

1. **Granicą izolacji jest `Space`.** Każdy rekord zależny ma `space_id`; każda polityka dostępu sprawdza członkostwo.
2. **MVP: uprawnienia na poziomie `Space`.** Uprawnienia na poziomie pojedynczej `Case` to wersja późniejsza, ale model danych ma być na to przygotowany (patrz Dodatek A, tabela `case_grants`, nieużywana w MVP).
3. Jedno `User` może mieć wiele `Membership`. Tylko `owner` zaprasza i odbiera dostęp.
4. `Space` może mieć **wielu właścicieli**. Nie można usunąć ostatniego właściciela ani wyjść z `Space` jako ostatni właściciel.
5. Zaproszenie jest związane z konkretnym adresem e-mail, jednorazowe, wygasa po 7 dniach.
6. Brak członkostwa = brak widoczności. API zwraca 404 dla zasobów spoza zakresu użytkownika.

### Macierz uprawnień

| Akcja | owner | editor | viewer |
|---|---|---|---|
| Zapraszanie, zmiana ról, odbieranie dostępu | ✅ | ❌ | ❌ |
| Tworzenie i edycja `Case` | ✅ | ✅ | ❌ |
| Dodawanie plików i folderów | ✅ | ✅ | ❌ |
| Zmiana nazwy, przenoszenie, nowa wersja | ✅ | ✅ | ❌ |
| Przeniesienie do kosza | ✅ | ✅ *(założenie domyślne, sekcja 14)* | ❌ |
| Trwałe usunięcie, opróżnienie kosza | ✅ | ❌ | ❌ |
| Podgląd i pobieranie | ✅ | ✅ | ✅ *(założenie domyślne)* |
| Podgląd dziennika zdarzeń `Space` | ✅ | ❌ | ❌ |
| Zmiana ustawień `Space`, usunięcie `Space` | ✅ | ❌ | ❌ |

---

## 5. Wymagania funkcjonalne

Priorytety: **MVP** = wymagane w pierwszym wydaniu użytkowym; **V2** = następne; **LATER** = późniejsze.

### Konta i dostęp

| ID | Wymaganie | Kryterium akceptacji | Faza |
|---|---|---|---|
| F1 | Rejestracja, weryfikacja e-mail, logowanie, reset hasła | pełny przepływ działa; brak ujawniania, czy konto istnieje | MVP |
| F2 | 2FA (TOTP) dostępne dla każdego konta | włączenie, wyłączenie, kody zapasowe | MVP |
| F3 | Zarządzanie sesjami i urządzeniami | lista aktywnych sesji, zdalne wylogowanie jednej lub wszystkich | MVP |
| F4 | Tworzenie `Space`, zapraszanie przez e-mail z rolą | zaproszenie działa dla istniejącego i nowego użytkownika | MVP |
| F5 | Zmiana roli i odebranie dostępu | efekt natychmiastowy (U6) | MVP |
| F6 | Wielu właścicieli, przekazanie własności | nie można zostać bez właściciela | MVP |
| F7 | Przełącznik `Space` w UI | zawsze widać, w której `Space` się pracuje | MVP |
| F8 | Uprawnienia na poziomie pojedynczej `Case` | tabela `case_grants` aktywna, RLS rozszerzone, testy izolacji | V2 |
| F9 | Wymuszanie 2FA per `Space` | ustawienie właściciela | V2 |

### Zawartość

| ID | Wymaganie | Kryterium akceptacji | Faza |
|---|---|---|---|
| F10 | Tworzenie, edycja, archiwizacja `Case` z imieniem i nazwiskiem `Subject` | formularz prosty, walidacja, duplikaty ostrzegane, nie blokowane | MVP |
| F11 | Zagnieżdżone foldery wewnątrz `Case` | tworzenie, zmiana nazwy, przenoszenie | MVP |
| F12 | Upload z komputera: drag & drop, wiele plików, pasek postępu, wznawianie | patrz sekcja 6 | MVP |
| F13 | Upload z telefonu: aparat, wybór wielu zdjęć z galerii, **tryb serii** | patrz sekcja 10 | MVP |
| F14 | Podgląd: JPG, PNG, HEIC, PDF, wideo; pozostałe typy: ikona, metadane, pobranie | HEIC wyświetla się w każdej przeglądarce przez `Derivative` | MVP |
| F15 | Pobieranie pojedynczych plików i całych folderów/`Case` (ZIP) | ZIP strumieniowany, nie buforowany w pamięci | MVP |
| F16 | Kosz: przeniesienie, przywracanie, automatyczne trwałe usunięcie po okresie karencji | domyślnie 30 dni, konfigurowalne | MVP |
| F17 | Wyszukiwanie: po imieniu i nazwisku `Subject`, nazwie `Case`, nazwie pliku, tagach; filtr po `Space` i dacie | tylko w zakresie uprawnień użytkownika | MVP |
| F18 | Tagi i opis przy plikach | wielokrotne tagi, filtrowanie | MVP |
| F19 | Dziennik zdarzeń (kto, co, kiedy, skąd) z przeglądaniem przez właściciela | patrz S9 | MVP |
| F20 | Powiadomienia e-mail o nowych plikach (bez danych osobowych w treści) | konfigurowalne przez użytkownika, zbiorcze (digest), nie lawina maili | MVP |
| F21 | Wersjonowanie plików, nigdy nie nadpisujemy oryginału | edycja = nowa wersja, dostęp do poprzednich | V2 |
| F22 | Komentarze i notatki przy plikach | wątki, powiadomienia | V2 |
| F23 | Prosta edycja obrazu (obrót, przycięcie) jako nowa wersja | oryginał nietknięty | V2 |
| F24 | Linki udostępniania z hasłem i datą ważności | odwoływalne, logowane w dzienniku | V2 |
| F25 | Aktualizacje na żywo (real-time) listy plików | nowy plik pojawia się bez odświeżania | V2 |
| F26 | Eksport całych danych konta/`Space` | zgodność z żądaniami użytkowników | V2 |
| F27 | Przeglądarka specjalistycznych formatów obrazowych (serie 3D) | przeglądanie w przeglądarce | LATER |
| F28 | Integracje z urządzeniami zewnętrznymi (API/agent lokalny) | do zaprojektowania osobno | LATER |
| F29 | Natywne aplikacje iOS i Android (patrz sekcja 10) | **wymagane docelowo** | Faza 5 |

### Operator (właściciel produktu)

| ID | Wymaganie | Faza |
|---|---|---|
| A1 | Panel administracyjny: lista `Space` i kont rozliczeniowych, zużycie storage, blokada/odblokowanie konta | MVP |
| A2 | Panel kosztów i zużycia (GB per konto rozliczeniowe) | MVP |
| A3 | Wsparcie: podgląd metadanych konta **bez dostępu do treści plików** | MVP |
| A4 | Integracja płatności i plany freemium | V2 |

---

## 6. Pliki i upload (specyfikacja szczegółowa)

### Zakres formatów

Przyjmujemy **praktycznie dowolny plik** (jak popularne usługi chmurowe). Podgląd tam, gdzie to możliwe:

| Poziom | Formaty | Zachowanie |
|---|---|---|
| Pełny podgląd | JPG, PNG, **HEIC/HEIF**, PDF, wideo MP4/MOV | miniatura, podgląd w przeglądarce |
| Przechowanie i pobranie | archiwa ZIP, duże serie plików obrazowych, dokumenty biurowe, modele 3D, reszta | ikona, metadane, pobranie |

### Przepływ uploadu

1. Klient wysyła żądanie utworzenia sesji uploadu: `client_upload_id` (UUID generowany po stronie klienta), nazwa, rozmiar, deklarowany typ, `case_id`, opcjonalnie `parent_id`.
2. Serwer sprawdza uprawnienia i **limity (quota, rozmiar pliku)**, tworzy rekord `Asset` ze statusem `uploading` i **nieznaczącym kluczem w storage (UUID)**. Zwraca dane wznawialnego uploadu (tus lub S3 multipart, wybór w ADR).
3. Klient wgrywa **bezpośrednio do storage** (nie przez backend aplikacji).
4. Klient zgłasza zakończenie. Serwer weryfikuje obiekt (rozmiar, suma kontrolna) i ustawia status `processing`.
5. **Worker** (osobna usługa, nie funkcja serverless): wykrywa prawdziwy typ po zawartości, skanuje antywirusowo (np. ClamAV), generuje `Derivative` (miniatury, JPEG z HEIC, podglądy), **usuwa dane GPS z wersji podglądowych**. Oryginał pozostaje bez zmian. Status końcowy: `ready` lub `quarantined`.
6. Operacja jest **idempotentna** po `client_upload_id` (ponowienie nie tworzy duplikatu).

### Pobieranie

- Dostęp wyłącznie przez **krótkotrwałe podpisane URL-e** (TTL ≤ 2 minuty) generowane po sprawdzeniu uprawnień. Każde wygenerowanie zapisuje `AuditEvent`.
- Treści użytkowników serwowane z **oddzielnej domeny** i/lub z `Content-Disposition: attachment`, nigdy wykonywane w kontekście domeny aplikacji.

### Limity

- Maksymalny rozmiar pojedynczego pliku: konfigurowalny per plan; technicznie wsparcie dla plików **≥ 5 GB**.
- Quota storage per `BillingAccount`. Do zużycia liczone są **oryginały** (nie `Derivative`).
- Przekroczenie quota: czytelny komunikat po polsku, bez utraty danych i bez cichych błędów.

### Usuwanie

- Miękkie usunięcie → kosz (domyślnie 30 dni) → zadanie czyszczące usuwa obiekty ze storage, `Derivative` i rekordy.
- Usunięcie `Case` lub `Space` musi realnie usuwać pliki ze storage. Kopie zapasowe wygasają według ustalonego okresu (decyzja otwarta, domyślnie 35 dni).
- Konfigurowalna polityka retencji per `Space` (domyślnie brak automatycznego usuwania).

---

## 7. Wymagania niefunkcjonalne

### UX (priorytet wysoki)

- Docelowo użytkownik wgrywa zdjęcie do wybranej `Case` w **≤ 3 interakcjach** od otwarcia aplikacji.
- Interfejs po polsku, prosty język, **zero żargonu technicznego**, czytelne komunikaty błędów z podpowiedzią co zrobić.
- Duże cele dotykowe, układ mobile-first, ostatnio używane `Case` na górze.
- Stan uploadu zawsze widoczny ("12 z 14 wysłanych") i odporny na zerwanie połączenia.
- Dostępność: kontrast i etykiety zgodne z WCAG 2.1 AA jako cel.
- Przed każdym wydaniem: test użyteczności z 2-3 osobami nietechnicznymi.

### Wydajność

- Lista 50 plików z miniaturami: pierwsze wyświetlenie ≤ 1 s na typowym łączu mobilnym (cel).
- Miniatury leniwie ładowane, lista stronicowana kursorem, optymistyczny UI.
- Pliki nigdy nie przechodzą przez backend aplikacji (bezpośrednio do/ze storage).

### Skalowalność i niezawodność

- Multi-tenancy od początku (`space_id` wszędzie), brak założeń "jeden klient".
- Kopie zapasowe bazy i storage, **udokumentowane i przetestowane odtwarzanie** (próba przed pilotażem).
- Obserwowalność: logi strukturalne (bez danych osobowych), metryki, alerty, monitoring kosztów storage.
- Zdefiniowane cele RPO/RTO przed pilotażem.

---

## 8. Bezpieczeństwo i prywatność (wymagania obowiązkowe)

| ID | Wymaganie |
|---|---|
| S1 | **Izolacja na poziomie bazy (RLS) jako główna granica bezpieczeństwa**, a nie tylko filtrowanie w kodzie aplikacji. Wszystkie tabele z `space_id` mają polityki dostępu. |
| S2 | **Automatyczne testy izolacji w CI** (Dodatek B). Zmiana, która je psuje, nie może zostać scalona. |
| S3 | Storage prywatny; dostęp wyłącznie przez podpisane URL-e po sprawdzeniu uprawnień. |
| S4 | **Brak danych osobowych w:** kluczach obiektów storage, URL-ach, logach technicznych, treści powiadomień (mail, push), narzędziach błędów i analityce (skonfiguruj scrubbing). Imię i nazwisko `Subject` żyje wyłącznie w bazie. |
| S5 | Szyfrowanie w tranzycie (TLS) i w spoczynku; hosting w **UE** (preferencyjnie Polska, jeśli dostępne). |
| S6 | Hasła: haszowanie nowoczesnym algorytmem (przez dostawcę auth), sprawdzanie względem znanych wycieków, limity prób logowania, ochrona przed brute force. |
| S7 | Tokeny sesji krótkotrwałe + odświeżanie; zdalne unieważnianie sesji; weryfikacja uprawnień **przy każdym żądaniu** (nie tylko w tokenie). |
| S8 | Zaproszenia: tokeny jednorazowe, przechowywane jako hash, wygasające, związane z e-mailem; ochrona przed enumeracją kont. |
| S9 | **Dziennik zdarzeń niezmienialny** (rola aplikacji nie może aktualizować ani usuwać wpisów): logowanie, podgląd i pobranie pliku, upload, usunięcie, zmiany ról i zaproszeń, eksporty. Zawiera aktora, cel, czas, IP, user agent. |
| S10 | Walidacja typu pliku po zawartości, skan antywirusowy, kwarantanna, limity rozmiaru, serwowanie z osobnej domeny. |
| S11 | Nagłówki bezpieczeństwa (CSP, HSTS itd.), ochrona CSRF, walidacja wejścia na granicy API, bezpieczne obsłużenie nazw plików. |
| S12 | Sekrety poza repozytorium, osobne środowiska (dev / staging / prod), skanowanie zależności i sekretów w CI. |
| S13 | Zapisywanie akceptacji regulaminu i polityki prywatności (wersja, czas) w `terms_acceptances`. |
| S14 | Usuwanie konta i danych na żądanie (także wymóg sklepów z aplikacjami), z jasnym skutkiem dla `Space` współdzielonych. |
| S15 | Panel operatora nie daje dostępu do treści plików ani `Subject` bez jawnej, zalogowanej procedury awaryjnej (domyślnie niedostępne). |

> Aspekty prawne i organizacyjne (umowa powierzenia przetwarzania, rejestr czynności, procedura incydentów, regulamin, polityka prywatności) są po stronie właściciela produktu i **nie są zadaniem dla agenta**. Agent odpowiada za techniczne wsparcie tych wymagań (S4, S9, S13, S14, eksport, usuwanie).

---

## 9. Architektura (kierunek; szczegóły przez ADR)

Poniższe to **proponowany kierunek zgodny ze stackiem właściciela**. Każdy punkt oznaczony ADR wymaga krótkiego dokumentu decyzyjnego w `docs/adr/` przed implementacją.

- **Monorepo** (pnpm/turborepo): `apps/web` (Next.js, TypeScript), `apps/worker` (przetwarzanie plików), później `apps/mobile` (Expo). Współdzielone pakiety: `packages/types`, `packages/validation` (zod), `packages/api-client`, `packages/permissions`, `packages/upload` (maszyna stanów uploadu), `packages/design-tokens`.
- **Baza i auth:** PostgreSQL z RLS (np. Supabase, region UE). ADR: hosting.
- **API-first:** cała logika biznesowa za wersjonowanym, otypowanym API (`/api/v1`, OpenAPI lub generowany klient). Klienty (web, mobile) są wyłącznie konsumentami. Bezpośredni dostęp klienta do bazy tylko jeśli ADR to uzasadni i RLS jest pokryte testami.
- **Uwierzytelnianie tokenowe** (access + refresh), nie tylko cookie, żeby ten sam mechanizm działał w aplikacjach natywnych.
- **`StorageProvider`:** interfejs (`createUpload`, `getSignedDownloadUrl`, `delete`, `head`) z implementacją dla magazynu zgodnego z S3 w regionie UE. Brak kodu zależnego od konkretnego dostawcy poza adapterem. ADR: storage i protokół uploadu (tus vs S3 multipart).
- **Worker i kolejka zadań:** przetwarzanie plików poza ścieżką żądania; ponowienia, dead-letter. ADR: kolejka.
- **Powiadomienia:** warstwa abstrakcji z tabelą `devices` (platforma, token); dziś e-mail i Web Push, później APNs/FCM. Payload bez danych osobowych.
- **Deep linki** (Universal Links / App Links) uwzględnione w przepływie zaproszeń od początku.
- **Multi-tenancy i limity:** `BillingAccount` → plan → limity (`storage_quota_bytes`, `max_file_size_bytes`, `max_spaces`, `max_members`), liczniki zużycia aktualizowane transakcyjnie.
- **Obserwowalność:** logowanie strukturalne, identyfikatory korelacji, błędy bez danych osobowych.

---

## 10. Klienci: web, PWA i aplikacje natywne

### MVP: PWA

- Instalowalna (manifest, service worker), mobile-first.
- Aparat przez `<input type="file" accept="image/*" capture>` i wybór wielu zdjęć z galerii.
- **Tryb serii:** wybór `Case` raz → seria zdjęć → jedno „Wyślij”.
- **Kolejka offline** (IndexedDB): zdjęcia czekają na połączenie, wznawianie po wznowieniu aplikacji.
- Jawnie zaakceptowane ograniczenia iOS: brak uploadu w tle po zamknięciu aplikacji, brak pozycji w systemowym arkuszu „Udostępnij”, push tylko po dodaniu do ekranu głównego. UI informuje, że wysyłka trwa tylko przy otwartej aplikacji, jeśli kolejka nie jest pusta.

### Docelowo (Faza 5, wymagane): Expo / React Native dla iOS i Android

Ta sama logika i API, nowy klient. Priorytety natywne:

- **upload w tle** ze statusem i bezpiecznym ponawianiem,
- **systemowy „Udostępnij”** → wybór `Case`,
- **zdjęcia robione w aplikacji nie trafiają do rolki aparatu** (domyślnie piaskownica aplikacji); dla zdjęć wybranych z galerii opcja „usuń z telefonu po wysłaniu” (za zgodą systemu),
- **blokada biometryczna** (Face ID / odcisk), tokeny w Keychain/Keystore,
- zdalne wylogowanie przy zgubionym urządzeniu (F3),
- zgodność z wymaganiami sklepów: usuwanie konta w aplikacji, formularze prywatności spójne z polityką.

### Bramka decyzyjna

Po pilotażu (Faza 4) mierzymy: odsetek nieudanych lub ponawianych uploadów, liczbę kroków do wysłania serii, zgłoszenia braku „Udostępnij” i uploadu w tle. Wynik wyznacza zakres Fazy 5. **Zawsze** projektujemy zgodnie z sekcją 9, żeby port był tani.

---

## 11. Plany i monetyzacja (freemium)

- W MVP **brak płatności**, ale model danych i egzekwowanie limitów działają od początku.
- Osie limitów: storage (GB), liczba `Space`, liczba członków, maksymalny rozmiar pliku, okres przechowywania dziennika.
- Plan darmowy: twardy limit storage (koszt przechowywania obrazów jest znaczący).
- Rozliczenie preferowane poza aplikacjami mobilnymi (zasady sklepów sprawdzić przed projektowaniem cennika). Jednostka cenowa: decyzja otwarta.

---

## 12. Roadmapa i definicja ukończenia faz

| Faza | Zakres | Definicja ukończenia |
|---|---|---|
| **0. Fundamenty** | monorepo, CI/CD, środowiska, ADR-y, schemat bazy, szkielet RLS, **harness testów izolacji**, i18n, design tokens | pipeline zielony; testy izolacji uruchamiane w CI; pierwsze ADR-y zaakceptowane |
| **1. Dostęp** | F1-F7, S1-S3, S6-S9, S13 (rdzeń) | wszystkie testy izolacji zielone; zaproszenia i odebranie dostępu działają; krótki dokument modelu zagrożeń |
| **2. Zawartość** | F10-F20, sekcja 6, A1-A3, S4, S10 | upload, przetwarzanie, podgląd, pobranie, kosz, wyszukiwanie, dziennik i limity działają end-to-end; pliki testowe obejmują HEIC i duże pliki |
| **3. Mobile PWA** | F13, tryb serii, kolejka offline, polerowanie UX | test użyteczności z 2-3 osobami zaliczony; kryteria U4, U5 spełnione |
| **4. Pilotaż i utwardzanie** | ograniczony pilotaż, metryki, próba odtworzenia z backupu, checklista bezpieczeństwa, przegląd zależności | udokumentowane odtworzenie; metryki bramki z sekcji 10; brak otwartych błędów krytycznych |
| **5. Aplikacje natywne** | F29 (Expo, iOS + Android) | w sklepach / w testach zamkniętych; priorytety natywne z sekcji 10 |
| **Później** | V2 i LATER z sekcji 5, płatności | wg priorytetów |

---

## 13. Poza zakresem (na teraz)

- Funkcje diagnostyczne, analiza obrazów, jakiekolwiek zalecenia na podstawie treści plików.
- Płatności i fakturowanie (V2).
- Integracje ze skanerami i urządzeniami (LATER).
- Przeglądarka specjalistycznych formatów obrazowych (LATER).
- Odbiór plików przez e-mail, OCR, automatyczne rozpoznawanie osób.
- Aplikacje natywne w MVP (Faza 5).
- Wiele języków interfejsu (architektura i18n gotowa, treści tylko po polsku).

---

## 14. Decyzje otwarte (do czasu rozstrzygnięcia obowiązują wartości domyślne)

| # | Decyzja | Wartość domyślna dla agenta |
|---|---|---|
| D1 | Czy `editor` może przenosić do kosza? | tak; trwałe usunięcie tylko `owner` |
| D2 | Czy `viewer` może pobierać pliki? | tak; przełącznik per `Space` jako V2 |
| D3 | Uprawnienia na poziomie `Case` | V2; w MVP wyłącznie na poziomie `Space`, model gotowy |
| D4 | Wielu właścicieli `Space` | tak |
| D5 | Czyje limity zużywają pliki edytorów | `BillingAccount` właściciela `Space` |
| D6 | Jednostka cenowa freemium | nierozstrzygnięta; model wspiera storage, liczbę `Space` i członków |
| D7 | Retencja | konfigurowalna per `Space`; domyślnie brak automatycznego usuwania; kosz 30 dni; kopie zapasowe 35 dni |
| D8 | Maksymalny rozmiar pliku per plan | konfigurowalny; techniczne wsparcie ≥ 5 GB |
| D9 | Region hostingu | UE; Polska, jeśli dostępne po rozsądnym koszcie |
| D10 | Nazwa produktu i domena | do ustalenia; potrzebne przed deep linkami (Faza 3/5) |
| D11 | Strona umowy powierzenia przetwarzania (kto z kim) | decyzja prawna właściciela; bez wpływu na kod poza S13 |
| D12 | Czy aplikacja ma być miejscem docelowym archiwizacji | domyślnie **nie**: traktujemy ją jako warstwę roboczą; komunikat w regulaminie i UI |

---

## 15. Zasady pracy agenta i definicja ukończenia

### Zasady

1. **Planuj przed kodowaniem.** Dla każdej funkcji: krótki plan, wskazanie ID wymagań, lista zmian w schemacie i uprawnieniach.
2. **Migracje bazy** zawsze w kontroli wersji, odwracalne, z politykami RLS w tej samej migracji co tabela. Tabela bez RLS nie może zostać scalona.
3. **Testy:** jednostkowe dla logiki, integracyjne dla API, **testy izolacji** dla każdej nowej tabeli i endpointu (Dodatek B), testy E2E dla krytycznych ścieżek (zaproszenie, upload, odebranie dostępu).
4. **ADR** dla decyzji wymienionych w sekcji 9. Krótki, w repozytorium.
5. **Małe, odwracalne zmiany**, jeden temat na PR, opis powiązany z ID wymagań.
6. **Nie dodawaj zależności bez uzasadnienia** (utrzymanie, licencja, bezpieczeństwo).
7. **Dane testowe wyłącznie syntetyczne.** Nie loguj danych osobowych, nawet w dev.
8. Gdy wymaganie jest niejasne lub sprzeczne: zatrzymaj się i zapytaj.

### Definicja ukończenia (DoD) dla każdej funkcji

- [ ] Spełnia kryteria akceptacji z tabeli wymagań.
- [ ] Uprawnienia egzekwowane w bazie (RLS) **i** w API; dodane testy izolacji.
- [ ] Zdarzenia istotne dla bezpieczeństwa trafiają do dziennika (S9).
- [ ] Brak danych osobowych w logach, kluczach storage, powiadomieniach (S4).
- [ ] Teksty interfejsu w i18n, po polsku, prostym językiem.
- [ ] Działa na telefonie (mobile-first) i na komputerze.
- [ ] Obsłużone stany: ładowanie, błąd, pusty widok, brak uprawnień, przekroczony limit.
- [ ] Testy przechodzą w CI; brak nowych ostrzeżeń bezpieczeństwa.
- [ ] Dokumentacja i ADR zaktualizowane, jeśli dotyczy.

---

## Dodatek A: szkic schematu danych (kierunkowy)

Wszystkie tabele zależne od `Space` zawierają `space_id` i mają włączone RLS. Poniżej tylko kluczowe pola; szczegóły wynikają z implementacji.

| Tabela | Kluczowe pola |
|---|---|
| `users` | id, email, display_name, locale, created_at (dane uwierzytelniania u dostawcy auth) |
| `billing_accounts` | id, owner_user_id, plan, storage_quota_bytes, max_file_size_bytes, max_spaces, max_members |
| `spaces` | id, billing_account_id, name, created_by, archived_at |
| `memberships` | id, space_id, user_id, role (`owner`/`editor`/`viewer`), status, invited_by, created_at |
| `invitations` | id, space_id, email, role, token_hash, expires_at, accepted_at, revoked_at |
| `cases` | id, space_id, subject_full_name, title, notes, created_by, archived_at, deleted_at |
| `assets` | id, space_id, case_id, parent_id, kind (`file`/`folder`), name, size_bytes, detected_mime, checksum, storage_key (UUID), version_group_id, version_no, uploaded_by, status (`uploading`/`processing`/`ready`/`quarantined`), client_upload_id (unikalne), deleted_at, purge_after |
| `asset_derivatives` | id, asset_id, space_id, kind (`thumbnail`/`preview`), storage_key |
| `tags`, `asset_tags` | id, space_id, name; powiązania |
| `audit_events` | id, space_id, actor_user_id, action, target_type, target_id, metadata (bez danych osobowych), ip, user_agent, created_at; **append-only** |
| `devices` | id, user_id, platform, push_token, last_seen_at |
| `usage_counters` | billing_account_id, bytes_used, updated_at |
| `retention_policies` | space_id, trash_days, auto_purge_days (nullable) |
| `terms_acceptances` | user_id, document_version, accepted_at |
| `case_grants` | case_id, user_id, role (**nieużywana w MVP**, przygotowanie pod F8) |
| `comments` | (V2) |

Indeksy: `space_id` jako prefiks indeksów złożonych; wyszukiwanie po `subject_full_name` i nazwach z użyciem indeksu pełnotekstowego/trigramowego w zakresie jednej `Space`.

---

## Dodatek B: minimalny zestaw testów izolacji (CI)

Każdy test uruchamiany jako rzeczywisty użytkownik (nie jako rola serwisowa):

1. Użytkownik bez członkostwa w `Space` X nie widzi **żadnego** rekordu z X (listy, szczegóły, wyszukiwanie, liczniki) i dostaje 404.
2. `viewer` nie może tworzyć, edytować, usuwać ani zapraszać (każdy endpoint modyfikujący zwraca odmowę).
3. `editor` nie może zapraszać, zmieniać ról, trwale usuwać ani czytać dziennika.
4. Użytkownik z członkostwem w `Space` A i B widzi A i B, ale **nie** C.
5. Współpracownik zaproszony tylko do A nie może ustalić istnienia B (brak różnicy w odpowiedziach między „nie istnieje” a „nie masz dostępu”).
6. Po odebraniu członkostwa kolejne żądanie API jest odrzucone natychmiast; wygenerowane wcześniej podpisane URL-e wygasają w czasie ≤ TTL.
7. Podpisany URL jednej `Space` nie działa dla zasobu innej `Space` (zmiana ścieżki/klucza).
8. Nie da się zapisać `Asset` z `case_id` należącym do innej `Space` niż `space_id`.
9. Użytkownik nie może zmienić własnej roli ani `space_id` zasobu przez bezpośrednie wywołanie API/bazy.
10. Wpisy `audit_events` nie mogą być modyfikowane ani usuwane przez rolę aplikacji.
11. Zaproszenie nie może być użyte przez inny adres e-mail ani drugi raz, ani po wygaśnięciu/odwołaniu.
12. Wyszukiwanie po imieniu i nazwisku zwraca wyłącznie `Case` z `Space`, do których użytkownik ma dostęp.
13. Operator (rola administracyjna) bez jawnej procedury awaryjnej nie odczytuje treści plików ani `subject_full_name`.
