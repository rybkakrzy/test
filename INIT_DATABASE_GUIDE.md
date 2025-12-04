# D2 Creator Database - Complete Initialization Guide

## Przegląd

Plik `init-database-complete.sql` zawiera kompletny schemat bazy danych D2 Creator z wszystkimi zastosowanymi migracjami. Jest to jednoplikowe rozwiązanie do szybkiego postawienia całej bazy danych.

## Co zawiera ten plik?

Ten skrypt SQL zawiera:

### 1. **Wszystkie tabele**
- `tenants` - Zespoły/organizacje (z kolumną `created_by`)
- `tenant_members` - Członkowie zespołów
- `users` - Użytkownicy systemu
- `projects` - Projekty dokumentów (z opcjonalnym `tenant_id`)
- `project_versions` - Wersje projektów (status jako VARCHAR)
- `files` - Pliki szablonów
- `parse_results` - Wyniki parsowania (zaktualizowana struktura)
- `workers` - Konfiguracje mapowania (nowa tabela)
- `mappings` - Mapowania pól (zaktualizowane nazwy kolumn)
- `approvals` - Zatwierdzenia (status jako VARCHAR)
- `audit_log` - Log audytu
- `app_settings` - Ustawienia aplikacji

### 2. **Wszystkie zastosowane migracje**
- ✅ 001: Complete schema
- ✅ 002: Fix schema alignment (renamed columns in mappings, parse_results, approvals)
- ✅ 003: Change status to VARCHAR (removed ENUMs)
- ✅ 004: Fix timestamp columns (TIMESTAMP WITH TIME ZONE)
- ✅ 005: Add tenant_members table
- ✅ 006: Migrate projects to tenants (schema only)
- ✅ 007: Add workers table
- ✅ 008: Make tenant_id optional in projects

### 3. **Indeksy i ograniczenia**
- Wszystkie klucze obce (foreign keys)
- Indeksy optymalizacyjne
- Indeksy GIN dla kolumn JSONB
- Indeksy z filtrem WHERE dla kolumn nullable

### 4. **Funkcje i triggery**
- Funkcja `update_updated_at_column()`
- Triggery automatycznej aktualizacji `updated_at`

### 5. **Dane początkowe**
- Domyślne tenants (NL, PL, DE)
- Domyślni użytkownicy (admin, testuser)
- Domyślne ustawienia aplikacji

## Jak użyć?

### Opcja 1: PostgreSQL (psql)

```bash
# Utwórz bazę danych
createdb d2creatordb

# Wykonaj skrypt
psql -d d2creatordb -f init-database-complete.sql
```

### Opcja 2: pgAdmin

1. Otwórz pgAdmin
2. Utwórz nową bazę danych `d2creatordb`
3. Kliknij prawym przyciskiem na bazę → Query Tool
4. Otwórz plik `init-database-complete.sql`
5. Wykonaj (F5)

### Opcja 3: Docker PostgreSQL

```bash
# Skopiuj plik do kontenera
docker cp init-database-complete.sql postgres-container:/tmp/

# Wykonaj w kontenerze
docker exec -it postgres-container psql -U postgres -d d2creatordb -f /tmp/init-database-complete.sql
```

### Opcja 4: Podman PostgreSQL

#### A. Szybka instalacja - użyj gotowego skryptu

```bash
cd d2creatordb
setup-postgres.bat
```

Skrypt `setup-postgres.bat` automatycznie:
1. Zatrzyma i usunie stary kontener (jeśli istnieje)
2. Utworzy volume dla trwałych danych
3. Uruchomi nowy kontener PostgreSQL
4. Utworzy bazę danych `d2creatordb`
5. Wykona skrypt inicjalizacyjny
6. Pokaże utworzone tabele i dane początkowe

#### B. Ręczna instalacja krok po kroku

**Krok 1: Zatrzymaj i usuń stary kontener (jeśli istnieje)**

```bash
podman stop postgres
podman rm postgres
```

**Krok 2: Utwórz volume dla danych**

```bash
podman volume create postgres_data
```

**Krok 3: Uruchom kontener PostgreSQL**

```bash
podman run -d ^
  --name postgres ^
  -e POSTGRES_PASSWORD=postgres ^
  -e POSTGRES_USER=postgres ^
  -e POSTGRES_DB=postgres ^
  -p 5432:5432 ^
  -v postgres_data:/var/lib/postgresql/data ^
  postgres:15
```

**Parametry:**
- `-d` - uruchom w tle (detached mode)
- `--name postgres` - nazwa kontenera
- `-e POSTGRES_PASSWORD=postgres` - hasło administratora
- `-e POSTGRES_USER=postgres` - nazwa użytkownika
- `-p 5432:5432` - mapowanie portu (host:kontener)
- `-v postgres_data:/var/lib/postgresql/data` - trwałe dane
- `postgres:15` - obraz PostgreSQL w wersji 15

**Krok 4: Sprawdź czy kontener działa**

```bash
podman ps
podman logs postgres
```

**Krok 5: Poczekaj na uruchomienie (5-10 sekund)**

```bash
timeout /t 10
```

**Krok 6: Utwórz bazę danych**

```bash
podman exec -it postgres psql -U postgres -c "CREATE DATABASE d2creatordb;"
```

**Krok 7: Wykonaj skrypt inicjalizacyjny**

Metoda 1 - przez stdin (zalecana):
```bash
podman exec -i postgres psql -U postgres -d d2creatordb < init-database-complete.sql
```

Metoda 2 - kopiując plik do kontenera:
```bash
podman cp init-database-complete.sql postgres:/tmp/
podman exec -it postgres psql -U postgres -d d2creatordb -f /tmp/init-database-complete.sql
```

**Krok 8: Zweryfikuj instalację**

```bash
# Sprawdź tabele
podman exec -it postgres psql -U postgres -d d2creatordb -c "\dt"

# Sprawdź dane początkowe
podman exec -it postgres psql -U postgres -d d2creatordb -c "SELECT * FROM tenants;"
podman exec -it postgres psql -U postgres -d d2creatordb -c "SELECT * FROM users;"
```

#### C. Połączenie z bazą danych

Po pomyślnej instalacji możesz się połączyć używając:

```
Host: localhost
Port: 5432
Database: d2creatordb
Username: postgres
Password: postgres
```

**Z poziomu aplikacji .NET:**
```
Host=localhost;Port=5432;Database=d2creatordb;Username=postgres;Password=postgres
```

**Interaktywna sesja psql:**
```bash
podman exec -it postgres psql -U postgres -d d2creatordb
```

#### D. Zarządzanie kontenerem

**Zatrzymaj kontener:**
```bash
podman stop postgres
```

**Uruchom ponownie:**
```bash
podman start postgres
```

**Restart:**
```bash
podman restart postgres
```

**Usuń kontener (dane pozostaną w volume):**
```bash
podman stop postgres
podman rm postgres
```

**Usuń kontener I dane (UWAGA: trwała utrata danych!):**
```bash
podman stop postgres
podman rm postgres
podman volume rm postgres_data
```

### Opcja 4: Z poziomu .NET aplikacji

```bash
# Użyj Entity Framework lub ADO.NET do wykonania skryptu
# Przykład z psql command:
psql "Host=localhost;Database=d2creatordb;Username=postgres;Password=yourpassword" -f init-database-complete.sql
```

## Weryfikacja instalacji

Po wykonaniu skryptu, sprawdź czy wszystko działa:

```sql
-- Sprawdź liczbę tabel
SELECT COUNT(*) FROM information_schema.tables 
WHERE table_schema = 'public';
-- Powinno być: 12 tabel

-- Sprawdź dane początkowe
SELECT * FROM tenants;
SELECT * FROM users;
SELECT * FROM app_settings;

-- Sprawdź indeksy
SELECT tablename, indexname 
FROM pg_indexes 
WHERE schemaname = 'public'
ORDER BY tablename, indexname;
```

## Kluczowe zmiany względem oryginalnego schematu

1. **ENUM → VARCHAR**: Statusy w tabelach `project_versions` i `approvals` są teraz VARCHAR(50)
2. **TIMESTAMP → TIMESTAMP WITH TIME ZONE**: Wszystkie kolumny dat mają strefę czasową
3. **Tenant opcjonalny**: `projects.tenant_id` może być NULL (projekty osobiste)
4. **Nowa tabela workers**: Przechowuje konfigurację mapowania pól
5. **Tenant members**: Nowa tabela do zarządzania członkami zespołów
6. **Zaktualizowane nazwy kolumn**: 
   - `mappings.field_name` → `tag_name`
   - `mappings.mapper_type` → `mapping_type`
   - `mappings.mapping_config` → `mapping_json`
   - `approvals.comments` → `comment`

## Uwagi

- **Usunięte ENUMy**: Dla lepszej kompatybilności z różnymi ORM-ami
- **JSONB**: Używane dla elastycznych danych (metadata, step_data, config_json)
- **UUID**: Wszystkie ID używają UUID v4
- **Soft delete**: Zamiast usuwania, ustaw `is_active = false` gdzie dostępne
- **Circular reference**: Tabele `projects` i `project_versions` mają wzajemne referencje

## Rozwiązywanie problemów

### Problem: Extension uuid-ossp nie istnieje
```sql
-- Upewnij się że masz uprawnienia do tworzenia extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

### Problem: Baza już istnieje
```sql
-- Usuń istniejącą bazę (UWAGA: usuwa wszystkie dane!)
DROP DATABASE IF EXISTS d2creatordb;
CREATE DATABASE d2creatordb;
```

### Problem: Konflikty podczas INSERT
Skrypt używa `ON CONFLICT DO NOTHING` dla danych początkowych, więc można go uruchomić wielokrotnie bez błędów.

## Aktualizacje w przyszłości

Jeśli potrzebujesz zaktualizować istniejącą bazę:
1. **Użyj Liquibase** z plikami changelog (zalecane dla production)
2. **Lub** wykonaj ręcznie nowe pliki migracji z folderu `liquibase/changelog/`

## Support

W razie problemów sprawdź:
- Logi PostgreSQL
- Uprawnienia użytkownika bazy danych
- Wersję PostgreSQL (zalecana: 13+)
- Czy extension uuid-ossp jest dostępne

## Historia zmian

- **2025-12-04**: Utworzenie pliku inicjalizacyjnego z wszystkimi migracjami 001-008
