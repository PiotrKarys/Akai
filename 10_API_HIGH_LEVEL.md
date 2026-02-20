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
| 429 | `RATE_LIMIT_EXCEEDED` | Zbyt wiele requestów | /auth/login, /secrets/reveal |
| 503 | `SATEL_CONNECTION_LOST` | Worker stracił połączenie z centralą | /integrations/satel/* |
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
