# 03_FUNCTIONAL_MODULES

## Functional Modules – Source of Truth

Każdy moduł posiada przypisany priorytet:
- MVP – absolutne minimum do uruchomienia systemu
- v1.0 – wymagane do produkcyjnego użycia
- v2.0 – rozbudowa funkcjonalna
- nice-to-have – dodatki

Agent AI MUSI respektować te priorytety przy planowaniu i implementacji.

---

## 1. Obiekty (Objects)

Priorytet: MVP

Opis:
Centralna encja systemu. Wszystko jest przypisane do obiektu.

Funkcje:
- Tworzenie / edycja / archiwizacja obiektów
- Adres, współrzędne GPS
- Dane kontaktowe
- Typ obiektu
- Status obiektu (ACTIVE, SERVICE, CLOSED)
- Przypisane centrale alarmowe
- Przypisane rejestratory CCTV (tylko dane)

---

## 2. Centrale Alarmowe (Panels)

Priorytet: MVP

Opis:
Reprezentacja fizycznych central alarmowych.

Funkcje:
- Numer konta / ID centrali
- Producent (SATEL)
- Typ komunikacji (TCP/IP przez ETHM-1)
- Powiązanie z obiektem
- Status komunikacji
- Lista stref

---

## 3. Strefy i Linie (Zones & Lines)

Priorytet: MVP

Opis:
Logiczny podział systemu alarmowego.

Funkcje:
- Definicja stref
- Linie / wejścia
- Typ czujki
- Opisy lokalizacji
- Powiązanie z centralą

---

## 4. Alarmy i Zdarzenia (Alarms & Events)

Priorytet: MVP

Opis:
Sercem systemu są alarmy i zdarzenia.

Funkcje:
- Przyjmowanie alarmów z Satel Worker (TCP/IP) i parserów SMS
- Klasyfikacja (ALARM, TAMPER, INFO, TEMP)
- Statusy alarmu (KANONICZNE — patrz 05_ALARM_LIFECYCLE.md):
  - NEW
  - IN_PROGRESS
  - ACK
  - RESOLVED
  - CLOSED
- Przypisanie operatora
- Historia zmian statusu
- Timestampy

---

## 5. Bundling Alarmów

Priorytet: MVP

Opis:
Grupowanie wielu alarmów w jeden incydent. Kluczowe dla operatora — bez tego przy burście 200+ eventów operator widzi surowe linie zamiast incydentów.

Funkcje:
- Łączenie alarmów z tego samego obiektu
- Reguły czasowe (np. 5 min)
- Wspólny status dla grupy
- Obsługa jako jeden incydent
- Alarmy temperaturowe wymagają obowiązkowej notatki przy zamykaniu

---

## 6. Integracja SATEL (Satel Worker TCP/IP)

Priorytet: MVP

Opis:
Bezpośrednia integracja z centralami Satel przez protokół ETHM-1 (TCP/IP, port 10004).
Szczegóły protokołu: **06_INTEGRATIONS.md, sekcja 1**.

Funkcje:
- Stałe połączenie TCP/IP z modułem ETHM-1 Plus
- Polling stanów wejść, stref, wyjść, awarii
- Odbiór zdarzeń w czasie rzeczywistym
- Mapowanie kont na obiekty
- Obsługa heartbeatów i reconnect (Exponential Backoff)
- Zrzut stanu centrali do Redis (Live State Cache)
- Sterowanie centralą Arm/Disarm (v2.0 / Faza 2)

---

## 7. SMS Integration (Temperatura)

Priorytet: MVP

Opis:
Odbiór SMS z czujników temperatury (Efento Cloud, Bluelog).
Szczegóły formatów: **06_INTEGRATIONS.md, sekcja 2**.

Funkcje:
- Odczyt SMS z modemu GSM
- Parser treści (Efento i Bluelog — dwa różne formaty)
- Mapowanie numerów na zaufanych nadawców (SMS_PHONE_MAPPING)
- Mapowanie czujników na obiekty (TEMP_SENSORS)
- Generowanie eventów temperatury z flagą `requires_note`

---

## 8. Użytkownicy i Role

Priorytet: MVP

Opis:
Zarządzanie dostępem (RBAC).
Szczegóły ról: **08_SECURITY_AND_ROLES.md**.

Role (KANONICZNE):
- SYSTEM (rola techniczna — Workery, parsery)
- MASTER (właściciel — pełen dostęp)
- ADMIN (zarządzanie użytkownikami, konfiguracja)
- OPERATOR (obsługa alarmów, centrum monitorowania)
- TECHNICIAN (serwisant, tryb terenowy)
- FIELD_WORKER (audytor, testy cykliczne, zgłoszenia serwisowe)

Funkcje:
- Uprawnienia per moduł
- Aktywność użytkowników
- Timed Secret Reveal (odsłanianie haseł z logowaniem)

---

## 9. Powiadomienia

Priorytet: v1.0

Opis:
System notyfikacji wewnętrznych.

Funkcje:
- Powiadomienia w aplikacji
- Eskalacje
- Przypomnienia o niezamkniętych alarmach

---

## 10. Raporty

Priorytet: v1.0

Opis:
Raportowanie operacyjne i SLA.

Funkcje:
- Raporty alarmów
- Czas reakcji
- Czas zamknięcia
- Raporty per obiekt
- Eksport PDF / CSV
- **Stale Alarm Report:** Codzienny cron job generujący listę Bundle CRITICAL otwartych >24h. Wysyłany automatycznie do użytkowników z rolą MASTER. Parametr: `STALE_ALARM_REPORT_CRON = "0 8 * * *"` (domyślnie: codziennie o 8:00 UTC).

---

## 11. Dokumentacja Obiektu

Priorytet: v2.0

Opis:
Pliki i instrukcje dla operatorów.

Funkcje:
- Plany obiektu
- Zdjęcia
- Instrukcje postępowania
- Checklisty

---

## 12. Integracje Zewnętrzne (Future)

Priorytet: nice-to-have

Opis:
Integracje poza SATEL i SMS.

Przykłady:
- API firm ochrony
- Integracja z BMS
- Integracja z systemami ticketowymi

---

## 13. Audyt i Logi

Priorytet: v1.0

Opis:
Pełna ścieżka audytu (encja AUDIT_LOG).

Funkcje:
- Logi akcji użytkowników
- Logi systemowe
- Historia zmian konfiguracji
- Eksport logów

---

## 14. Backup & Archiwizacja

Priorytet: v1.0

Opis:
Zabezpieczenie danych.

Funkcje:
- Backup bazy danych
- Backup plików
- Retencja danych
- Archiwizacja alarmów

---

## 15. Aplikacja Mobilna — Widok Operatora Terenowego

Priorytet: v1.0

Opis:
Uproszczony widok mobilny dla operatora pracującego w terenie. Zaprojektowany pod szybki dostęp do najważniejszych informacji bez konieczności przeglądania pełnego dashboardu.

Funkcje:
- **Lista aktywnych alarmów** — tylko kluczowe informacje: obiekt, priorytet, czas, status. Bez szczegółów technicznych (raw events).
- **Szybkie akcje Arm/Disarm** — dla obiektów oznaczonych jako ulubione. Jedno kliknięcie > komenda do Satel Worker.
- **Widok Ulubione** — lista obiektów oznaczonych gwiazdką przez operatora. Szybki dostęp z głównego ekranu.
- **Widok Ostatnio używane** — lista ostatnio odwiedzanych obiektów (max 10-20 pozycji).
- Offline: podgląd ostatniego znanego stanu obiektów (cache SQLite).

Uwagi:
- Dane ulubionych i ostatnio używanych przechowywane w Redis (backend) + SQLite (frontend offline).
- Patrz również: moduł 17 (Ulubione i Ostatnio używane — wspólny dla desktop i mobile).

---

## 16. Harmonogram Uzbrajania / Rozbrajania (Auto-Arm)

Priorytet: v1.0

Opis:
Automatyczne uzbrajanie/rozbrajanie obiektów wg zdefiniowanego harmonogramu.
Przykład: przychodnie uzbrajane automatycznie o 24:00, rozbrajane o 6:00.

Funkcje:
- **Konfiguracja per obiekt:**
  - Dzień tygodnia (lub codziennie)
  - Godzina
  - Typ uzbrojenia: `FULL` / `STAY` / `AWAY`
  - Strefa (partition) — opcjonalnie, domyślnie wszystkie
- **Wykonawca:** Dedykowany cron job w backendzie > komenda Arm/Disarm do Satel Worker przez tabelę `satel_commands` + `outbox` + RabbitMQ.
- **Logowanie:** Każde automatyczne uzbrojenie/rozbrojenie zapisywane w `AUDIT_LOG` z adnotacją `source: "AUTO_ARM"`.
- **Obsługa błędów:**
  - Brak łączności z centralą > system generuje alarm `WARNING` (Auto-arm failed: brak komunikacji z centralą).
  - Komenda `NACK` > retry 2x z 30s delay, potem > alarm `WARNING` + log.
- **UI:** Panel konfiguracji harmonogramów w ustawieniach obiektu (dostępny dla ADMIN/MASTER).

### Conflict Resolution (Manual > Automatic)

**Reguła nadrzędna:** Manualna komenda operatora ZAWSZE ma priorytet nad automatycznym harmonogramem.

**Mechanizm cooldown:** Jeśli operator ręcznie rozbroił obiekt, Auto-Arm pomija ten obiekt przez czas cooldown. Zapobiega sytuacji: technik rozbreja centralę o 23:59, cron uzbraja z powrotem o 24:00.

**Łańcuch decyzyjny (6 kroków):**

```mermaid
flowchart TD
    A["Cron: Auto-Arm trigger dla Obiektu X"] --> B{"Obiekt w trybie SERVICE/CLOSED?"}
    B -- TAK --> C["SKIP — log: obiekt w trybie serwisowym"]
    B -- NIE --> D{"Aktywny manualny DISARM w oknie cooldown?"}
    D -- TAK --> E["SKIP — log: manual override active"]
    D -- NIE --> D2{"Aktywne okno serwisowe DLOAD?"}
    D2 -- TAK --> E2["SKIP — log: DLOAD_SERVICE_WINDOW"]
    D2 -- NIE --> F{"Istnieje PENDING/SENT komenda dla tej centrali?"}
    F -- TAK --> G["SKIP — log: komenda już w trakcie"]
    F -- NIE --> H{"Status połączenia z centralą?"}
    H -- DISCONNECTED/UNKNOWN --> I["Generuj alarm WARNING: Auto-arm failed"]
    H -- CONNECTED --> J["Wyślij komendę ARM przez satel_commands + outbox"]
    J --> K{"ACK w ramach timeout?"}
    K -- ACK --> L["Sukces — AUDIT_LOG source: AUTO_ARM"]
    K -- NACK/TIMEOUT --> M{"Wyczerpano próby?"}
    M -- NIE --> J
    M -- TAK --> I
```

> [!NOTE]
> **Krok 3 (DLOAD Service Window Check):** Sprawdza czy istnieje aktywne okno serwisowe Connection Release. Jeśli Worker jest rozłączony z centralą z powodu sesji DLOAD, Auto-Arm NIE inkrementuje licznika błędów i NIE generuje alarmu WARNING. SQL: `SELECT 1 FROM audit_log WHERE action = 'PANEL_CONNECTION_RELEASED' AND panel_id = :panel_id AND reconnect_at > NOW()`.

**Sprawdzenie cooldown (SQL):**
```sql
SELECT EXISTS (
  SELECT 1 FROM satel_commands
  WHERE panel_id = :panel_id
    AND command_type = 'DISARM'
    AND source = 'MANUAL'
    AND status = 'ACK'
    AND created_at > now() - interval ':cooldown minutes'
) AS manual_override_active;
```

### Parametry konfiguracyjne (Auto-Arm)

| Parametr | Domyślna | Lokalizacja | Opis |
|---|---|---|---|
| `AUTO_ARM_COOLDOWN_MINUTES` | 60 | Backend .env | Czas cooldown po manualnym disarm |
| `AUTO_ARM_RETRY_COUNT` | 2 | Backend .env | Ilość retry’ów na NACK (total = retry + 1) |
| `AUTO_ARM_RETRY_DELAY_SECONDS` | 30 | Backend .env | Opóźnienie między retryami |

Powiązania:
- Sterowanie centralą: **06_INTEGRATIONS.md, sekcja 1.3**
- Komendy: **14_SATEL_COMMANDS.md, sekcja 2**
- Command tracking: **04_DATA_MODEL_ERD.md, sekcja 7b** (tabela `satel_commands`)
- Audit: **08_SECURITY_AND_ROLES.md, sekcja 3**

---

## 17. Ulubione i Ostatnio Używane (UI)

Priorytet: v1.0

Opis:
Funkcjonalność wspólna dla interfejsu desktop i mobile, poprawiająca UX przy częstym przełączaniu się między obiektami.

Funkcje:
- **Ulubione (gwiazdka):**
  - Operator oznacza obiekt jako ulubiony (ikona gwiazdki).
  - Lista ulubionych dostępna z głównego widoku / sidebar.
  - Dane przechowywane per użytkownik w backendzie.
- **Ostatnio używane:**
  - System automatycznie śledzi ostatnio odwiedzane obiekty (max 20 pozycji).
  - Lista dostępna z głównego widoku / quick-access panel.
  - Dane przechowywane w Redis (backend, key: `user:{id}:recent_objects`, TTL: 30 dni).
  - Alternatywnie: cache lokalny (SQLite) na frontendzie.
- **Desktop:** Widget w sidebar lub top-bar.
- **Mobile:** Dedykowany tab lub sekcja na ekranie głównym.

---

## 18. Zgłoszenia Serwisowe (Service Tickets)

Priorytet: v1.0

Opis:
System zgłoszeń serwisowych umożliwiający operatorom tworzenie ticketów z poziomu alarmu lub ręcznie.
Usprawnienie przepływu pracy: operator > ticket > technik.

Funkcje:
- **Tworzenie zgłoszenia:**
  - Z poziomu alarmu (przycisk Utwórz zgłoszenie serwisowe) — automatyczne powiązanie z alarmem.
  - Ręcznie z poziomu obiektu.
- **Dane zgłoszenia:**
  - Obiekt (wymagane)
  - Typ usterki (z predefiniowanej listy + opis tekstowy)
  - Opis problemu
  - Priorytet: `LOW` / `MEDIUM` / `HIGH` / `CRITICAL`
  - Data zgłoszenia (automatyczna)
  - Powiązany alarm (`bundle_id`, opcjonalne)
- **Statusy:** `OPEN` > `IN_PROGRESS` > `CLOSED`
- **Powiadamianie technika:**
  - Email na zdefiniowaną skrzynkę (np. serwis@firma.pl) — konfiguracja globalna lub per obiekt.
  - Opcjonalnie: integracja z zewnętrznym systemem ticketowym (Jira, OSTicket) — v2.0.
- **Śledzenie:** Lista zgłoszeń z filtrami (status, obiekt, priorytet, data).
- **Uprawnienia:** Rola `TECHNICIAN` ma dostęp do przeglądania i aktualizacji statusu zgłoszeń (patrz **08_SECURITY_AND_ROLES.md**).

Powiązania:
- Role: **08_SECURITY_AND_ROLES.md** — uprawnienia `ticket:read`, `ticket:write`
- Alarmy: **05_ALARM_LIFECYCLE.md** — powiązanie ticket > bundle

---

## 19. UI Resilience & Degraded Mode (Flutter)

Priorytet: v1.0

Opis:
Specyfikacja zachowań UI podczas awarii, degradacji i niestabilności połączenia. Operator MUSI zawsze wiedzieć, w jakim stanie jest system.

### 19.1 Tryb Degraded Mode (UI-01)

| Stan systemu | UI zachowanie | Baner |
|---|---|---|
| **Online (normal)** | Pełna funkcjonalność | Brak banneru |
| **WS disconnected, REST OK** | Dane odświeżane przez polling (co 10s), brak push alarmów | 🟡 "Utracono połączenie real-time. Alarmy odświeżane co 10s." |
| **REST timeout/error** | Ostatnie dane z cache SQLite, formularz read-only | 🔴 "Brak połączenia z serwerem. Dane mogą być nieaktualne." |
| **Pełny offline** | Tylko cache SQLite, brak nowych danych | 🔴 "Tryb offline. Akcje zostaną zsynchronizowane po przywróceniu połączenia." |

**Zasada:** Baner widoczny na KAŻDYM ekranie (top bar), nie tylko na dashboard.

### 19.2 Skeleton / Loading States (UI-02)

Każdy widok MUSI mieć stan skeleton (szkielet ładowania) zamiast pustego ekranu lub spinnera:

| Ekran | Skeleton |
|---|---|
| Lista alarmów | 5 prostokątów (shimmer animation) w kształcie alarm card |
| Szczegóły obiektu | Header placeholder + 3 sekcje shimmer |
| Mapa | Szary placeholder z napisem "Ładowanie mapy..." |
| Raport | Shimmer table (nagłówki + 10 wierszy) |

**Zasady:**
- Skeleton wyświetlany max 5s — po tym czas wyświetl komunikat "Serwer nie odpowiada" z przyciskiem "Ponów próbę".
- Po załadowaniu danych, płynne przejście (fade) z skeleton do treści (200ms).

### 19.3 Obsługa Konfliktu 409 (UI-03)

> **Scenariusz:** Operator edytuje alarm, a inny operator zmienił go w międzyczasie. Serwer zwraca `409 ALARM_STALE_VERSION`.

**Wymagane zachowanie:**
1. **NIE czyść formularza** — dane operatora pozostają w polach.
2. Pokaż dialog: "Alarm został zmodyfikowany przez {modified_by}. Odświeżyć dane?"
3. Dwie opcje:
   - **"Odśwież"** → pobierz nowe dane z serwera, zachowaj zmiany operatora w draft.
   - **"Nadpisz"** → wyślij ponownie z nowym `version` (force update).
4. Toast: "Dane odświeżone. Sprawdź zmiany i zapisz ponownie."

**Implementacja:** Flutter `BLoC` trzyma `draftState` (zmiany operatora) i `serverState` (dane z serwera). Refresh aktualizuje `serverState`, `draftState` pozostaje.

### 19.4 Offline Intent Queue (UI-04)

> [!CAUTION]
> **Tryb offline = READ-ONLY.** Przyciski mutacji (Claim, Ack, Resolve, Close) są **wyłączone** w trybie offline. Zapobiega to masywnym konfliktom wersji (409) przy synchronizacji.

> **Scenariusz:** Operator jest w terenie (telefon offline), widzi alarmy z cache SQLite.

**Mechanizm Intent Queue:**
1. Operator może zapisać **lekkie intencje** ("zamierzam obsłużyć ten alarm") — NIE pełne mutacje stanu.
2. Intencja zapisywana w SQLite: `offline_intents(id, intent_type, bundle_id, created_at, status)`.
3. UI wyświetla: "Intencja zakolejkowana — zostanie przesłana po przywróceniu połączenia. ⏳"
4. Po odzyskaniu połączenia (WS reconnect):
   - System wysyła batch intencji: `POST /api/sync/intents`
   - Backend dla każdej intencji: pobierz aktualny stan → jeśli alarm nadal dostępny (np. status NEW) → wykonaj akcję z bieżącą `version`.
   - Jeśli alarm już zajęty przez innego operatora → `INTENT_REJECTED` z informacją o bieżącym operatorze.
   - Intencje starsze niż 30 min → oznaczone jako `EXPIRED` (nie wysyłane).
5. Po sync: dialog podsumowania: "✅ 2/3 intencji zaakceptowane, ❌ 1 odrzucona (alarm obsługiwany przez Anna Nowak)".
6. Badge na ikonie: "2 oczekujące intencje".

**Obsługiwane intencje offline:**
- INTENT_CLAIM — zamiar obsłużenia alarmu
- INTENT_NOTE — dodanie notatki
- INTENT_SERVICE_TICKET — tworzenie zgłoszenia serwisowego

**NIE obsługiwane offline** (wymagają natychmiastowej odpowiedzi):
- Arm/Disarm (wymaga komunikacji z centralą)
- Secret Reveal
- Zmiana statusu alarmu (Ack, Resolve, Close) — zbyt wysokie ryzyko konfliktu

### 19.5 Data Staleness Indicator (UI-05)

> **Cel:** Operator MUSI wiedzieć, czy patrzy na aktualne dane.

| Kontekst | Trigger | Wskaźnik |
|---|---|---|
| **Lista alarmów** | `last_refresh > 30s` bez WS heartbeat | Zegar: "Dane sprzed {Xs}" + przycisk 🔄 |
| **Stan centrali** | `panel_redis_ttl < 60s` lub `status = UNKNOWN` | ⚠️ "Stan panelu nieznany — brak danych od {X}s" |
| **Raporty** | `X-Data-Freshness: stale` header od API | 🟡 Baner: "Dane mogą być opóźnione o {lag}s (replika)" |
| **Obiekt** | `version` z cache ≠ `version` z serwera | 🔄 "Dostępna nowsza wersja danych" + auto-refresh |

### 19.6 WebSocket Reconnect Strategy (UI-06)

**Algorytm:** Exponential backoff BEZ limitu prób.

```
Initial delay: 3s
Multiply: x2
Max delay cap: 30s

Sequence: 3s → 6s → 12s → 24s → 30s → 30s → 30s → ...
```

**Na każdym reconnect:**
1. Pobierz nowy ticket: `POST /api/auth/ws-ticket`
2. Jeśli ticket failed (401) → redirect na ekran logowania.
3. Jeśli ticket OK → connect `wss://host/api/ws?ticket={ticket}`
4. Po connect → wyślij `replay_request` z ostatnim `sequence_id`
5. Jeśli serwer zwraca `replay_overflow` → przeładuj dane z REST API.

**UI podczas reconnect:**
- Pasek: "Łączenie z serwerem... (próba {N})"
- Po 60s bez połączenia: "Nie udaje się połączyć. Sprawdź połączenie sieciowe."
- Po reconnect: "✅ Połączono. Synchronizacja danych..." → auto-refresh aktywnego widoku.

---

## Zasady dla Agenta AI

- Nie implementuj modułów v2.0 i nice-to-have w MVP
- Nie mieszaj odpowiedzialności modułów
- Każdy moduł to osobny kontekst logiczny
- Obiekt = główny agregat domenowy
