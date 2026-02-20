# 01_PROJECT_OVERVIEW

## Cel projektu
Stworzenie centralnej, nowoczesnej aplikacji do zarządzania bezpieczeństwem fizycznym firmy: alarmy Satel (bezpośrednia integracja TCP/IP z ETHM-1), dane obiektów, dokumentacja techniczna, monitoring zdarzeń (w tym SMS temperatur z Efento i Bluelog), raportowanie oraz mobilny dostęp. System ma w dłuższej perspektywie zastąpić STAM jako główne narzędzie operacyjne.

## Zakres
System obejmuje:
- Zarządzanie obiektami i ich danymi technicznymi
- Bezpośrednią integrację z centralami Satel przez TCP/IP (protokół ETHM-1, port 10004)
- Obsługę alarmów (SSWiN + temperatury z SMS: Efento, Bluelog)
- Mapę obiektów z geokodowaniem
- Audyt, logowanie, raporty
- Aplikację desktop + mobile

## Co system NIE robi
- Brak integracji live z CCTV
- Brak podglądu kamer
- Brak sterowania rejestratorami CCTV
- W bazie są tylko dane opisowe CCTV

## Główne założenia
- System krytyczny operacyjnie (monitoring)
- HA 99.9%
- Architektura event‑driven
- Push (nie polling)
- Źródło prawdy: PostgreSQL (historia, konfiguracja) + Redis (stan live central)
- Offline tylko do przeglądania danych (mobile)

## Użytkownicy
- SYSTEM (rola techniczna — Workery, parsery)
- MASTER (właściciel)
- ADMIN
- OPERATOR (centrum monitorowania)
- TECHNICIAN (serwisant)
- FIELD_WORKER (audytor / tester obiektowy)

## Status
Blueprint zatwierdzony. Projekt w fazie projektowej. Brak implementacji produkcyjnej.

Ten dokument jest nadrzędny i definiuje cel całego systemu.
