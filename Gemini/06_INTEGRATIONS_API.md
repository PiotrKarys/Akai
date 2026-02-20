# 06_INTEGRATIONS.md

## Cel
Opis techniczny integracji z systemami zewnętrznymi.

---

## 1. SATEL (ETHM-1 Plus Integration)

### 1.1 Protokół i Komunikacja
Integracja odbywa się bezpośrednio po protokole TCP/IP (port 10004) z modułem ETHM-1 Plus.
- **Protokół:** Oficjalny "Protokół integracji ETHM-1" (ramki binarne, start 0xFE 0xFE, CRC-16).
- **Tryb:** Klient TCP (Worker łączy się do modułu).
- **Ograniczenie:** Moduł pozwala na max 1 połączenie. Worker musi utrzymywać sesję (Keep-Alive) i wznawiać ją po zerwaniu.

### 1.2 Zakres Danych (MVP)
Worker odpytuje centralę (Polling) lub nasłuchuje zdarzeń:
1. **Stan Wejść (Zones):** Naruszenie, Sabotaż, Alarm, Awaria.
2. **Stan Stref (Partitions):** Uzbrojona, Rozbrojona, Alarm, Czas na wyjście.
3. **Stan Wyjść:** Załączone/Wyłączone (np. sterowanie bramą).
4. **Awarie Systemowe:** Brak zasilania 230V, Akumulator, Awaria linii tel.

### 1.3 Sterowanie (v2.0 / Faza 2)
Dzięki stałemu połączeniu TCP możliwy jest command flow:
1. Operator klika "Uzbrój" w App.
2. Backend wysyła komendę do RabbitMQ (`cmd_queue`).
3. Worker odbiera komendę, formatuje ramkę binarną (z kodem użytkownika).
4. Worker wysyła ramkę do ETHM-1.
5. Worker odbiera potwierdzenie (ACK/NACK) i aktualizuje status w Redis/App.

### 1.4 Zdarzenia Nieautoryzowanego Dostepu

Centrala SATEL generuje specjalne zdarzenia bezpieczenstwa, ktore Satel Worker musi odbierac i mapowac na typy systemowe:

| Zdarzenie SATEL | Event Code | Typ w systemie | Priorytet | Opis |
|---|---|---|---|---|
| 3 bledne hasla | Zone event (tamper context) | `UNAUTHORIZED_ACCESS` | WARNING/CRITICAL | 3 nieudane proby wprowadzenia kodu na manipulatorze/czytniku |
| Nieznany kod dostepu | Zone event (access context) | `UNAUTHORIZED_ACCESS` | WARNING/CRITICAL | Proba uzycia niezarejestrowanego kodu |

**Notatki:**
- Priorytet (WARNING vs CRITICAL) konfigurowalny per obiekt.
- **Two-Phase Priority:** Worker ustawia `default_priority` na podstawie kodu zdarzenia. Backend wzbogaca koncowe `priority` konsultujac `ZONES.priority_override`. Szczegoly: **13_EVENT_SCHEMAS.md, sekcja 2**.
- Zdarzenia te tworza Bundle Alarm z flaga `requires_note = true` (patrz **05_ALARM_LIFECYCLE.md, sekcja 6.1**).
- Dedup key: `{panel_id}:UNAUTHORIZED_ACCESS:{timestamp_minute}`

### 1.5 Handshake i Autoryzacja ETHM-1 (SAT-02)

Połączenie TCP z modułem ETHM-1 Plus wymaga sekwencji inicjalizacyjnej:

```
1. Worker → TCP connect do ETHM-1 (port 10004)
2. ETHM-1 ← odpowiada ramką identyfikacyjną (0xFE 0xFE + device info)
3. Worker → wysyła komendę autoryzacji (jeśli skonfigurowany hasłem integracji)
   → Ramka: 0xFE 0xFE + cmd 0x00 + integration_password (6 znaków)
4. ETHM-1 ← ACK (0xEF) lub NACK (0xED)
5. Worker → po ACK rozpoczyna polling (Full State Dump, patrz sekcja 1.2)
```

**Hasło integracji:**
- Ustawiane w DLOAD lub na manipulatorze centrali.
- Przechowywane jako Docker Secret: `satel_integration_password`.
- Per panel — każdy panel może mieć inne hasło.

**Timeout handshake:** 5 sekund. Brak odpowiedzi → reconnect z exponential backoff.

### 1.6 Kompatybilność Firmware ETHM-1 (SAT-03)

| Firmware ETHM-1 | Wspierane | Testowane | Uwagi |
|---|---|---|---|
| **v1.06+** | ✅ TAK | ✅ | Minimalna wersja — pełny protokół integracji |
| v1.04 - v1.05 | ⚠️ Częściowe | ❌ | Brak komendy odczytu nazw stref (0x04). Polling działa, ale nazwy stref = "Strefa {N}" |
| < v1.04 | ❌ NIE | ❌ | Niekompatybilny format ramek |

**Centrala INTEGRA:**

| Model | Wspierane | Testowane | Uwagi |
|---|---|---|---|
| INTEGRA 128 | ✅ TAK | ✅ | Pełen zakres: 128 wejść, 32 strefy |
| INTEGRA 64 | ✅ TAK | ❌ | 64 wejścia, reszta jak 128 |
| INTEGRA 32 | ✅ TAK | ❌ | 32 wejścia |
| INTEGRA 24 | ⚠️ Prawdopodobnie | ❌ | Wymaga weryfikacji |
| VERSA | ❌ NIE | ❌ | Inny protokół (ETHM-1 nie obsługuje) |

**Wymaganie:** Przy onboarding nowego obiektu, Technician MUSI zweryfikować wersję firmware ETHM-1 i odnotować ją w danych panelu.

### 1.7 DLOAD — Współdzielenie Połączenia (SAT-04)

**Problem:** ETHM-1 pozwala na max 1 połączenie TCP. Gdy serwisant chce użyć DLOAD (oprogramowanie konfiguracyjne SATEL), musi rozłączyć Worker.

**Rozwiązanie:** Endpoint API do kontrolowanego zwolnienia połączenia.

```
POST /api/satel/connection-release
{
  "panel_id": "PAT001",
  "reason": "DLOAD service session",
  "duration_minutes": 30,
  "requested_by": "user_id"
}

Response:
{
  "status": "released",
  "reconnect_at": "2026-02-16T15:00:00Z",
  "warning": "Worker nie będzie monitorował panelu PAT001 przez 30 min"
}
```

**Logika Workera:**
1. Otrzymuje sygnał "release" (przez Redis pub/sub).
2. Zamyka połączenie TCP z ETHM-1 dla danego panelu.
3. **NIE próbuje reconnect** przez `duration_minutes`.
4. Po upływie czasu — automatyczny reconnect + Full State Dump.
5. Event: `PANEL_CONNECTION_RELEASED` → `AUDIT_LOG`.

**Ograniczenia:**
- Max 60 min. Po 60 min Worker reconnect nawet bez jawnego sygnału.
- Tylko rola ADMIN/MASTER/TECHNICIAN może wywołać release.

### 1.8 Symulator ETHM-1 (SAT-01)

**Cel:** Umożliwić rozwój i testy Satel Worker bez fizycznego hardware.

| Element | Wartość |
|---|---|
| Typ | TCP server (Python, mock) |
| Port | 10004 (identyczny jak ETHM-1) |
| Protokół | Pełna emulacja ramek binarnych ETHM-1 |
| Scenariusze | Normalny polling, alarm (zone violation), tamper, auto-arm response, disconnect |
| CI/CD | Symulator startuje jako kontener w pipeline testowym |
| Tryby | `NORMAL` (odpowiada poprawnie), `FLAKY` (traci pakiety 10%), `TIMEOUT` (brak odpowiedzi co 5 ramkę) |

**Minimalny zakres emulacji:**
- Komendy 0x00-0x06 (odczyt stref, wejść, wyjść, awarii)
- ACK/NACK na komendy sterowania (0x80-0x84)
- Heartbeat (0x7F) — odpowiedź w < 500ms
- Scenariusz: wejście 1 w violation → Worker generuje event → test weryfikuje event w RabbitMQ

---

## 2. SMS (Efento / Bluelog)

### 2.1 Hardware
- **Modem GSM (USB/Serial):** Fizycznie podłączony do serwera.
- **Narzędzie:** `gammu-smsd` lub dedykowany skrypt Python (`pyserial`).
- Numer SIM dedykowany wyłącznie do odbioru alertów.

### 2.2 Zaufani Nadawcy i Bezpieczeństwo
System reaguje **TYLKO** na SMSy z 2 znanych numerów:
- **Numer 1:** Efento Cloud (alarmy temperatury lodówek/zamrażarek).
- **Numer 2:** Bluelog (alarmy temperatury chłodni/spedycji).

SMSy z innych numerów → **ignorowane** + zapis w logu systemowym.

- Mapowanie numerów przechowywane w tabeli `SMS_PHONE_MAPPING`.
- Niezidentyfikowane numery → log WARNING, ignore event.
- Raw SMS body **nie logowany** w produkcji (potencjalnie zawiera dane osobowe), jedynie hash MD5 treści.

### 2.3 Format SMS (Efento Cloud)

**Alarm (początek):**
```
2026-02-10 11:03:00 Alarm! Regula Leg_szczep_prawa_MIN, czujnik: Leg_szczep_prawa w Swiat Zdrowia Operat - Leg_Szczep, wartosc 1.7C
```

**Parsowanie:**
| Pole | Wartość z przykładu |
|---|---|
| Timestamp | `2026-02-10 11:03:00` |
| Typ | `Alarm!` → CRITICAL |
| Reguła | `Leg_szczep_prawa_MIN` |
| Czujnik | `Leg_szczep_prawa` → mapuj na `TEMP_SENSORS.name` |
| Lokalizacja | `Swiat Zdrowia Operat - Leg_Szczep` → mapuj na `OBJECTS.name` |
| Wartość | `1.7C` |

**Powrót do normy:**
```
2026-02-10 11:24:00 Powrot do normalnego stanu. Regula Leg_szczep_prawa_MIN, czujnik Leg_szczep_prawa w Swiat Zdrowia Operat - Leg_Szczep: Wartosc 2.0C
```
- Typ: `Powrot do normalnego stanu` → INFO (zamyka otwarty alarm temp.).

### 2.3a Jakość parsowania SMS (SIL-05)

> **Problem:** SMS może dotrzeć ucięty (160 znaków), zgarblowany (kodowanie) lub niekompletny. Parser powinien oznaczyć jakość parsowania aby operator wiedział czy może ufać danym.

**Pole `sms_quality` w event payload:**

| Wartość | Opis | Akcja |
|---|---|---|
| `complete` | SMS w pełni sparsowany, wszystkie pola wypełnione | Normalny flow |
| `truncated` | SMS ucięty (> 160 znaków brak terminatora), ale kluczowe dane obecne | Alarm tworzony, ale z flagą "dane mogą być niekompletne" |
| `garbled` | Kodowanie uszkodzone, parser wyciągnął co mógł | Alarm tworzony z priorytetem WARNING, wymaga ręcznej weryfikacji |
| `unparseable` | Parser nie rozpoznał formatu | Brak alarmu, log WARNING, raw SMS zapisany do AUDIT_LOG |

**Implementacja:** Parser ustawia `sms_quality` w event payload → Worker propaguje do `EVENTS.details` → UI wyświetla ikonę jakości.

Przykład oryginalnego SMS:
```
(Alertt) Gad Spedycja (S1, 21040DD5): Leg_szczep_prawa, -4.0°C
```

| Pole | Wartość |
|---|---|
| Lokalizacja | `Gad Spedycja` |
| Czujnik | `S1` / `21040DD5` |
| Pomiar | `Leg_szczep_prawa` |
| Temp | `-4.0°C` |
| Typ | `Alert temp.` → CRITICAL |

**Koniec alertu:**
```
Koniec alertu temp. dla rejestratora Gad Spedycja (S1, 21040DD5)
```
- Typ: `Koniec alertu` → INFO (zamyka otwarty alarm temp.).

### 2.5 Przepływ SMS → RabbitMQ
1. Modem odbiera SMS.
2. Skrypt sprawdza nadawcę → czy w `SMS_PHONE_MAPPING`?
3. Jeśli TAK → parser (Efento lub Bluelog) wyciąga dane + ustawia `sms_quality`.
4. Mapowanie czujnika → Obiekt (przez `TEMP_SENSORS`).
5. **PII Isolation:** SMS Agent hashuje pole `raw_sms` (SHA-256) → `raw_sms_hash`. Oryginalny tekst SMS zapisywany do tabeli `sms_raw_archive` (dostęp: MASTER/SYSTEM). Pole `raw_sms` **NIGDY** nie trafia do RabbitMQ ani `EVENTS.details`.
6. Tworzenie Raw Event (z polem `sms_quality` i `raw_sms_hash`) → RabbitMQ.
7. Worker tworzy / aktualizuje Bundle Alarm z flagą `requires_note = true`.

### 2.6 Fallback (Future)
- Bramka SMS online (API) w przypadku awarii modemu.

---

## 3. Mapy (Google Maps / OpenStreetMap)

### 3.1 Wyświetlanie
- Widget mapy we Flutterze.
- Markery (Pinezki) dla każdego Obiektu.
- Kolor markera zależny od statusu (Zielony = OK, Czerwony = Alarm).

### 3.2 Geokodowanie
- Przy dodawaniu obiektu, adres zamieniany jest na Lat/Lon.
- Możliwość ręcznej korekty pozycji pinezki (Drag & Drop).

---

## 4. CCTV (Tylko Metadane)

**NON-GOAL:** Nie robimy streamingu wideo w aplikacji.

### Zakres integracji:
- Przechowywanie danych w bazie:
  - Model rejestratora
  - Adres IP (LAN/WAN)
  - Login / Hasło (zaszyfrowane, Timed Reveal)
  - Ilość kanałów
- Przycisk "Kopiuj hasło" / "Otwórz w przeglądarce" (uruchamia zewnętrzną przeglądarkę).


---
---


# 10_API_HIGH_LEVEL

## Cel
High-level specyfikacja API (kontrakt dla Frontendu i Backendu).
Szczegółowa specyfikacja powstanie w formacie **OpenAPI 3.1** (Swagger) generowanym automatycznie z kodu FastAPI (`GET /api/docs`).

Konwencja: wszystkie endpointy mają prefix `/api/v1/` (wersjonowanie URL-based).

> **Uwaga o wersjonowaniu:** Prefix `/api/v1/` gwarantuje kompatybilność wsteczną. Wprowadzenie breaking change = nowa wersja `/api/v2/`, stara wspierana min. 6 miesięcy.

---

## 0. Health Check & Infrastructure

### `GET /healthz`
- **Auth:** Brak (publiczny).
- **Cel:** Docker HEALTHCHECK / Kubernetes liveness probe.
- **Response 200:** `{ "status": "healthy", "uptime_seconds": 3600 }`
- **Szczegóły:** `09_HA_RTO_RPO.md`, sekcja HA-03.

### `GET /readyz`
- **Auth:** Brak (publiczny).
- **Cel:** Reverse proxy routing (gotowość serwisu).
- **Response 200:** `{ "status": "ready", "checks": { "postgresql": {...}, "redis": {...}, "rabbitmq": {...} } }`
- **Response 503:** Serwis niegotowy (przynajmniej jeden check failed).

---

## 1. Authentication (Auth)

### `POST /api/auth/login`
- **Request:** `{ "email": "...", "password": "..." }`
- **Response:** `{ "access_token": "JWT...", "user": { "role": "ADMIN", ... } }`

### `POST /api/auth/refresh`
- Obsługa odświeżania tokena.

### `POST /api/auth/ws-ticket`
- **Auth:** Bearer JWT (wymagany).
- **Cel:** Wygeneruj jednorazowy bilet do nawiązania połączenia WebSocket.
- **Response:** `{ "ticket": "abc123", "expires_in": 10 }`
- **Szczegóły:** `08_SECURITY_AND_ROLES.md`, sekcja 4.3a (ticket-based WS auth).
- **Rate limit:** 10 req/min per user (ochrona przed flooding).

---

## 2. Objects (Obiekty)

### `GET /api/objects`
- **Query Params:** `?query=warszawa&status=ACTIVE`
- **Response:** Lista obiektów (skrócona).

### `GET /api/objects/{id}`
- **Response:** Pełne dane obiektu + lista central + ostatnie alarmy.

### `POST /api/objects`
- Dodawanie nowego obiektu.
- **Response:** `{ "id": "...", "version": 1 }`

### `PATCH /api/objects/{id}`
- Edycja obiektu (częściowa aktualizacja).
- **Request:** `{ "name": "Nowa nazwa", "address": "...", "version": 3 }` — pole `version` **wymagane**.
- **Response:** `{ "id": "...", "name": "...", "version": 4 }`
- **Conflict:** Jeśli `version` nie zgadza się → `409 OBJECT_STALE_VERSION`.

## 3. Alarms (Alarmy / Bundle)

### `GET /api/alarms`
- **Query Params:** `?status=NEW,IN_PROGRESS&priority=CRITICAL&after={cursor}&limit=20`
- **Paginacja:** Cursor-based (Keyset Pagination). Patrz sekcja 9.
- **Response:** Lista Bundli (incydentów) z cursorem do następnej strony.

### `POST /api/alarms`
- **Request:** `{ "object_id": "...", "priority": "WARNING", "type": "MAINTENANCE", "description": "Uszkodzona czujka w strefie 3" }`
- **Opis:** Manualne zgłoszenie alarmu/incydentu przez Operatora, Technika lub Audytora.
- **Response:** `{ "bundle_id": "..." }`

### `GET /api/alarms/{id}`
- **Response:** Szczegóły Bundle + lista Raw Events w środku.

### `POST /api/alarms/{id}/claim`
- **Request:** `{ "version": 1 }`
- Operator przypisuje się do alarmu.
- **Zmiana statusu:** NEW → CLAIMING → IN_PROGRESS (w jednej transakcji).
- **Response:** `{ "bundle_id": "...", "status": "IN_PROGRESS", "version": 2, "assigned_to": "Jan Kowalski" }`
- **Conflict:** Jeśli `version` nie zgadza się → `409 ALARM_STALE_VERSION`. Jeśli już claimed → `409 ALARM_ALREADY_CLAIMED`.

### `POST /api/alarms/{id}/ack`
- **Request:** `{ "note": "Fałszywy alarm, serwis w drodze", "version": 2 }`
- **Zmiana statusu:** IN_PROGRESS → ACK.
- **Response:** `{ "bundle_id": "...", "status": "ACK", "version": 3 }`
- Uwaga: `userId` pobierany z tokena JWT (nie z body).

### `POST /api/alarms/{id}/resolve`
- **Request:** `{ "note": "Problem rozwiązany, technik potwierdził", "version": 3 }`
- **Zmiana statusu:** ACK → RESOLVED.
- **Response:** `{ "bundle_id": "...", "status": "RESOLVED", "version": 4 }`

### `POST /api/alarms/{id}/close`
- **Request:** `{ "note": "...", "version": 4 }` — notatka obowiązkowa jeśli alarm temperaturowy (`requires_note = true`).
- **Zmiana statusu:** RESOLVED → CLOSED.
- **Response:** `{ "bundle_id": "...", "status": "CLOSED", "version": 5 }`

---

## 4. Documentation (Pliki)

### `POST /api/objects/{id}/files`
- Upload zdjęcia/planu (Multipart).

### `GET /api/files/{id}/download`
- Pobranie pliku.

---

## 5. Integrations (Internal)

> **Outbox Pattern:** Endpointy integracyjne nie publikują bezpośrednio do RabbitMQ. Zamiast tego zapisują event w tabeli `outbox` w tej samej transakcji co zapis do `EVENTS`/`BUNDLE_ALARMS`. Dedykowany relay (co 100ms) publikuje wiadomości do RabbitMQ. Szczegóły: **04_DATA_MODEL_ERD.md, sekcja 7a**.

### `POST /api/integrations/satel/events`
- Endpoint dla Satel Worker (Internal Only).
- **Request:** Struktura zdarzenia z centrali (event code, timestamp, panel ID, zone ID).

### `POST /api/integrations/sms/incoming`
- Endpoint dla daemona modemu SMS.
- **Request:** `{ "sender": "+48...", "text": "...", "timestamp": "..." }`

---

## 5b. Sync Intents (Offline Reconciliation)

### `POST /api/sync/intents`
- **Auth:** Bearer JWT (wymagany).
- **Cel:** Synchronizacja intencji zapisanych offline po odzyskaniu połączenia.
- **Request:**
  ```json
  {
    "intents": [
      { "intent_type": "INTENT_CLAIM", "bundle_id": "ba_001", "offline_at": "2026-02-19T14:00:00Z" },
      { "intent_type": "INTENT_NOTE", "bundle_id": "ba_002", "note": "Sprawdzono na miejscu", "offline_at": "2026-02-19T14:05:00Z" }
    ]
  }
  ```
- **Response 200:**
  ```json
  {
    "results": [
      { "bundle_id": "ba_001", "status": "ACCEPTED", "new_status": "IN_PROGRESS", "version": 2 },
      { "bundle_id": "ba_002", "status": "REJECTED", "reason": "ALARM_ALREADY_CLAIMED", "assigned_to": "Anna Nowak" }
    ]
  }
  ```
- **Logika:** Backend pobiera bieżący stan alarmu, jeśli intencja jest możliwa do wykonania (np. alarm nadal NEW) → wykonuje ją z aktualną `version`. Intencje starsze niż 30 min → `EXPIRED`.
- **Szczegóły offline:** **03_FUNCTIONAL_MODULES.md, sekcja 19.4**.

---

## 5a. Satel Commands (Sterowanie Centralą — v1.0/v2.0)

> **Priorytet:** v1.0 (Auto-Arm), v2.0 (sterowanie manualne pełne).
> Wszystkie komendy przechodzą przez tabelę `satel_commands` + `outbox` → RabbitMQ → Satel Worker.

### `POST /api/commands/satel`
- **Request:** `{ "panel_id": "...", "command_type": "ARM_STAY", "partition_id": "part_01", "user_code": "(encrypted)" }`
- **Response:** `202 Accepted` — komenda zakolejkowana, NIE wykonana natychmiast.
- **Response Body:** `{ "command_id": "uuid", "status": "PENDING", "panel_id": "..." }`
- Komenda zapisywana w `satel_commands` (status: PENDING) + `outbox` (relay opublikuje do RabbitMQ).
- Frontend powinien pollować status komendy lub otrzymać update przez WebSocket.

### `GET /api/commands/satel/{command_id}`
- **Response:** `{ "command_id": "...", "status": "ACK", "executed_at": "...", "error_message": null }`
- Statusy: `PENDING` / `SENT` / `EXECUTING` / `ACK` / `NACK` / `TIMEOUT` / `CANCELLED`.

### `DELETE /api/commands/satel/{command_id}`
- Anulowanie komendy (tylko jeśli status = PENDING, przed wysłaniem do RabbitMQ).
- **Zmiana statusu:** PENDING → CANCELLED.
- Jeśli status ≠ PENDING → `409 COMMAND_ALREADY_SENT`.

### `POST /api/satel/connection-release`
- **Auth:** ADMIN, MASTER, TECHNICIAN.
- **Cel:** Tymczasowe zwolnienie połączenia TCP z panelem (dla sesji DLOAD).
- **Request:** `{ "panel_id": "PAT001", "reason": "DLOAD service session", "duration_minutes": 30 }`
- **Response 200:** `{ "status": "released", "reconnect_at": "2026-02-16T15:00:00Z" }`
- **Response 409:** `{ "error": { "code": "PANEL_ALREADY_RELEASED" } }`
- **Ograniczenia:** Max 60 min. Szczegóły: `06_INTEGRATIONS.md`, sekcja 1.7.
- **Rate limit:** 5 req/h per user.

---

## 6. Secrets

### `POST /api/secrets/reveal`
- **Request:** `{ "secret_id": "...", "reason": "Interwencja techniczna" }`
- **Response:** Odsłonięte hasło (TTL: 60s). Zdarzenie logowane w AUDIT_LOG.
- Uwaga: `userId` pobierany z tokena JWT.

---

## 7. Error Handling — Standard

### 7.1 Format odpowiedzi błędu

Każdy endpoint w przypadku błędu zwraca jednolity format:

```json
HTTP {status_code}
{
  "error": {
    "code": "ERROR_CODE_UPPERCASE",
    "message": "Opis czytelny dla developera",
    "details": {
      "current_state": "aktualny stan zasobu na serwerze (opcjonalne)",
      "your_version": "wersja wysłana przez klienta (opcjonalne)",
      "server_version": "aktualna wersja na serwerze (opcjonalne)"
    }
  }
}
```

> **Zasada:** Każdy błąd `409` MUSI zawierać `current_state` i/lub `server_version` w `details`, aby klient mógł podjąć decyzję (odświeżyć UI vs pokazać komunikat).

### 7.2 Kody błędów biznesowych

| HTTP Status | Error Code | Opis | Endpoint |
|---|---|---|---|
| 400 | `INVALID_REQUEST` | Brak wymaganych pól lub błędny format | Wszystkie |
| 401 | `UNAUTHORIZED` | Brak tokena lub token wygasł | Wszystkie |
| 403 | `FORBIDDEN` | Brak uprawnień (rola nie pasuje) | Wszystkie |
| 404 | `OBJECT_NOT_FOUND` | Obiekt o podanym ID nie istnieje | /objects/{id} |
| 404 | `ALARM_NOT_FOUND` | Alarm o podanym ID nie istnieje | /alarms/{id} |
| 409 | `ALARM_ALREADY_CLAIMED` | Alarm jest już obsługiwany przez innego operatora | /alarms/{id}/claim |
| 409 | `ALARM_STALE_VERSION` | Wersja alarmu nie zgadza się — ktoś inny zmodyfikował alarm w międzyczasie | /alarms/{id}/* |
| 409 | `ALARM_INVALID_STATE` | Próba przejścia do niedozwolonego stanu (np. NEW→CLOSED) | /alarms/{id}/* |
| 409 | `OBJECT_STALE_VERSION` | Wersja obiektu nie zgadza się | /objects/{id} |
| 409 | `COMMAND_DUPLICATE` | Komenda z tym samym idempotency key już istnieje | /commands/satel |
| 409 | `COMMAND_ALREADY_SENT` | Nie można anulować komendy która już została wysłana | /commands/satel/{id} |
| 422 | `TEMP_ALARM_NOTE_REQUIRED` | Alarm temperaturowy wymaga notatki przy zamykaniu | /alarms/{id}/close |
| 422 | `NOTE_TOO_SHORT` | Notatka krótsza niż 10 znaków | /alarms/{id}/close |
| 422 | `IMPORT_INVALID_FORMAT` | Plik nie jest .xlsx lub brak wymaganych kolumn | /admin/import/objects |
| 422 | `IMPORT_DUPLICATE_NAME` | Nazwa obiektu już istnieje w systemie | /admin/import/objects |
| 422 | `DISPATCH_CONTACT_CONFLICT` | Podano zarówno `contact_id` jak i pola manualne | /dispatch-log |
| 429 | `RATE_LIMIT_EXCEEDED` | Zbyt wiele requestów | /auth/login, /secrets/reveal |
| 503 | `SATEL_CONNECTION_LOST` | Worker stracił połączenie z centralą | /integrations/satel/* |
| 503 | `PANEL_NOT_CONNECTED` | Panel nie ma aktywnego połączenia TCP | /objects/{id}/service-mode |
| 409 | `SERVICE_MODE_ALREADY_ACTIVE` | Tryb serwisowy już jest aktywny na tym obiekcie | /objects/{id}/service-mode |
| 503 | `SMS_MODEM_UNAVAILABLE` | Modem SMS niedostępny | /integrations/sms/* |

### 7.3 Strategia retry vs fail-fast

| Kontekst | Strategia | Retry |
|---|---|---|
| API request z frontendu | Fail-fast | Frontend decyduje (max 2 retry) |
| Worker → ETHM-1 | Retry | Exponential Backoff (patrz 14_SATEL_COMMANDS) |
| Worker → RabbitMQ publish | Retry | 3x z 1s delay, potem → DLQ |
| SMS Agent → RabbitMQ | Retry | 3x, potem → local buffer file |
| Backend → PostgreSQL | Retry | 2x z 500ms, potem → 503 |
| Backend → Redis | Fail-fast | Fallback na direct DB query |

### 7.4 Przykłady Error Payload (JSON)

#### 409 ALARM_STALE_VERSION (Optimistic Lock Conflict)

Operator A i B równocześnie edytują ten sam alarm. A jest szybszy.

```json
HTTP 409
{
  "error": {
    "code": "ALARM_STALE_VERSION",
    "message": "Alarm został zmodyfikowany przez innego użytkownika. Odśwież dane i spróbuj ponownie.",
    "details": {
      "bundle_id": "ba_001",
      "your_version": 2,
      "server_version": 3,
      "current_state": "ACK",
      "modified_by": "Jan Kowalski",
      "modified_at": "2026-02-13T14:30:00Z"
    }
  }
}
```

#### 409 ALARM_ALREADY_CLAIMED (Double Claim)

Dwóch operatorów klika "Obsługuj" jednocześnie.

```json
HTTP 409
{
  "error": {
    "code": "ALARM_ALREADY_CLAIMED",
    "message": "Alarm jest już obsługiwany przez innego operatora.",
    "details": {
      "bundle_id": "ba_001",
      "current_state": "IN_PROGRESS",
      "assigned_to": "Anna Nowak",
      "claimed_at": "2026-02-13T14:29:58Z"
    }
  }
}
```

#### 409 ALARM_INVALID_STATE (Niedozwolone przejście)

Próba zamknięcia alarmu, który nie przeszedł przez RESOLVED.

```json
HTTP 409
{
  "error": {
    "code": "ALARM_INVALID_STATE",
    "message": "Nie można przejść ze stanu IN_PROGRESS do CLOSED. Wymagany pośredni stan: ACK lub RESOLVED.",
    "details": {
      "bundle_id": "ba_001",
      "current_state": "IN_PROGRESS",
      "requested_state": "CLOSED",
      "allowed_transitions": ["ACK", "CLOSED"]
    }
  }
}
```

#### 409 OBJECT_STALE_VERSION (Edycja obiektu)

```json
HTTP 409
{
  "error": {
    "code": "OBJECT_STALE_VERSION",
    "message": "Obiekt został zmodyfikowany. Odśwież dane i spróbuj ponownie.",
    "details": {
      "object_id": "obj_123",
      "your_version": 5,
      "server_version": 6,
      "modified_by": "Admin",
      "modified_at": "2026-02-13T14:25:00Z"
    }
  }
}
```

#### 409 COMMAND_DUPLICATE (Podwójna komenda)

```json
HTTP 409
{
  "error": {
    "code": "COMMAND_DUPLICATE",
    "message": "Komenda z tym samym kluczem idempotentności już istnieje.",
    "details": {
      "existing_command_id": "cmd_456",
      "current_state": "SENT",
      "created_at": "2026-02-13T14:30:02Z"
    }
  }
}
```

#### 422 TEMP_ALARM_NOTE_REQUIRED (Brak notatki)

```json
HTTP 422
{
  "error": {
    "code": "TEMP_ALARM_NOTE_REQUIRED",
    "message": "Alarm temperaturowy wymaga notatki wyjaśniającej przyczynę.",
    "details": {
      "bundle_id": "ba_789",
      "requires_note": true,
      "alarm_type": "TEMP",
      "min_note_length": 10
    }
  }
}
```

### 7.5 Client Guidance — Obsługa błędów w UI (Flutter)

| HTTP | Error Code | Akcja UI | UX Pattern |
|---|---|---|---|
| **409** | `ALARM_STALE_VERSION` | Auto-refresh danych alarmu z serwera, pokaż toast "Alarm zaktualizowany, odświeżono dane" | Refresh + Retry |
| **409** | `ALARM_ALREADY_CLAIMED` | Pokaż dialog: "Alarm obsługiwany przez {assigned_to}". Wyłącz przycisk Claim. | Inform + Disable |
| **409** | `ALARM_INVALID_STATE` | Pokaż toast z `allowed_transitions`. Odśwież stan. | Inform + Refresh |
| **409** | `OBJECT_STALE_VERSION` | Pokaż dialog: "Dane zostały zmienione przez {modified_by}. Odświeżyć i ponowić edycję?" | Confirm + Refresh |
| **409** | `COMMAND_DUPLICATE` | Ignoruj cicho — komenda już istnieje, nie trzeba ponawiać | Silent Ignore |
| **409** | `COMMAND_ALREADY_SENT` | Pokaż toast "Komenda już wysłana, nie można anulować" | Inform |
| **422** | `TEMP_ALARM_NOTE_REQUIRED` | Podświetl pole notatki na czerwono, pokaż tekst "Wymagana notatka (min 10 znaków)" | Validate + Focus |
| **422** | `NOTE_TOO_SHORT` | Podświetl pole, pokaż counter "X/10 znaków" | Validate + Counter |
| **202** | (Command accepted) | Pokaż spinner na przycisku Arm/Disarm. Polluj `GET /commands/satel/{id}` co 1s lub czekaj na WS event `command.status_change` | Spinner + Poll/WS |

**Zasady ogólne:**

1. **409 = Odśwież i spróbuj ponownie.** Klient MUSI pobrać aktualny stan z serwera po otrzymaniu 409. NIE ponawiaj requestu z tą samą `version`.
2. **422 = Popraw dane wejściowe.** Klient musi poprawić formularz i wysłać ponownie. Nie trzeba odświeżać całego stanu.
3. **202 = Operacja zakolejkowana.** Klient NIE może zakładać, że komenda się powiodła. Musi czekać na potwierdzenie (poll lub WebSocket).
4. **`version` jest wymagane** w każdym req mutation (PATCH/POST do /claim, /ack, /resolve, /close). Brak `version` → `400 INVALID_REQUEST`.

---

## 8. WebSocket Events — Specification

### 8.1 Połączenie

- **URL:** `wss://{host}/api/ws?ticket={ticket}`
- **Auth:** Ticket-based. Klient pobiera jednorazowy ticket z `POST /api/auth/ws-ticket` (TTL 10s) — szczegóły: `08_SECURITY_AND_ROLES.md`, sekcja 4.3a.
- **Re-walidacja:** Co 15 min Backend wysyła frame `auth_check` — klient odpowiada nowym `access_token`.
- **Reconnect:** Exponential backoff 3s → 6s → 12s → 30s (cap). **Brak limitu prób** — klient odłączony = klient ślepy na alarmy.
- **Catch-up:** Po reconnect klient wysyła `replay_request` z ostatnim `sequence_id` — patrz sekcja 8.5.

### 8.2 Eventy Server → Client

| Event Type | Payload | Opis |
|---|---|---|
| `alarm.new` | `{ bundle_id, object_name, priority, event_count, first_seen, version }` | Nowy Bundle Alarm (migaj na czerwono!) |
| `alarm.updated` | `{ bundle_id, status, assigned_to, event_count, last_seen, version }` | Zmiana statusu alarmu (zawiera `version` do optimistic refresh) |
| `alarm.closed` | `{ bundle_id }` | Alarm zamknięty |
| `panel.status` | `{ panel_id, connection_status, zones_summary }` | Zmiana stanu połączenia z centralą |
| `command.status_change` | `{ command_id, panel_id, status, error_message, executed_at }` | Zmiana statusu komendy Satel (SENT→ACK/NACK/TIMEOUT) |
| `notification` | `{ message, type, target_user_id }` | Powiadomienie systemowe |

### 8.3 Eventy Client → Server

| Event Type | Payload | Opis |
|---|---|---|
| `ping` | `{}` | Keep-alive (co 30s) |
| `subscribe` | `{ channels: ["alarms", "panels"] }` | Subskrypcja kanałów |
| `replay_request` | `{ last_sequence_id: 41 }` | Żądanie odtworzenia utraconych eventów po reconnect |

### 8.4 Format wiadomości

```json
{ "sequence_id": 42, "type": "alarm.new", "data": { ... }, "timestamp": "ISO8601" }
```

- `sequence_id` (int) — Monotonicznie rosnący identyfikator per sesja WebSocket. Używany do catch-up po reconnect.

### 8.5 Catch-up po reconnect (Replay Mechanism)

Po rozłączeniu i ponownym połączeniu, klient może wysłać `replay_request` z ostatnim odebranym `sequence_id`. Backend odtwarza utracone eventy z Redis buffer.

**Redis buffer:** `ws:events:buffer` — Sorted Set (score = `sequence_id`, value = JSON message). Max **5000** wpisów (ZREMRANGEBYRANK przy każdym insercie). Tier 1 mechanizmu Tiered Catch-Up.

**Flow:**
1. Klient reconnect → `replay_request { last_sequence_id: N }`
2. Backend sprawdza tier:
   - **Tier 1 (Redis, <5 min):** `ZRANGEBYSCORE ws:events:buffer (N +inf` → lista utraconych eventów z Redis.
   - **Tier 2 (PostgreSQL, 5 min–2h):** Jeśli `last_sequence_id` starszy niż Redis buffer → `SELECT * FROM outbox WHERE sequence_id > :N AND created_at > NOW() - INTERVAL '2 hours' ORDER BY sequence_id ASC LIMIT 5000`.
   - **Tier 3 (>2h):** Zwróć `replay_overflow` → klient robi pełny refresh przez REST API.
3. Backend wysyła eventy w kolejności `sequence_id` (ascending).
4. Po replay → live stream wznowiony od bieżącego `sequence_id`.

**Edge cases:**

| Scenariusz | Zachowanie |
|---|---|
| `last_sequence_id` w Redis buffer | Replay z Redis (szybki, <50ms) |
| `last_sequence_id` starszy niż Redis, ale <2h | Replay z PostgreSQL (wolniejszy, ~200ms) |
| `last_sequence_id` starszy niż 2h | `{ type: "replay_overflow", retry_after_ms: {random 0-5000} }` → klient czeka `retry_after_ms` przed pełnym REST refresh (ochrona przed thundering herd) |
| Klient nie wysyła `replay_request` | Brak replay, live stream zaczyna się od bieżącego — backwards compatible |
| Wielokrotne reconnecty w krótkim czasie | Każdy `replay_request` przetwarzany niezależnie, idempotentny |
| Redis restart (pusty bufor) | Automatyczny fallback na Tier 2 (PostgreSQL). Brak utraty danych. |

## 9. Paginacja — Standard

### 9.1 Request

Wszystkie endpointy listujące (GET /api/objects, GET /api/alarms, etc.) akceptują:

| Parametr | Typ | Default | Max | Opis |
|---|---|---|---|---|
| `page` | int | 1 | — | Numer strony (1-based) |
| `limit` | int | 20 | 100 | Ilość elementów na stronę |
| `sort` | string | zależny od endpointu | — | Pole sortowania (np. `first_seen`) |
| `order` | enum | `desc` | — | `asc` / `desc` |

### 9.2 Response Wrapper

```
{
  "data": [ ... ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 156,
    "total_pages": 8
  }
}
```

---

## 10. Timeout Raportów

Wszystkie endpointy generujące raporty (moduł Raporty, eksporty CSV/PDF) podlegają limitowi czasu wykonania.

### 10.1 Zasada

- **Timeout:** Każde zapytanie raportowe, które trwa dłużej niż **30 sekund**, jest automatycznie anulowane.
- **Kod błędu:** `408 Request Timeout`
- **Komunikat:** `"Raport przekroczył dozwolony czas generowania. Spróbuj zawęzić zakres dat."`
- Parametr konfiguracyjny: `REPORT_TIMEOUT_SECONDS = 30` (domyślnie).

### 10.2 Dotyczy endpointów

| Endpoint | Opis |
|---|---|
| `GET /api/reports/alarms` | Raport alarmów (z filtrami dat) |
| `GET /api/reports/response-time` | Raport czasu reakcji |
| `GET /api/reports/objects/{id}` | Raport per obiekt |
| `GET /api/reports/export/csv` | Eksport CSV |
| `GET /api/reports/export/pdf` | Eksport PDF |

### 10.3 Format odpowiedzi błędu

```
HTTP 408
{
  "error": {
    "code": "REPORT_TIMEOUT",
    "message": "Raport przekroczył dozwolony czas generowania. Spróbuj zawęzić zakres dat.",
    "details": {
      "timeout_seconds": 30
    }
  }
}
```

### 10.4 Zalecenia

- Raporty powinny korzystać z **read-only repliki** bazy danych (patrz **02_ARCHITECTURE.md, sekcja Separacja ruchu**).
- Frontend powinien wyświetlać spinner z informacją o limicie czasu.
- Dla długich raportów rozważyć generowanie asynchroniczne (v2.0).

---

## 11. Rate Limiting

> Ochrona przed nadużyciem API. Wszystkie limity egzekwowane na poziomie reverse proxy (Nginx/Traefik) + middleware FastAPI.

| Ścieżka | Limit | Per | Kod odpowiedzi |
|---|---|---|---|
| `POST /api/auth/login` | 5 req/min | IP | 429 |
| `POST /api/auth/ws-ticket` | 10 req/min | User | 429 |
| `POST /api/secrets/reveal` | 10 req/h (OPERATOR), 20 req/h (TECH) | User | 429 |
| `GET /api/alarms` | 60 req/min | User | 429 |
| `POST /api/commands/satel` | 10 req/min | User | 429 |
| `GET /api/reports/*` | 10 req/min | User | 429 |
| **Pozostałe** | **120 req/min** | User | 429 |

**Odpowiedź 429:**
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Zbyt wiele żądań. Spróbuj ponownie za {retry_after} sekund.",
    "details": { "retry_after": 30 }
  }
}
```

---

## 12. Paginacja — Edge Cases

### Cursor Pagination (`GET /api/alarms`)

| Scenariusz | Zachowanie |
|---|---|
| Brak `after` | Zwróć pierwszą stronę (najnowsze alarmy) |
| Nieprawidłowy `after` (base64 decode fail) | `400 INVALID_CURSOR` |
| `limit > 100` | Obetnij do max 100, zwróć `limit: 100` |
| `limit < 1` | `400 VALIDATION_ERROR` |
| `has_more = false` | Klient nie powinien pobierać następnej strony |

### Offset Pagination (`GET /api/objects`, `GET /api/reports/*`)

| Scenariusz | Zachowanie |
|---|---|
| `page > total_pages` | Zwróć pustą tablicę `data: []` z `total: N`, `total_pages: M` |
| `page < 1` | `400 VALIDATION_ERROR` |
| `limit > 100` | Obetnij do max 100, zwróć `limit: 100` w response |
| `limit < 1` | `400 VALIDATION_ERROR` |

---

## 13. Import Excel (Faza 1)

### `POST /api/admin/import/objects`
- **Auth:** ADMIN, MASTER.
- **Request:** `multipart/form-data`, pole `file` — plik `.xlsx`.
- **Response (200):**
  ```json
  {
    "imported": 45,
    "skipped": 3,
    "errors": [
      { "row": 12, "column": "nazwa", "code": "IMPORT_DUPLICATE_NAME", "value": "Biuro Centrala" },
      { "row": 28, "column": "typ", "code": "IMPORT_INVALID_FORMAT", "value": "nieznany_typ" }
    ],
    "import_id": "imp_abc123"
  }
  ```
- **Kody błędów:** `IMPORT_INVALID_FORMAT`, `IMPORT_DUPLICATE_NAME`.
- **Limit:** Max 1000 wierszy per import.
- **Schemat kolumn:** **19_DEV_TOOLING.md, sekcja 4**.
- **Log:** Każdy import zapisywany w tabeli `IMPORT_LOG` (**04_DATA_MODEL_ERD.md**).

---

## 14. Push Notifications — FCM Tokens (Faza 2)

### `POST /api/devices/register`
- **Auth:** Bearer JWT (dowolna rola z dostępem mobilnym).
- **Request:**
  ```json
  {
    "fcm_token": "eB3sD4...",
    "device_name": "Samsung Galaxy S24",
    "platform": "android"
  }
  ```
- **Response (201):** `{ "device_id": "dev_abc123" }`
- **Logika:** Jeśli `fcm_token` już istnieje, aktualizuj `last_used_at`. Jeden user może mieć wiele urządzeń.

### `DELETE /api/devices/{device_id}`
- **Auth:** Bearer JWT (właściciel urządzenia).
- **Cel:** Wyrejestrowanie tokenu FCM przy wylogowaniu.
- **Response (204):** No content.

### `GET /api/devices`
- **Auth:** Bearer JWT.
- **Response:** Lista urządzeń użytkownika z `device_name`, `platform`, `last_used_at`.

---

## 15. Tryb Serwisowy (Faza 1: config, Faza 2: efekt)

### `POST /api/objects/{id}/service-mode`
- **Auth:** ADMIN, MASTER, TECHNICIAN.
- **Request:**
  ```json
  {
    "reason": "Wymiana czujnika PIR w strefie 3",
    "until": "2026-02-21T06:00:00Z"
  }
  ```
- **Response (200):** `{ "service_mode_active": true, "activated_by": "user_id", "until": "..." }`
- **Efekt (Faza 2):** Alarmy z tego obiektu: priorytet obniżony, brak dźwięku, szary styl + 🔧.

### `DELETE /api/objects/{id}/service-mode`
- **Auth:** ADMIN, MASTER, TECHNICIAN.
- **Response (204):** Tryb serwisowy dezaktywowany.

---

## 16. Dziennik Dyspozytora (Faza 2)

### `POST /api/dispatch-log`
- **Auth:** OPERATOR, ADMIN, MASTER.
- **Request:**
  ```json
  {
    "object_id": "obj_123",
    "bundle_id": "ba_456",
    "entry_type": "outbound_call",
    "contact_id": null,
    "contact_name_manual": "Jan Kowalski",
    "contact_phone_manual": "+48123456789",
    "note": "Poinformowano o alarmie, wysłany patrol"
  }
  ```
- **Response (201):** `{ "log_id": "dl_789" }`

> [!IMPORTANT]
> **Walidacja `contact_id` vs pola manualne:** Jeśli `contact_id` != null, pola `contact_name_manual` i `contact_phone_manual` MUSZĄ być puste (null). Backend zwraca `422 DISPATCH_CONTACT_CONFLICT` przy próbie podania obu.

### `GET /api/dispatch-log?object_id={id}&bundle_id={id}`
- **Auth:** OPERATOR, ADMIN, MASTER.
- **Response:** Lista wpisów z paginacją.

---

## 17. Historia Stanu Centrali (Faza 2)

### `GET /api/panels/{panel_id}/state-history`
- **Auth:** OPERATOR, ADMIN, MASTER, TECHNICIAN.
- **Query params:** `partition_id`, `from`, `to`, `page`, `limit`.
- **Response:**
  ```json
  {
    "data": [
      {
        "id": "psh_001",
        "panel_id": "panel_123",
        "partition_id": 1,
        "old_state": "DISARMED",
        "new_state": "ARMED_FULL",
        "source": "satel_worker",
        "triggered_by": "system",
        "recorded_at": "2026-02-20T14:30:00Z"
      }
    ],
    "total": 150,
    "page": 1
  }
  ```

---

## 18. Konfiguracja Alarmowa Obiektu (Faza 1)

### `GET /api/objects/{id}/alarm-config`
- **Auth:** ADMIN, MASTER.
- **Response:** Aktualna konfiguracja `OBJECT_ALARM_CONFIG` (bundle window, auto-close timeout itp.).

### `PUT /api/objects/{id}/alarm-config`
- **Auth:** ADMIN, MASTER.
- **Request:**
  ```json
  {
    "bundle_window_minutes": 10,
    "auto_close_timeout_hours": 24,
    "suppress_duplicate_minutes": 5
  }
  ```
- **Response (200):** Zaktualizowana konfiguracja.
- **Uwaga:** Wartości `null` = użyj globalnych ustawień domyślnych.

### `GET /api/objects/{id}/contacts`
- **Auth:** OPERATOR, ADMIN, MASTER.
- **Response:** Lista kontaktów alarmowych z `OBJECT_CONTACTS`.

### `POST /api/objects/{id}/contacts`
- **Auth:** ADMIN, MASTER.
- **Request:** `{ "name": "...", "phone": "...", "role": "...", "priority_order": 1 }`
- **Response (201):** Nowy kontakt.

### `GET /api/objects/{id}/procedures`
- **Auth:** OPERATOR, ADMIN, MASTER.
- **Response:** Lista procedur z `OBJECT_PROCEDURES`.
