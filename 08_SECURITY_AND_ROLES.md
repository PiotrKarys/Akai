# 08_SECURITY_AND_ROLES.md

## Cel
Opis modelu bezpieczeństwa i ról użytkowników (RBAC).

---

## 1. Role (RBAC)

### SYSTEM (Bot) — Sub-role

Rola techniczna rozdzielona na **sub-role** dla zasady minimalnych uprawnień:

| Sub-rola | Serwis | Uprawnienia |
|---|---|---|
| `SYSTEM_WORKER` | Satel Worker | `integration:write` (publish events), `panel:state:write` (Redis) |
| `SYSTEM_SMS` | SMS Agent | `integration:write` (publish SMS events) |
| `SYSTEM_RELAY` | Outbox Relay | `outbox:read`, `outbox:publish` |

**Uzasadnienie:** Pojedynczy token SYSTEM z pełnym dostępem stanowi ryzyko — kompromitacja Workera dawała dostęp do alarmów, użytkowników i konfiguracji. Sub-role eliminują ten wektor.

**Wdrożenie:** Każdy serwis otrzymuje własny JWT z `role: "SYSTEM_WORKER"` (lub odpowiednio). Backend weryfikuje sub-rolę przy każdym endpoincie `[Internal Only]`.

### MASTER (Właściciel)
- Pełen dostęp do wszystkiego.
- Może odsłaniać każde hasło.
- Widzi pełen Audit Log.

### ADMIN
- Zarządzanie użytkownikami (niższymi rolami).
- Konfiguracja systemu.
- Nie ma dostępu do haseł "Highest Security" bez logowania incydentu.

### OPERATOR (Centrum Monitorowania)
- **Główna rola operacyjna.**
- Widzi i obsługuje alarmy.
- Nie może usuwać obiektów ani historii.
- Dostęp do haseł technicznych tylko "na żądanie" (Timed Reveal).

### TECHNICIAN (Serwisant)
- Dostęp głównie mobilny / terenowy.
- Tryb "Serwisowy" na obiektach.
- Widzi dokumentację techniczną.

### FIELD WORKER (Audytor / Tester)
- Wykonuje okresowe audyty fizyczne instalacji.
- Testuje poprawność działania czujek i integracji z centralą.
- Raportuje wyniki audytów i inicjuje zgłoszenia serwisowe.

#### Scope Enforcement (FIELD_WORKER + TECHNICIAN)

**Problem:** Rola FIELD_WORKER ma adnotację "tylko swoje obiekty" — ale bez mechanizmu egzekucji backend zwraca WSZYSTKIE obiekty.

**Rozwiązanie:** Tabela `user_object_assignments` (szczegóły: `04_DATA_MODEL_ERD.md`).

```
user_object_assignments:
  user_id   FK → USERS.id
  object_id FK → OBJECTS.id
  assigned_by FK → USERS.id (kto przypisał)
  assigned_at TIMESTAMP
  PRIMARY KEY (user_id, object_id)
```

**Middleware filtrowania:**

```python
# Backend middleware — automatycznie dodaje filtr per rola
if user.role in ("FIELD_WORKER", "TECHNICIAN"):
    query = query.filter(
        Object.id.in_(
            select(UserObjectAssignment.object_id)
            .where(UserObjectAssignment.user_id == user.id)
        )
    )
```

**Zasady:**
- ADMIN/MASTER przypisuje obiekty do FIELD_WORKER/TECHNICIAN w panelu administracyjnym.
- Brak przypisań = brak dostępu do jakiegokolwiek obiektu (fail-closed).
- Alarmy filtrowane przez łańcuch: alarm → bundle → object → assignment → user.
- Każda zmiana przypisań logowana w `AUDIT_LOG`.

---

## 2. Mechanizmy Bezpieczeństwa

### Timed Secret Reveal
Operator nie widzi haseł do central/kamer domyślnie.
Aby zobaczyć hasło:
1. Klika "Pokaż hasło".
2. Podaje powód (np. "Interwencja techniczna").
3. Hasło odsłania się na 60 sekund.
4. Zdarzenie jest logowane w `AUDIT_LOG` (Kto, Co, Kiedy, Powód).

### PIN / Biometria (App Mobile)
- Aplikacja mobilna wymaga PINu przy otwieraniu.
- Możliwe użycie odcisku palca (Biometria natywna Android/iOS).

### Sieć
- Backend nie jest wystawiony publicznie.
- Dostęp tylko z sieci LAN lub przez VPN.
- Szyfrowanie TLS zalecane nawet w LAN (Self-signed lub wewnętrzne CA).

---

## 3. Audyt (Audit Log)

System rejestruje **każdą** istotną akcję:
- Logowanie / Wylogowanie.
- Potwierdzenie / Zamknięcie alarmu.
- Edycja danych obiektu.
- Odsłonięcie hasła.
- Zmiana uprawnień użytkownika.

**Zasada:** Logi audytowe są "append-only".

---

## 4. Auth / Authorization Flow (JWT)

### 4.1 Token Policy

| Parametr | Wartość |
|---|---|
| Access Token TTL | 15 min |
| Refresh Token TTL | 7 dni |
| **Algorytm** | **RS256 (asymetryczny)** |
| Issuer | `stam-replacer-api` |
| Audience | `stam-replacer-clients` |

#### RS256 — Klucze Asymetryczne

| Element | Opis |
|---|---|
| Klucz prywatny (`JWT_PRIVATE_KEY`) | Używany **wyłącznie** przez Auth Service do podpisywania tokenów. RSA 2048 bit. |
| Klucz publiczny (`JWT_PUBLIC_KEY`) | Dystrybuowany do wszystkich serwisów weryfikujących tokeny. |
| `kid` (Key ID) | Każdy token zawiera `kid` w nagłówku JWT, wskazujący na wersję klucza. Umożliwia rotację bez przerwy. |
| Rotacja | Co 90 dni. Nowy klucz dodawany, stary pozostaje aktywny do wygaśnięcia ostatniego tokena (max 7 dni overlap). |

**Uzasadnienie migracji z HS256:** HS256 używa jednego klucza do podpisywania i weryfikacji. Wyciek klucza z dowolnego serwisu (Worker, SMS Agent) pozwala fałszować tokeny dla dowolnej roli (włącznie z MASTER). RS256 separuje podpisywanie (prywatny) od weryfikacji (publiczny).

**Generowanie kluczy:**
```bash
# Generuj parę kluczy RSA 2048
openssl genrsa -out jwt_private.pem 2048
openssl rsa -in jwt_private.pem -pubout -out jwt_public.pem
```

### 4.2 Access Token Claims (Payload)

| Claim | Typ | Opis |
|---|---|---|
| `sub` | string | User ID (UUID) |
| `role` | string | Rola kanonyczna (np. "OPERATOR", "SYSTEM_WORKER") |
| `permissions` | string[] | Lista uprawnień (np. ["alarm:read", "alarm:write"]) |
| `scope` | string[] | *Opcjonalne.* Ograniczenie zakresu (np. lista `object_id` dla FIELD_WORKER) |
| `exp` | int | Timestamp wygaśnięcia |
| `iat` | int | Timestamp wydania |
| `iss` | string | Stałe: "stam-replacer-api" |
| `kid` | string | **W nagłówku JWT** — ID klucza podpisującego |

**Walidacja krytyczna:** Przy operacjach wrażliwych (zmiana roli, reveal hasła, sterowanie centralą) backend **MUSI** sprawdzić rolę użytkownika w bazie danych, a nie polegać wyłącznie na claim `role` z JWT. Wykrycie rozbieżności → natychmiastowa invalidacja tokena.

### 4.3 Przepływ Autoryzacji

```
1. POST /api/auth/login {email, password}
   → Backend: sprawdź dane (Argon2id), wygeneruj Access + Refresh Token (RS256)
   ← Response: {access_token, refresh_token, user}

2. Każdy request REST API:
   → Header: Authorization: Bearer {access_token}
   → Backend: middleware weryfikuje JWT (klucz publiczny), wyciąga role/permissions
   → Jeśli brak/expired → HTTP 401

3. POST /api/auth/refresh {refresh_token}
   → Backend: sprawdź refresh token, wygeneruj nowy Access Token
   ← Response: {access_token}

4. POST /api/auth/logout
   → Backend: unieważnij refresh token (blacklist w Redis, TTL = refresh TTL)
```

### 4.3a Przepływ WebSocket — Ticket-Based Auth

**Problem:** Token JWT w query string (`?token=xxx`) trafia do logów proxy, reverse proxy, i historii przeglądarki.

**Rozwiązanie:** Jednorazowy bilet (ticket/nonce) z krótkim TTL.

```
1. Klient → POST /api/auth/ws-ticket
   → Header: Authorization: Bearer {access_token}
   ← Response: {ticket: "abc123", expires_in: 10}
   → Backend: zapisz ticket w Redis z TTL 10s:
     SET ws_ticket:abc123 {user_id, role, permissions} EX 10

2. Klient → WebSocket CONNECT ws://host/ws?ticket=abc123
   → Backend: pobierz ticket z Redis (GET ws_ticket:abc123)
   → Jeśli istnieje → usuń z Redis (jednorazowy), nawiąż sesję WS
   → Jeśli nie istnieje lub wygasł → odrzuć z kodem 4001

3. W trakcie sesji WS — re-walidacja co 15 min:
   → Backend: co 15 min wysyła frame "auth_check"
   → Klient: odpowiada nowym access_token (refresh jeśli potrzeba)
   → Backend: weryfikuje token, jeśli invalid → zamknij WS z kodem 4002
   → Klient: automatyczny reconnect (nowy ticket → nowe WS)
```

**Bezpieczeństwo:**
- Ticket jednorazowy — użycie go dwa razy jest niemożliwe (DELETE po pierwszym użyciu).
- TTL 10s — okno ataku minimalne.
- Re-walidacja co 15 min — sesja WS nie może trwać wiecznie z wygasłym tokenem.

### 4.4 Permission Matrix (Role → Uprawnienia)

| Uprawnienie | SYSTEM | MASTER | ADMIN | OPERATOR | TECHNICIAN | FIELD_WORKER |
|---|---|---|---|---|---|---|
| `alarm:read` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ (tylko swoje) |
| `alarm:write` | ✓ | ✓ | ✓ | ✓ | ✗ | ✓ (zgłoszenia) |
| `object:read` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ (tylko swoje) |
| `object:write` | ✓ | ✓ | ✓ | ✗ | ✗ | ✗ |
| `user:read` | ✗ | ✓ | ✓ | ✗ | ✗ | ✗ |
| `user:write` | ✗ | ✓ | ✓ | ✗ | ✗ | ✗ |
| `secret:reveal` | ✗ | ✓ | ✓ | ✓ (limited) | ✓ (limited) | ✓ (limited) |
| `audit:read` | ✗ | ✓ | ✓ | ✗ | ✗ | ✗ |
| `system:config` | ✗ | ✓ | ✓ | ✗ | ✗ | ✗ |
| `integration:write` | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ |
| `ticket:read` | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `ticket:write` | ✗ | ✓ | ✓ | ✓ | ✗ | ✓ (zgłoszenia) |
| `sms_raw:read` | ✓ (SYSTEM) | ✓ | ✗ | ✗ | ✗ | ✗ |

> [!CAUTION]
> **PII Isolation:** Dostęp do tabeli `sms_raw_archive` (oryginalne treści SMS zawierające dane osobowe) ograniczony wyłącznie do ról MASTER i SYSTEM. Operator/Admin widzi tylko `raw_sms_hash`. Każdy dostęp logowany w `AUDIT_LOG` (action: `SMS_RAW_ACCESS`).

### 4.5 Password Security

| Parametr | Wartość |
|---|---|
| Hashing | Argon2id |
| Memory cost | 64 MB |
| Iterations | 3 |
| Parallelism | 1 |
| Min password length | 8 znaków |
| Max password length | 128 znaków |

### 4.6 Brute-Force Protection

| Parametr | Wartość |
|---|---|
| Rate limit `/auth/login` | 5 prób / minutę (per IP) |
| Lockout po X failures | 10 failures → konto blokowane na 30 min |
| Tracking | Redis counter: `login_fail:{email}` (TTL: 30 min) |
| Alert | Po 3 failures → log WARNING, po 10 → log CRITICAL + AUDIT_LOG |

### 4.7 Parallel Sessions

- Dozwolone: jeden użytkownik może być zalogowany na wielu urządzeniach jednocześnie.
- Każde urządzenie dostaje własny refresh token.
- Admin/Master może wymusić "wyloguj wszystkie sesje" (invalidacja wszystkich refresh tokenów użytkownika).

---

## 5. Timed Secret Reveal — Specyfikacja Techniczna

### Przechowywanie haseł

| Element | Metoda |
|---|---|
| Hasła do central/kamer | Szyfrowanie symetryczne: **Fernet / MultiFernet** (Python `cryptography`) |
| Klucz szyfrowania | **Docker Secret:** `encryption_key` (nie `.env` — patrz sekcja 6) |
| Backend decryptuje | Tylko w momencie "reveal", po weryfikacji uprawnień |

### Rotacja Kluczy (MultiFernet)

```python
from cryptography.fernet import Fernet, MultiFernet

# Rotacja: nowy klucz na pierwszym miejscu, stary na drugim
key_new = Fernet(ENCRYPTION_KEY_V2)  # aktualny klucz
key_old = Fernet(ENCRYPTION_KEY_V1)  # poprzedni klucz (do odczytu starych danych)
multi_fernet = MultiFernet([key_new, key_old])

# Dekrypcja — próbuje nowy, potem stary:
password = multi_fernet.decrypt(encrypted_password)

# Re-encrypt starych danych nowym kluczem (batch job):
new_token = multi_fernet.rotate(old_encrypted_password)
```

**Harmonogram rotacji:**
- Co **180 dni** generuj nowy `ENCRYPTION_KEY`.
- Przez 30 dni oba klucze aktywne (MultiFernet).
- Po 30 dniach uruchom batch re-encryption i usuń stary klucz.
- Każda rotacja logowana w `AUDIT_LOG` (action: `KEY_ROTATION`).

### Przepływ Reveal

```
1. Frontend: POST /api/secrets/reveal {secret_id, reason}
2. Backend: sprawdź JWT → czy rola ma `secret:reveal`
3. Backend: dekryptuj hasło (MultiFernet)
4. Backend: zapisz w AUDIT_LOG {user_id, action: "SECRET_REVEAL", entity_id: secret_id, details: {reason}}
5. Backend: zwróć {password: "...", expires_at: now + 60s}
6. Frontend: wyświetl hasło + countdown 60s
7. Frontend: po 60s → ukryj hasło (client-side), wyczyść z pamięci
```

### Ograniczenia
- OPERATOR: max 10 reveals / godzinę
- TECHNICIAN: max 20 reveals / godzinę
- FIELD_WORKER: max 20 reveals / godzinę
- MASTER: bez limitu
- Każdy reveal logowany w AUDIT_LOG (immutable)

---

## 6. Zarządzanie Sekretami (Secret Management)

### Zasada Ogólna

Żaden sekret nie może być przechowywany w pliku `.env` w formie plaintext. Plik `.env` służy wyłącznie do konfiguracji **nie-wrażliwej** (porty, nazwy hostów, poziomy logowania).

### Mechanizm: Docker Secrets

| Sekret | Docker Secret Name | Uwagi |
|---|---|---|
| Klucz prywatny JWT (RS256) | `jwt_private_key` | Dostępny TYLKO dla serwisu Backend (Auth) |
| Klucz publiczny JWT (RS256) | `jwt_public_key` | Dostępny dla: Backend, Worker, SMS Agent |
| Klucz szyfrowania Fernet | `encryption_key` | Dostępny TYLKO dla serwisu Backend |
| Hasło PostgreSQL | `postgres_password` | Dostępny dla: Backend, pg-primary, pg-replica |
| Hasło Redis | `redis_password` | Dostępny dla: Backend, Worker, SMS Agent |
| Hasło RabbitMQ | `rabbitmq_password` | Dostępny dla: Backend, Worker, SMS Agent, Outbox Relay |

### Konfiguracja w Docker Compose

```yaml
secrets:
  jwt_private_key:
    file: ./secrets/jwt_private.pem
  jwt_public_key:
    file: ./secrets/jwt_public.pem
  encryption_key:
    file: ./secrets/encryption.key
  postgres_password:
    file: ./secrets/pg_password.txt
  redis_password:
    file: ./secrets/redis_password.txt
  rabbitmq_password:
    file: ./secrets/rabbitmq_password.txt

services:
  backend:
    secrets:
      - jwt_private_key
      - jwt_public_key
      - encryption_key
      - postgres_password
      - redis_password
      - rabbitmq_password

  satel-worker:
    secrets:
      - jwt_public_key  # tylko weryfikacja
      - redis_password
      - rabbitmq_password

  sms-agent:
    secrets:
      - jwt_public_key  # tylko weryfikacja
      - rabbitmq_password
```

### Odczyt w kodzie Python

```python
def read_secret(name: str) -> str:
    """Odczytaj Docker Secret z /run/secrets/"""
    secret_path = Path(f"/run/secrets/{name}")
    if secret_path.exists():
        return secret_path.read_text().strip()
    # Fallback na .env dla środowiska DEV
    return os.environ.get(name.upper())
```

### Migracja na v2.0+

Dla większych wdrożeń rozważyć **HashiCorp Vault** z:
- Automatyczną rotacją sekretów
- Audit logiem dostępu do sekretów
- Dynamic DB credentials (tymczasowe hasła do PostgreSQL)

### Backup Kluczy

| Element | Procedura |
|---|---|
| `jwt_private.pem` | Backup zaszyfrowany (GPG) na osobnym nośniku. Min. 2 kopie offline. |
| `encryption.key` | **KRYTYCZNY** — utrata = utrata WSZYSTKICH haseł paneli/kamer. Backup jak wyżej + sejf fizyczny. |
| Rotacja | Dokumentuj datę rotacji w `CHANGELOG.md`. Stary klucz archiwizuj (nigdy nie usuwaj przed pełną re-encryption). |

---

## 8. SMS Payload Verification (SEC-08)

> **Problem:** SMS nie ma mechanizmu podpisu cyfrowego. Każdy SMS z zaufanego numeru jest traktowany jak prawdziwy alarm. Fałszywy SMS (SIM spoofing) mógłby wygenerować fałszywy alarm CRITICAL.

### 8.1 Strategia Weryfikacji

| Metoda | Opis | Status |
|---|---|---|
| **Trusted sender list** | Tylko numery z `SMS_PHONE_MAPPING` generują alarmy | ✅ Zaimplementowane |
| **Content hash (Efento API)** | Efento Cloud może wysyłać webhook z hash sha256 payload — porównaj z SMS body | 🔜 v2.0 (wymaga konta Efento API) |
| **Anomaly detection** | Flaguj SMS z częstotliwością > 10/min z jednego numeru | ✅ v1.0 — alert MON-08 |
| **Rate limiting** | Ignoruj > 20 SMS/min globalnie | ✅ v1.0 |

### 8.2 Alarm Spoofing Mitigation

```
SMS Agent flow z weryfikacją:
1. Modem odbiera SMS
2. Sprawdź nadawcę → nie w SMS_PHONE_MAPPING? → IGNORE + log WARNING
3. Rate check → > 10 SMS/min z tego numeru? → THROTTLE + alert WARNING
4. Parse SMS → sms_quality = unparseable? → IGNORE + log WARNING
5. Anomaly check → temperatura spoza zakresu (-50°C do +80°C)? → Flag WARNING
6. Tworzenie event → RabbitMQ
```

### 8.3 Przyszłość (v2.0)

- Webhook z Efento Cloud jako **primary source**, SMS jako **fallback**.
- Podpis HMAC-SHA256 w webhook body pozwali na kryptograficzną weryfikację.
