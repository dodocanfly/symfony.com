# 📘 Komendy Symfony CLI – Przewodnik

## 📦 Tworzenie i zarządzanie projektami

- `local:new`, `new`
 Tworzy nowy projekt Symfon.

- `project:init`, `init`
 Inicjalizuje nowy projekt przy użyciu szablonó.

- `book:check-requirements`, `book:check`
 Sprawdza, czy lokalne środowisko spełnia wymagania książki "Symfony: The Fast Track.

- `book:checkout`
 Pobiera konkretny krok z repozytorium książki "Symfony: The Fast Track.

## 🧪 Sprawdzanie wymagań i bezpieczeństwa

- `local:check:requirements`, `check:requirements`, `check:req`    Sprawdza wymagania do uruchomienia Symfony i sugeruje optymalizacje PP.

- `local:check:security`, `security:check`, `check:security`, `local:security:check`    Sprawdza podatności w zależnościach projeku.

## 🖥️ Serwer lokalny i proxy

- `local:server:start`, `server:start`, `serve`
  Uruchamia lokalny serwer WW.

- `local:server:stop`, `server:stop`
  Zatrzymuje lokalny serwer WW.

- `local:server:status`, `server:status`
  Wyświetla status lokalnego serwera WW.

- `local:server:log`, `server:log`
  Wyświetla logi lokalnego serwera WW.

- `local:server:prod`, `server:prod`
  Przełącza projekt na środowisko produkcyjne Symfny.

- `local:server:ca:install`, `server:ca:install`
  Tworzy lokalny urząd certyfikacji (CA) do obsługi HTPS.

- `local:server:ca:uninstall`, `server:ca:uninstall`
  Usuwa lokalny urząd certyfikacji (A).

- `local:proxy:start`, `proxy:start`
  Uruchamia lokalny serwer proxy (obsługa lokalnych domn).

- `local:proxy:stop`, `proxy:stop`
  Zatrzymuje lokalny serwer prxy.

- `local:proxy:status`, `proxy:status`
  Wyświetla status lokalnego serwera prxy.

- `local:proxy:domain:attach`, `proxy:domain:attach`
  Przypisuje lokalną domenę do prxy.

- `local:proxy:domain:detach`, `proxy:domain:detach`
  Odłącza domeny od prxy.

- `local:proxy:tld`, `proxy:tld`, `proxy:change:tld`
  Wyświetla lub zmienia TLD dla prxy.

- `local:proxy:url`, `proxy:url`
  Pobiera URL lokalnego serwera prxy.

## 🔧 Uruchamianie programów i eksport zmiennych

- `local:run`, `run`
  Uruchamia program z ustawionymi zmiennymi środowiskowymi w zależności od bieżącego kontestu.

- `var:export`
  Eksportuje zmienne środowiskowe w zależności od bieżącego kontestu.

- `local:var:expose-from-tunnel`, `var:expose-from-tunnel`
  Udostępnia lokalnie zmienne środowiskowe usługi tunelwej.

## 🌐 Otwieranie usług lokalnych w przeglądarce

- `open:local`
  Otwiera lokalny projekt w przegląarce.

- `open:local:rabbitmq`
  Otwiera interfejs zarządzania RabbitMQ lokalnego projektu w przegląarce.

- `open:local:service`
  Otwiera interfejs WWW lokalnej usługi w przegląarce.

- `open:local:webmail`
  Otwiera interfejs webmail lokalnego projektu w przegląarce.

## 🛠️ Komendy ogólne

- `self:completion`, `completio`
  Generuje skrypt uzupełniania dla bieżącej pwłoki.

- `self:help`, `help`, `lis`
  Wyświetla pomoc dla komendy lub kategorii omend.

- `self:version`, `versio`
  Wyświetla wersję aplkacji.

## ☁️ SymfonyCloud – zarządzanie projektem w chmurze

### 🔧 SymfonyCloud – konfiguracja i zarządzanie

- `cloud:login`, `login`
  Loguje się do konta SymfonyCloud.

- `cloud:logout`, `logout`
  Wylogowuje z konta SymfonyCloud.

- `cloud:whoami`, `whoami`
  Wyświetla aktualnie zalogowanego użytkownika.

- `cloud:ssh`, `ssh`
  Łączy się z instancją w chmurze przez SSH.

- `cloud:organization:list`, `organization:list`
  Wyświetla listę organizacji SymfonyCloud.

- `cloud:organization:create`, `organization:create`
  Tworzy nową organizację SymfonyCloud.

- `cloud:organization:delete`, `organization:delete`
  Usuwa organizację SymfonyCloud.

- `cloud:organization:update`, `organization:update`
  Aktualizuje organizację SymfonyCloud.

### 🌍 Zarządzanie projektami i środowiskami

- `cloud:project:list`, `project:list`
  Wyświetla listę projektów SymfonyCloud.

- `cloud:project:create`, `project:create`
  Tworzy nowy projekt SymfonyCloud.

- `cloud:project:delete`, `project:delete`
  Usuwa projekt SymfonyCloud.

- `cloud:project:info`, `project:info`
  Wyświetla szczegóły projektu SymfonyCloud.

- `cloud:project:set-remote`, `project:set-remote`
  Ustawia zdalne repozytorium Git na SymfonyCloud.

- `cloud:environment:list`, `environment:list`
  Wyświetla listę środowisk SymfonyCloud.

- `cloud:environment:branch`, `environment:branch`
  Tworzy nowe środowisko z istniejącego.

- `cloud:environment:activate`, `environment:activate`
  Ustawia aktywne środowisko.

- `cloud:environment:delete`, `environment:delete`
  Usuwa środowisko.

- `cloud:environment:merge`, `environment:merge`
  Scala środowiska.

- `cloud:environment:sync`, `environment:sync`
  Synchronizuje dane (np. bazy danych) między środowiskami.

### 📦 Zarządzanie usługami (bazy danych, cache)

- `cloud:service:list`, `service:list`
  Lista usług SymfonyCloud w danym projekcie.

- `cloud:service:info`, `service:info`
  Szczegóły konkretnej usługi (np. PostgreSQL, Redis).

- `cloud:service:add`, `service:add`
  Dodaje nową usługę do projektu (np. bazę danych).

- `cloud:service:delete`, `service:delete`
  Usuwa usługę z projektu.

- `cloud:service:database:dump`, `db:dump`
  Tworzy zrzut bazy danych ze środowiska SymfonyCloud.

- `cloud:service:database:import`, `db:import`
  Importuje lokalną bazę danych do chmury.

### 📄 Logi i monitorowanie

- `cloud:log`, `log`
  Wyświetla logi środowiska.

- `cloud:activity:list`, `activity:list`
  Lista aktywności (np. wdrożeń, restartów, zmian).

- `cloud:activity:log`, `activity:log`
  Szczegółowe logi danej aktywności.

- `cloud:activity:wait`, `activity:wait`
  Oczekuje na zakończenie aktywności (np. deploymentu).

### 📡 Tunele i zdalny dostęp

- `cloud:tunnel:open`, `tunnel:open`
  Tworzy tunel do zdalnej usługi (np. zdalna baza danych).

- `cloud:tunnel:close`, `tunnel:close`
  Zamyka tunel do zdalnej usługi.

- `cloud:tunnel:list`, `tunnel:list`
  Lista aktywnych tuneli.

- `cloud:tunnel:single`, `tunnel:single`
  Tworzy tunel jednorazowy.


## 🚀 Build, deploy, release

### 🔨 Budowanie i wdrażanie

- `cloud:build`, `build`
  Uruchamia lokalny proces budowania aplikacji tak, jak robi to SymfonyCloud podczas deploymentu.

- `cloud:deploy`, `deploy`
  Wdraża zmiany na SymfonyCloud (push + deploy).

- `cloud:redeploy`, `redeploy`
  Powtarza ostatni deployment bez zmian w kodzie.

- `cloud:release`, `release`
  Tworzy nową wersję środowiska (release) — można przypisać do produkcji.

## 🌐 Zarządzanie domenami i SSL

- `cloud:domain:add`, `domain:add`
  Dodaje domenę do projektu.

- `cloud:domain:delete`, `domain:delete`
  Usuwa domenę z projektu.

- `cloud:domain:list`, `domain:list`
  Wyświetla listę przypisanych domen.

- `cloud:domain:update`, `domain:update`
  Zmienia ustawienia domeny (np. certyfikaty).

- `cloud:certificate:request`, `certificate:request`
  Żąda nowego certyfikatu SSL dla domeny.

- `cloud:certificate:info`, `certificate:info`
  Wyświetla szczegóły certyfikatu SSL.

## ⏰ Crontaby (zadania cykliczne)

- `cloud:cron:add`, `cron:add`
  Dodaje nowe zadanie cron do projektu.

- `cloud:cron:list`, `cron:list`
  Lista wszystkich zadań cron.

- `cloud:cron:delete`, `cron:delete`
  Usuwa zadanie cron.

## 💾 Backup i przywracanie danych

- `cloud:backup:list`, `backup:list`
  Lista dostępnych backupów dla usług (np. baz danych).

- `cloud:backup:restore`, `backup:restore`
  Przywraca backup do usługi.

## 🪄 Aliasowanie i wygodne skróty

- `cloud:alias`, `alias`
  Tworzy aliasy bash/zsh do projektów i środowisk (np. `@prod`, `@myproject.dev`).

## 🧰 Narzędzia developerskie

- `cloud:drush`
  Uruchamia komendy Drush (jeśli projekt oparty na Drupalu).

- `cloud:console`
  Uruchamia interaktywną powłokę do zarządzania aplikacją w środowisku.

- `cloud:url`
  Wyświetla adres URL środowiska.

- `cloud:ssh-key:add`, `ssh-key:add`
  Dodaje klucz SSH do konta SymfonyCloud.

- `cloud:ssh-key:list`, `ssh-key:list`
  Lista kluczy SSH.

- `cloud:ssh-key:delete`, `ssh-key:delete`
  Usuwa klucz SSH z konta.

- `cloud:self-update`
  Aktualizuje Symfony CLI do najnowszej wersji.

## 🧩 Inne

- `cloud:config:validate`, `config:validate`
  Waliduje pliki `.symfony.cloud.yaml`, `.services.yaml` itd.

- `cloud:checkout`
  Automatycznie tworzy lokalny branch zdalnego środowiska.
