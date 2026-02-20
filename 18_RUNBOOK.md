# 18_RUNBOOK

## Cel
Procedury operacyjne dla typowych scenariuszy awaryjnych i administracyjnych. Przewodnik "co robić o 3 w nocy".

---

## 1. Scenariusze Awaryjne

---

### RUN-001: Satel Worker — utrata połączenia z centralą

**Alert:** `SatelWorkerDown` (CRITICAL)
**Symptom:** Dashboard: panel X → "DISCONNECTED" od > 5 min

**Procedura:**

| Krok | Akcja | Oczekiwany wynik |
|---|---|---|
| 1 | Sprawdź status kontenera `satel-worker` | Czy działa? Restartował się? |
| 2 | Sprawdź logi: `docker logs satel-worker --tail 100` | Szukaj: "Connection refused", "Timeout", "CRC error" |
| 3 | Sprawdź łączność sieciową z ETHM-1: `telnet {ethm_ip} 10004` | Czy port odpowiada? |
| 4 | Sprawdź czy ktoś inny nie jest podłączony do ETHM-1 (limit: 1 połączenie) | Odłącz inne narzędzia (DLOAD, STAM) |
| 5 | Restart kontenera: `docker restart satel-worker` | Worker powinien się połączyć w ciągu 30s |
| 6 | Jeśli nadal nie działa → sprawdź fizycznie ETHM-1 (zasilanie, kabel sieciowy) | Dioda ETHM-1 miga? |
| 7 | Jeśli ETHM-1 wymaga resetu → wyłącz/włącz zasilanie modułu | Worker połączy się automatycznie |

**Eskalacja:** Jeśli > 30 min bez połączenia → powiadom MASTER, ponieważ brak monitoringu obiektu.

---

### RUN-002: Redis Down

**Alert:** `RedisDown` (CRITICAL)
**Symptom:** Backend loguje "Redis connection refused", dashboard stanu central pusty

**Procedura:**

| Krok | Akcja |
|---|---|
| 1 | Sprawdź kontener: `docker ps \| grep redis` |
| 2 | Restart: `docker restart redis` |
| 3 | Weryfikacja: `docker exec redis redis-cli ping` → "PONG" |
| 4 | Po restarcie: Worker automatycznie odbuduje cache (fresh poll central) |
| 5 | Backend: fallback na direct DB queries (wolniejsze, ale działa bez Redis) |

**Wpływ:** System działa BEZ Redis (degraded mode). Stan live central niedostępny do momentu odbudowy cache. Alarmy nadal przechodzą (przez RabbitMQ → DB).

---

### RUN-003: RabbitMQ Queue Overflow

**Alert:** `QueueBacklog` (WARNING → CRITICAL)
**Symptom:** `rabbitmq_queue_messages > 1000` rośnie zamiast spadać

**Procedura:**

| Krok | Akcja |
|---|---|
| 1 | Sprawdź czy Backend konsumuje: `rabbitmq_consumers > 0`? |
| 2 | Jeśli consumers = 0 → kontener Backend padł. Restart: `docker restart backend` |
| 3 | Jeśli consumers > 0 ale queue rośnie → Backend nie nadąża. Sprawdź logi backendu pod kątem błędów DB |
| 4 | Sprawdź PostgreSQL: `docker exec db psql -U user -c "SELECT count(*) FROM pg_stat_activity"` |
| 5 | Jeśli DB zablokowana → restart DB + Backend |
| 6 | Po rozwiązaniu: kolejka przetworzy zaległe eventy automatycznie |

**Wpływ:** Eventy są buforowane. Żaden event nie jest tracony (RabbitMQ durable). Po rozwiązaniu — wszystkie zaległe alarmy pojawią się u operatora.

---

### RUN-004: Modem SMS nie odbiera

**Alert:** `SMSModemSilent` (WARNING)
**Symptom:** Brak SMSów > 1h (normalnie powinny przychodzić co ~15-30 min jeśli temperatura w normie)

**Procedura:**

| Krok | Akcja |
|---|---|
| 1 | Sprawdź kontener: `docker logs sms-agent --tail 50` |
| 2 | Sprawdź fizycznie: czy modem świeci? Czy ma sygnał? |
| 3 | Sprawdź kartę SIM: czy ma środki / nie jest zablokowana? |
| 4 | Test manualny: wyślij SMS na numer modemu z innego telefonu |
| 5 | Restart kontenera: `docker restart sms-agent` |
| 6 | Jeśli nie pomaga: odłącz/podłącz modem USB fizycznie |
| 7 | Sprawdź w dmesg czy system rozpoznaje urządzenie: `dmesg \| tail` |

**Eskalacja:** Jeśli modem martwy > 2h → użyj zapasowego modemu (na półce).

---

### RUN-005: Database Full

**Alert:** `DiskSpaceLow` (WARNING)
**Symptom:** PostgreSQL loguje "disk full" / "no space left"

**Procedura:**

| Krok | Akcja |
|---|---|
| 1 | Sprawdź disk: `df -h` |
| 2 | Wyczyść stare logi Dockera: `docker system prune --volumes` (UWAGA: nie usuwaj wolumenów bazy!) |
| 3 | Sprawdź rozmiar bazy: `docker exec db psql -U user -c "SELECT pg_size_pretty(pg_database_size('stam'))"` |
| 4 | Jeśli tabela EVENTS za duża → rozważ archiwizację starych eventów (> 90 dni) |
| 5 | Jeśli logi Dockera za duże → skonfiguruj log rotation w `daemon.json` |

---

## 2. Procedury Administracyjne

---

### RUN-010: Onboarding nowego obiektu

| Krok | Kto | Akcja |
|---|---|---|
| 1 | ADMIN | Dodaj obiekt w aplikacji (nazwa, adres, kontakty) |
| 2 | ADMIN | Dodaj centralę do obiektu (panel_id, model, IP ETHM-1) |
| 3 | ADMIN | Skonfiguruj ETHM-1 — włącz integrację, ustaw port 10004 |
| 4 | ADMIN | Dodaj konfigurację centrali do Satel Worker (config file / DB) |
| 5 | ADMIN | Restart Satel Worker → Worker połączy się z nową centralą |
| 6 | ADMIN | Sprawdź na dashboardzie: status "CONNECTED", stan stref widoczny |
| 7 | ADMIN | Test: uzbroj/rozbrój strefę na obiekcie → sprawdź czy event pojawił się w systemie |

---

### RUN-011: Backup Restore

| Krok | Akcja |
|---|---|
| 1 | Zatrzymaj kontenery: `docker compose down` |
| 2 | Zlokalizuj backup: `ls -la /backups/pg_dumps/` |
| 3 | Restore: `docker exec -i db psql -U user -d stam < backup_YYYYMMDD.sql` |
| 4 | Start usług: `docker compose up -d` |
| 5 | Weryfikacja: zaloguj się, sprawdź czy dane są aktualne |
| 6 | Satel Worker: automatycznie reconnect do central |
| 7 | Redis: automatycznie odbuduje cache |

---

### RUN-012: Deploy nowej wersji

| Krok | Akcja |
|---|---|
| 1 | Sprawdź czy CI/CD green na main |
| 2 | SSH do serwera PROD |
| 3 | Pull najnowsze obrazy: `docker compose pull` |
| 4 | Restart z nową wersją: `docker compose up -d --force-recreate` |
| 5 | Sprawdź logi: `docker compose logs -f --tail 50` |
| 6 | Smoke test: zaloguj się, sprawdź listę alarmów, stan central |
| 7 | Jeśli problemy → rollback: `docker compose up -d --force-recreate` z image digest z poprzedniej wersji |

---

### RUN-013: Dodanie użytkownika

| Krok | Kto | Akcja |
|---|---|---|
| 1 | ADMIN | Panel Admin → Użytkownicy → Dodaj nowego |
| 2 | ADMIN | Wypełnij: email, hasło tymczasowe, rola (OPERATOR/TECHNICIAN/etc.) |
| 3 | ADMIN | Jeśli FIELD_WORKER/TECHNICIAN → przypisz obiekty (sekcja „Przypisania obiektów") |
| 4 | ADMIN | Wymuś zmianę hasła przy pierwszym logowaniu |
| 5 | ADMIN | Weryfikacja: nowy użytkownik loguje się i widzi swój zakres danych |

---

### RUN-014: Outbox Relay zablokowany (alarmy nie docierają)

**Symptom:** Alert `OutboxStuck` — `outbox_pending_count > 100` przez > 5 min. Alarmy widoczne w PostgreSQL, ale operatorzy ich nie widzą.

> [!NOTE]
> Od v8.0 ruch Outbox Relay jest rozłożony na **2 instancje** (`outbox-relay-1`, `outbox-relay-2`). Obie instancje muszą być sprawdzone.

```
Diagnoza:
├── Sprawdź obie instancje relay:
│   ├── docker compose ps outbox-relay-1 outbox-relay-2
│   ├── Obie żyją? → Sprawdź logi obu instancji
│   └── Jedna padła? → Restart: docker compose up -d outbox-relay-{N}
│       (druga instancja kontynuuje przetwarzanie — brak utraty)
│   │         ├── "Connection refused" do RabbitMQ → idź do RUN-003
│   │         ├── "Timeout" publishing → RabbitMQ przeciążony? → RUN-017
│   │         └── Unhandled exception → restart: docker compose restart outbox-relay
│   └── NIE → docker compose up -d outbox-relay
│
├── Po restarcie: obserwuj outbox_pending_count (powinien spadać)
│   ├── Spada → OK, obserwuj 15 min
│   └── Nie spada → sprawdź RabbitMQ management: http://host:15672
│       └── Jeśli dead letter queue niepusta → ręcznie republish lub eskaluj
│
└── Szacowany czas: 5-15 min
```

---

### RUN-015: WebSocket buffer overflow (operatorzy tracą eventy)

**Symptom:** Alert `WSBufferFull` lub `ws_replay_overflow_count > 0`. Operatorzy zgłaszają "brak nowych alarmów".

```
Diagnoza:
├── Skala problemu: ile sesji WS dotknęło overflow?
│   └── Sprawdź metric: ws_replay_overflow_count (per instance)
│
├── Przyczyna:
│   ├── Operator offline > 2h → Tier 1 (Redis, 5000 eventów) i Tier 2 (PostgreSQL, 2h) wyczerpane
│   │   └── Normalne zachowanie. Klient po reconnect dostanie replay_overflow
│   │       z retry_after_ms (thundering herd protection).
│   │       Klient czeka retry_after_ms i przeładuje dane z REST API.
│   ├── Redis restart → Tier 2 (PostgreSQL fallback) przejmuje → brak overflow
│   │   └── Sprawdź: czy outbox.sequence_id rośnie normalnie
│   ├── Burst alarmów (np. 500 obiektów naraz) → chwilowy spike
│   │   └── Sprawdź: czy events.processing queue rośnie? → QueueBacklog
│   └── Backend memory leak → WS sesje nie zamykane
│       └── docker compose restart backend-1 (potem backend-2)
│
└── Szacowany czas: 5-10 min
```

---

### RUN-016: Auto-Arm failure (centrala nie uzbroiła się wg harmonogramu)

**Symptom:** Alert `satel_auto_arm_failure` lub raport poranny: obiekt niezabezpieczony w nocy.

```
Diagnoza:
├── Czy komenda została wysłana?
│   └── SELECT * FROM satel_commands 
│       WHERE source='AUTO_ARM' AND panel_id='...' ORDER BY created_at DESC LIMIT 5
│       ├── status=ACK → centrala potwierdziła, ale strefa otwarta?
│       │   └── Sprawdź: czy strefy zamknięte? (violation na wejściu)
│       │       └── Jeśli tak → zadzwoń do obiektu (drzwi otwarte?)
│       ├── status=NACK → centrala odrzuciła (np. strefa w alarmie)
│       │   └── Sprawdź event log centrali
│       ├── status=TIMEOUT → ETHM-1 nie odpowiedziało
│       │   └── Sprawdź Worker logs + satel_connection_status
│       └── Brak komendy → Auto-Arm scheduler nie odpalił
│           └── Sprawdź: czy scheduling cron działa? docker compose logs backend | grep auto_arm
│
├── Interwencja:
│   └── Ręczne uzbrojenie: POST /api/satel/command {command_type: "ARM_STAY", panel_id: "..."}
│
└── Szacowany czas: 5-20 min
```

---

### RUN-017: RabbitMQ disk alarm (kolejka zablokowana)

**Symptom:** Alert `RabbitMQ disk alarm` lub `rabbitmq_queue_messages` stale rośnie ale `consumers > 0`.

```
Diagnoza:
├── RabbitMQ Management UI: http://host:15672
│   ├── Zakładka "Overview" → czy jest "Disk space alarm"?
│   │   ├── TAK → dysk pełny
│   │   │   ├── Sprawdź: df -h /var/lib/rabbitmq
│   │   │   ├── Wyczyść stare logi: find /var/log/rabbitmq -mtime +7 -delete
│   │   │   ├── Jeśli dead letter queue duża → purge: rabbitmqadmin purge queue name=events.dead
│   │   │   └── Rozważ zwiększenie dysku (docker volume resize)
│   │   └── NIE → sprawdź consumers
│   │       ├── Consumers = 0 → Backend nie konsumuje → restart backend
│   │       └── Consumers > 0 ale messages rośnie → Backend za wolny
│   │           └── Sprawdź: pg_connections_active, response times
│   │               → Prawdopodobnie PostgreSQL bottleneck
│
└── Szacowany czas: 10-30 min
```

---

### RUN-018: Deploy z Rollback (szczegółowa procedura)

**Wymaganie:** Każdy deploy musi mieć zapisany image digest umożliwiający natychmiastowy rollback.

| Krok | Czas | Akcja |
|---|---|---|
| 1 | 0 min | Zapisz aktualny stan: `docker compose images > /var/log/deploy/pre_deploy_$(date +%Y%m%d_%H%M).txt` |
| 2 | 1 min | Pull nowe obrazy: `docker compose pull` |
| 3 | 2 min | Rolling restart backend-1: `docker compose up -d --no-deps backend-1` |
| 4 | 3 min | Poczekaj na `/readyz` → 200 OK (max 30s) |
| 5 | 4 min | Rolling restart backend-2: `docker compose up -d --no-deps backend-2` |
| 6 | 5 min | Restart Worker: `docker compose up -d --no-deps satel-worker` |
| 7 | 6 min | Smoke test: login, lista alarmów, stan central, WebSocket push |
| 8 | 7 min | ✅ Jeśli OK → gotowe. Zapisz: `docker compose images > /var/log/deploy/post_deploy_$(date +%Y%m%d_%H%M).txt` |

**Rollback (jeśli smoke test failed):**

| Krok | Akcja |
|---|---|
| 1 | Odczytaj image digest z `pre_deploy_*.txt` |
| 2 | `docker compose up -d --force-recreate` z pinowanymi digestami |
| 3 | Smoke test rollbacku |
| 4 | Loguj incydent: kto, co, dlaczego rollback |

---

## 3. Blameless Postmortem (RUN-05 Template)

> Po każdym incydencie CRITICAL stosuj ten szablon. **„Blameless"** = szukamy przyczyn systemowych, nie winnych ludzi.

```markdown
# Postmortem: [Tytuł incydentu]

## Podsumowanie
- **Data/czas:** YYYY-MM-DD HH:MM UTC
- **Czas trwania:** X min
- **Impact:** Co było niedostępne? Ilu operatorów dotkniętych?
- **Severity:** CRITICAL / WARNING

## Timeline
| Czas (UTC) | Zdarzenie |
|---|---|
| HH:MM | Pierwszy alert: [nazwa alertu] |
| HH:MM | On-call acknowledged |
| HH:MM | Diagnoza: [co ustalono] |
| HH:MM | Mitygacja: [co zrobiono] |
| HH:MM | Potwierdzenie rozwiązania |

## Root Cause
[Opis głównej przyczyny — technicznej, nie ludzkiej]

## Co zadziałało dobrze?
- [np. alert wykrył problem w < 5 min]

## Co nie zadziałało?
- [np. runbook nie obejmował tego scenariusza]

## Action Items
| # | Akcja | Priorytet | Odpowiedzialny | Termin |
|---|---|---|---|---|
| 1 | ... | CRITICAL | ... | ... |
| 2 | ... | HIGH | ... | ... |
```

---

## 4. Kontakty Eskalacyjne

> **Uwaga:** Dla pełnej konfiguracji rotacji dyżurów — patrz `17_MONITORING.md`, sekcja 10.

| Priorytet | Czas reakcji | Do kogo | Kanał |
|---|---|---|---|
| CRITICAL (system down) | < 10 min | On-call primary (PagerDuty) | SMS + Telefon (auto) |
| CRITICAL (eskalacja L2) | < 30 min | On-call backup | SMS + Telefon |
| CRITICAL (eskalacja L3) | > 30 min | MASTER (właściciel) | Telefon bezpośredni |
| WARNING (degradacja) | < 2h | ADMIN | Email + Push |
| INFO (obserwacja) | Następny dzień roboczy | ADMIN | Dashboard |

---

## 5. Nowe Metryki i Alerty (v8.0 Remediation)

| Metryka / Alert | Typ | Opis | Severity |
|---|---|---|---|
| `claiming_timeout_reverts_total` | Counter | Alarmy zrevertowane z CLAIMING do NEW (CLAIMING Reaper Job) | INFO (>10/h → WARNING) |
| `outbox_relay_active_instances` | Gauge | Liczba aktywnych instancji relay | CRITICAL jeśli <1 przez >30s |
| `ws_replay_tier2_queries_total` | Counter | Czerpanie z PostgreSQL zamiast Redis (Tier 2 Catch-Up) | INFO (wzrost → Redis problem) |
| `ws_replay_overflow_count` | Counter | Klienci otrzymujący replay_overflow (Tier 3) | WARNING jeśli >5/min |
| `stale_alarm_report_bundles` | Gauge | Liczba CRITICAL Bundle otwartych >24h | CRITICAL jeśli >0 |
| `intent_sync_rejected_total` | Counter | Odrzucone intencje offline (INTENT_REJECTED) | INFO |
| `sms_raw_archive_access_total` | Counter | Dostępy do surowych treści SMS | WARNING (każdy dostęp logowany) |
