# AI Security Platform – Agent Source of Truth Pack

Ten zestaw plików .md jest jedynym źródłem prawdy (single source of truth) dla agenta AI oraz zespołu developerskiego. Agent MA trzymać się tych dokumentów przy podejmowaniu decyzji architektonicznych, projektowych i implementacyjnych.

---

## 1. /01_PROJECT_OVERVIEW.md
**Cel:** high‑level opis projektu dla kontekstu
Zawiera:
- Cel projektu
- Zakres systemu
- Co NIE jest w zakresie (np. brak integracji live z CCTV)
- Definicje pojęć (obiekt, centrala, alarm, event, bundle)

---

## 2. /02_ARCHITECTURE.md
**Cel:** twarda architektura systemu
Zawiera:
- Warstwy systemu (Frontend / Backend / **Satel Worker** / Messaging / Storage)
- Odpowiedzialności każdej warstwy
- Zależności
- Tryby pracy (normal / test / serwis / awaryjny)

---

## 3. /03_FUNCTIONAL_MODULES.md
**Cel:** lista modułów + priorytety
Zawiera tabelę:
Moduł | Opis | Priorytet (MVP / v1.0 / v2.0 / Nice‑to‑have)

Moduły:
- Obiekty
- Centrale (Satel) — **integracja TCP/IP przez ETHM-1**
- Alarmy & Bundling (MVP)
- SMS Temperatury (Efento, Bluelog)
- Użytkownicy & Role
- Raporty
- Offline / Mobile cache
- Audyt i logi

---

## 4. /04_DATA_MODEL_ERD.md
**Cel:** model danych jako kontrakt
Zawiera:
- Opis encji
- Mermaid ERD
- Kluczowe relacje

Encje:
- OBJECTS
- PANELS
- ZONES
- TEMP_SENSORS
- SMS_PHONE_MAPPING
- EVENTS (surowe zdarzenia — zarówno z SATEL jak i SMS)
- BUNDLE_ALARMS
- USERS
- ROLES
- AUDIT_LOG
- FILES
- NOTIFICATIONS

---

## 5. /05_ALARM_LIFECYCLE.md
**Cel:** święta prawda o alarmach
Zawiera:
- Stany alarmu
- Event ID
- Bundling
- Spam detection
- Eskalacje
- Opóźnione alarmy
- Tryb testowy / serwisowy

Diagram stanów (Mermaid) — **KANONICZNE**:
- NEW → IN_PROGRESS → ACK → RESOLVED → CLOSED

---

## 6. /06_INTEGRATIONS.md
**Cel:** integracje zewnętrzne
Osobne sekcje:

### SATEL (ETHM-1 TCP/IP)
- Bezpośrednie połączenie TCP/IP z modułem ETHM-1 Plus (port 10004)
- Oficjalny protokół integracji (ramki binarne, CRC-16)
- Satel Worker jako dedykowany mikroserwis
- Polling stanów + odbiór zdarzeń real-time
- Sterowanie centralą Arm/Disarm (v2.0)

### SMS (Efento / Bluelog)
- Format SMS (konkretne wzory z produkcji)
- Start / End alarmu temp
- Dedup
- GSM modem
- Fallback SMS API

### Google Maps
- Geokodowanie
- Pinezki
- Ręczne przesuwanie

---

## 7. /07_TECH_STACK.md
**Cel:** technologia jako kontrakt
Tabela:
Warstwa | Technologia | Status

- Frontend: Flutter (Windows + Android)
- Backend: Python + FastAPI
- **Satel Integration: Python Asyncio (TCP/IP)**
- Realtime: WebSocket / SSE
- Queue: RabbitMQ
- DB: PostgreSQL
- **State Cache: Redis (stan live central)**
- Cache lokalny: SQLite
- Storage: lokalny + NAS backup
- Maps: Google Maps API
- SMS: GSM modem
- Monitoring: Prometheus + Grafana

---

## 8. /08_SECURITY_AND_ROLES.md
**Cel:** bezpieczeństwo i role
Zawiera:
- Role (KANONICZNE): SYSTEM, MASTER, ADMIN, OPERATOR, TECHNICIAN, FIELD_WORKER
- Dostęp do haseł (Timed Secret Reveal)
- PIN + biometria
- Audyt
- Retencja danych

---

## 9. /09_HA_RTO_RPO.md
**Cel:** dostępność
Tabela:
Komponent | RTO | RPO | Uwagi

- Backend
- PostgreSQL
- RabbitMQ
- **Satel Worker**
- **Redis**
- SMS modem
- Storage

Failover auto, backup nocny, archiwizacja.

---

## 10. /10_API_HIGH_LEVEL.md
**Cel:** kontrakt API
Zawiera:
- Lista endpointów (prefix: `/api/`)
- Opis request/response (high‑level)

Przykłady:
- POST /api/auth/login
- GET /api/objects
- GET /api/objects/{id}
- GET /api/alarms
- POST /api/alarms/{id}/claim
- POST /api/alarms/{id}/ack
- POST /api/alarms/{id}/resolve
- POST /api/alarms/{id}/close

---

## 11. /11_DEPLOYMENT.md
**Cel:** jak to stoi na serwerze
Zawiera:
- 6 kontenerów: backend, satel-worker, db, redis, rabbitmq, sms-agent
- Środowiska (DEV / STAGE / PROD)
- VPN
- Backup
- Aktualizacje (CI/CD)
- Kubernetes opcjonalnie (v2.0+)

---

## 12. /12_DECISIONS_AND_OPEN.md
**Cel:** rejestr decyzji architektonicznych (ADR)
Zawiera:
- ADR-001 do ADR-007 (podjęte decyzje)
- Otwarte pytania (do rozstrzygnięcia)

---

## 13. /13_EVENT_SCHEMAS.md
**Cel:** kontrakt między mikroserwisami
Zawiera:
- Topologia RabbitMQ (exchanges, queues, routing keys)
- Schemat wiadomości: Satel Alarm Event
- Schemat wiadomości: SMS Temperature Event
- Schemat wiadomości: Satel Command (v2.0)
- Redis Live State (klucze, TTL)
- Algorytm deduplikacji (dedup_key)

---

## 14. /14_SATEL_COMMANDS.md
**Cel:** referencyjna tabela komend ETHM-1
Zawiera:
- Komendy polling MVP (0x00-0x20, 0x7F)
- Komendy sterujące v2.0 (0x80-0x89)
- Format ramki binarnej (0xFE 0xFE, CRC-16)
- Mapowanie event codes → typy alarmów
- Keep-Alive i Reconnect logic
- Ograniczenia ETHM-1

---

## 15. /15_USER_STORIES_MVP.md
**Cel:** definicja "done" dla MVP
Zawiera:
- 15 User Stories w 6 Epicach
- Kryteria akceptacji (AC) dla każdego story
- Definition of Done (globalna)

---

## 16. /16_TESTING_STRATEGY.md
**Cel:** co, jak i kiedy testujemy
Zawiera:
- Piramida testów (unit/integration/E2E/performance/security)
- Fixture SMSów (Efento, Bluelog)
- Symulator centrali (mock TCP server)
- CI/CD Quality Gates

---

## 17. /17_MONITORING.md
**Cel:** monitoring i alerty produkcyjne
Zawiera:
- Strategia logowania (Structured JSON)
- Metryki biznesowe (alarm count, response time)
- Metryki infrastrukturalne (per komponent)
- Dashboardy Grafana (3 dashboardy)
- Reguły alertowania (CRITICAL/WARNING/INFO)

---

## 18. /18_RUNBOOK.md
**Cel:** procedury operacyjne
Zawiera:
- 5 scenariuszy awaryjnych (Worker down, Redis down, Queue overflow, Modem dead, Disk full)
- 4 procedury administracyjne (onboarding obiektu, backup restore, deploy, user management)
- Kontakty eskalacyjne

---

## Zasady dla agenta AI

1. Agent MUSI traktować te pliki jako źródło prawdy.
2. Agent NIE może wprowadzać technologii spoza 07_TECH_STACK bez wyraźnej decyzji.
3. Agent NIE może zmieniać modelu alarmów bez aktualizacji 05_ALARM_LIFECYCLE.md.
4. Każda zmiana architektury = aktualizacja 02_ARCHITECTURE.md.
5. Każda nowa encja = aktualizacja 04_DATA_MODEL_ERD.md.

---

Ten pakiet jest fundamentem projektu i ma zapobiegać dryfowi koncepcji.
