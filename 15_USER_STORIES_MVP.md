# 15_USER_STORIES_MVP

## Cel
User Stories dla modułów MVP z kryteriami akceptacji. Definicja "done" dla każdej funkcjonalności.

---

## Epic 1: Logowanie i Autoryzacja

### US-001: Logowanie operatora
**Jako** OPERATOR, **chcę** zalogować się emailem i hasłem, **aby** rozpocząć dyżur monitorowania.

**Kryteria akceptacji:**
- AC1: Ekran logowania z polami email + hasło
- AC2: Po poprawnym logowaniu → redirect do dashboardu alarmów
- AC3: Po błędnym logowaniu → komunikat "Nieprawidłowy email lub hasło" (nie ujawniaj co jest błędne)
- AC4: Po 5 nieudanych próbach w 1 min → blokada IP na 1 min
- AC5: Po 10 nieudanych próbach → konto zablokowane na 30 min + log w AUDIT_LOG
- AC6: Token JWT przechowywany w pamięci (nie localStorage)

### US-002: Automatyczne odświeżanie sesji
**Jako** OPERATOR, **chcę** żeby sesja działała bez konieczności ponownego logowania, **aby** nie tracić czasu na dyżurze.

**Kryteria akceptacji:**
- AC1: Access Token odświeża się automatycznie przed wygaśnięciem
- AC2: Sesja trwa max 7 dni (refresh token)
- AC3: Po 7 dniach → wylogowanie z komunikatem

---

## Epic 2: Dashboard Alarmów

### US-003: Widok listy alarmów
**Jako** OPERATOR, **chcę** widzieć listę aktywnych alarmów, **aby** szybko zidentyfikować incydenty do obsługi.

**Kryteria akceptacji:**
- AC1: Lista Bundle Alarms (nie surowych eventów)
- AC2: Sortowanie domyślne: first_seen DESC (najnowsze na górze)
- AC3: Kolory: CRITICAL → czerwony, WARNING → pomarańczowy, INFO → szary
- AC4: Alarmy NEW → migająca ikona / animacja
- AC5: Wyświetlane pola: nazwa obiektu, typ alarmu, priorytet, status, czas, liczba eventów, przypisany operator
- AC6: Filtrowanie po: status, priorytet, obiekt
- AC7: Aktualizacja w real-time (WebSocket) — nowy alarm pojawia się natychmiast bez odświeżania

### US-004: Obsługa alarmu (Claim)
**Jako** OPERATOR, **chcę** kliknąć "Obsługuj", **aby** pozostali operatorzy widzieli, że alarm jest w trakcie obsługi.

**Kryteria akceptacji:**
- AC1: Przycisk "Obsługuj" widoczny przy alarmach w statusie NEW
- AC2: Po kliknięciu → status zmienia się na IN_PROGRESS
- AC3: Wyświetla się imię operatora przy alarmie
- AC4: Jeśli inny operator kliknął pierwszy → komunikat "Alarm już obsługiwany przez [imię]"
- AC5: Zmiana statusu widoczna natychmiast u wszystkich operatorów (WebSocket)

### US-005: Potwierdzenie alarmu (ACK)
**Jako** OPERATOR, **chcę** potwierdzić alarm z notatką, **aby** udokumentować swoją reakcję.

**Kryteria akceptacji:**
- AC1: Przycisk "Potwierdź" widoczny przy alarmach IN_PROGRESS
- AC2: Wymagane pole „Notatka" (min. **10 znaków** — spójne z kodem błędu `NOTE_TOO_SHORT` w API)
- AC3: Po potwierdzeniu → status ACK
- AC4: Notatka zapisana w AUDIT_LOG

### US-006: Rozwiązanie alarmu (Resolve)
**Jako** OPERATOR, **chcę** oznaczyć alarm jako rozwiązany, **aby** oddzielić "potwierdzony" od "problem naprawiony".

**Kryteria akceptacji:**
- AC1: Przycisk "Rozwiązany" widoczny przy alarmach ACK
- AC2: Opcjonalna notatka
- AC3: Status → RESOLVED

### US-007: Zamknięcie alarmu (Close)
**Jako** OPERATOR, **chcę** zamknąć alarm, **aby** trafił do historii.

**Kryteria akceptacji:**
- AC1: Przycisk "Zamknij" widoczny przy alarmach RESOLVED
- AC2: Dla alarmów temperaturowych (`requires_note = true`) → notatka OBOWIĄZKOWA (min. 10 znaków)
- AC3: Brak notatki → walidacja: "Alarm temperaturowy wymaga notatki wyjaśniającej"
- AC4: Po zamknięciu → alarm znika z listy aktywnych, trafia do historii
- AC5: Zamknięty alarm nie może być ponownie otwarty

### US-008: Dźwięk alarmu
**Jako** OPERATOR, **chcę** słyszeć dźwięk przy nowym alarmie CRITICAL, **aby** zareagować nawet gdy nie patrzę na ekran.

**Kryteria akceptacji:**
- AC1: CRITICAL → dźwięk ciągły do momentu claim
- AC2: WARNING → dźwięk pojedynczy
- AC3: INFO → brak dźwięku
- AC4: Przycisk "Wycisz" na 5 min

---

## Epic 3: Obiekty

### US-009: Lista obiektów
**Jako** OPERATOR, **chcę** przeglądać listę obiektów, **aby** szybko znaleźć potrzebny adres lub telefon.

**Kryteria akceptacji:**
- AC1: Lista z wyszukiwaniem (po nazwie, adresie)
- AC2: Filtrowanie po statusie (ACTIVE, SERVICE, CLOSED)
- AC3: Wyświetlane: nazwa, adres, status, liczba aktywnych alarmów

### US-010: Szczegóły obiektu
**Jako** OPERATOR, **chcę** zobaczyć szczegóły obiektu, **aby** mieć pełen kontekst przy obsłudze alarmu.

**Kryteria akceptacji:**
- AC1: Adres, kontakt, GPS (mapa)
- AC2: Lista central przypisanych do obiektu
- AC3: Ostatnie alarmy
- AC4: Dane CCTV (tylko login, IP — hasło ukryte do Timed Reveal)

### US-011: Dodanie obiektu
**Jako** ADMIN, **chcę** dodać nowy obiekt do systemu, **aby** monitorować kolejną lokalizację.

**Kryteria akceptacji:**
- AC1: Formularz: nazwa, adres, typ, kontakty
- AC2: Geokodowanie adresu → automatyczne wypełnienie lat/lon
- AC3: Możliwość ręcznej korekty pinezki na mapie
- AC4: Walidacja: unikalna nazwa obiektu

---

## Epic 4: Integracja Satel

### US-012: Widok stanu centrali (stan live z Redis)
**Jako** OPERATOR, **chcę** widzieć aktualny stan central w real-time, **aby** wiedzieć co jest uzbrojone/rozbrojone.

**Kryteria akceptacji:**
- AC1: Widok per panel: lista stref (stan: Armed/Disarmed/Alarm)
- AC2: Lista wejść (stan: OK/Violation/Tamper)
- AC3: Status połączenia Worker (Connected/Reconnecting/Disconnected)
- AC4: Timestamp ostatniego odczytu
- AC5: Dane odświeżane automatycznie (z Redis, co 2s)

### US-013: Notyfikacja o utracie łączności
**Jako** OPERATOR, **chcę** być poinformowany gdy Worker straci połączenie z centralą, **aby** podjąć działania.

**Kryteria akceptacji:**
- AC1: Jeśli Worker "DISCONNECTED" > 2 min → banner ostrzegawczy
- AC2: Jeśli > 5 min → alarm systemowy (CRITICAL)

---

## Epic 5: Integracja SMS

### US-014: Alarm temperatury z SMS
**Jako** SYSTEM, **chcę** automatycznie tworzyć alarm po odebraniu SMS z Efento/Bluelog, **aby** operator został powiadomiony.

**Kryteria akceptacji:**
- AC1: SMS z zaufanego numeru → parsowanie → Raw Event → Bundle Alarm
- AC2: Alarm temperaturowy ma `requires_note = true`
- AC3: SMS z nieznanego numeru → ignoruj + log
- AC4: SMS "Powrót do normy" / "Koniec alertu" → zmiana statusu na INFO (zamknięcie alarmu temp.)

---

## Epic 6: Timed Secret Reveal

### US-015: Odsłonięcie hasła
**Jako** OPERATOR / TECHNIK / AUDYTOR, **chcę** odsłonić hasło do rejestratora CCTV, **aby** sprawdzić obraz z kamery lub wejść do strefy.

**Kryteria akceptacji:**
- AC1: Przycisk "Pokaż hasło" przy danych CCTV
- AC2: Wymagany powód (pole tekstowe, min. 5 znaków)
- AC3: Hasło widoczne przez 60s z countdownem
- AC4: Po 60s → hasło ukryte automatycznie
- AC5: Zdarzenie w AUDIT_LOG: kto, co, kiedy, powód
- AC6: Limity: Operator 10/h, Technik 20/h, Audytor 20/h

### US-016: Zgłoszenie Serwisowe (Manual Alarm)
**Jako** AUDYTOR / TECHNIK, **chcę** manualnie zgłosić usterkę lub wynik testu, **aby** system zarejestrował incydent wymagający obsługi.

**Kryteria akceptacji:**
- AC1: Przycisk "Zgłoś problem" na widoku obiektu
- AC2: Formularz: Priorytet, Opis, Typ (Maintenance/Test)
- AC3: Utworzenie nowego Bundle Alarm w statusie NEW
- AC4: Operatorzy widzą zgłoszenie na Dashboardzie

---

## Definition of Done (globalna)

Feature jest "done" gdy:
1. ✅ Wszystkie AC spełnione
2. ✅ Unit testy napisane i zielone (backend)
3. ✅ Brak ostrzeżeń lint (frontend + backend)
4. ✅ Zmiana ma review od co najmniej 1 osoby
5. ✅ Działa na środowisku STAGE
6. ✅ Dokumentacja zaktualizowana (jeśli dotyczy)
