## Symfony Console - Tłumaczenie komend CLI

### 🛠️ Ogólne
- `about` – Wyświetla informacje o bieżącym projekcie
- `completion` – Generuje skrypt do automatycznego uzupełniania poleceń w terminalu
- `help` – Wyświetla pomoc dla danego polecenia
- `list` – Lista wszystkich dostępnych komend

### 📦 Assety
- `asset-map:compile` – Kompiluje wszystkie mapowane assety do folderu publicznego
- `assets:install` – Instaluje assety bundli do katalogu publicznego

### 🔧 Cache
- `cache:clear` – Czyści pamięć podręczną
- `cache:pool:clear` – Czyści wybrane pule cache
- `cache:pool:delete` – Usuwa element z puli cache
- `cache:pool:invalidate-tags` – Inwaliduje tagi cache
- `cache:pool:list` – Lista dostępnych pul cache
- `cache:pool:prune` – Czyści przestarzałe dane z puli cache
- `cache:warmup` – Wstępne ładowanie cache

### ⚙️ Konfiguracja i kontenery
- `config:dump-reference` – Pokazuje domyślną konfigurację danego rozszerzenia
- `debug:autowiring` – Lista klas/interfejsów możliwych do autowire'owania
- `debug:config` – Wyświetla aktualną konfigurację danego rozszerzenia
- `debug:container` – Lista aktualnie dostępnych serwisów
- `lint:container` – Sprawdza poprawność konfiguracji usług

### 🧪 Debugowanie i analiza
- `debug:asset-map` – Wyświetla wszystkie zmapowane assety
- `debug:dotenv` – Lista plików .env z wartościami
- `debug:event-dispatcher` – Lista skonfigurowanych listenerów
- `debug:firewall` – Informacje o firewallach zabezpieczeń
- `debug:form` – Informacje o typach formularzy
- `debug:messenger` – Lista wiadomości dostępnych dla message busa
- `debug:router` – Lista dostępnych tras
- `debug:serializer` – Informacje o serializacji klas
- `debug:translation` – Informacje o wiadomościach tłumaczeń
- `debug:twig` – Lista funkcji, filtrów, globali i testów Twig
- `debug:twig-component` – Lista komponentów Twig i ich użycia
- `debug:validator` – Informacje o regułach walidacyjnych

### 🧰 Walidacja i lintowanie
- `lint:twig` – Sprawdza poprawność pliku Twig
- `lint:xliff` – Sprawdza poprawność pliku XLIFF
- `lint:yaml` – Sprawdza poprawność pliku YAML

### 🛢️ Doctrine
#### Cache:
- `doctrine:cache:clear-collection-region` – Czyści cache kolekcji drugiego poziomu
- `doctrine:cache:clear-entity-region` – Czyści cache encji drugiego poziomu
- `doctrine:cache:clear-metadata` – Czyści metadata cache
- `doctrine:cache:clear-query` – Czyści query cache
- `doctrine:cache:clear-query-region` – Czyści query cache drugiego poziomu
- `doctrine:cache:clear-result` – Czyści result cache

#### Baza danych:
- `doctrine:database:create` – Tworzy bazę danych
- `doctrine:database:drop` – Usuwa bazę danych

#### Mapowanie:
- `doctrine:mapping:info` – Informacje o mapowanych encjach

#### Zapytania:
- `doctrine:query:dql` – Wykonuje DQL z poziomu CLI
- `doctrine:query:sql` – Wykonuje SQL z poziomu CLI

#### Schemat:
- `doctrine:schema:create` – Tworzy schemat bazy danych
- `doctrine:schema:drop` – Usuwa schemat bazy danych
- `doctrine:schema:update` – Aktualizuje schemat do aktualnego mapowania
- `doctrine:schema:validate` – Waliduje pliki mapowania

#### Migracje:
- `doctrine:migrations:current` – Pokazuje aktualną wersję migracji
- `doctrine:migrations:diff` – Tworzy migrację na podstawie zmian
- `doctrine:migrations:dump-schema` – Eksportuje schemat jako migrację
- `doctrine:migrations:execute` – Ręczne wykonanie konkretnej wersji migracji
- `doctrine:migrations:generate` – Tworzy pustą klasę migracyjną
- `doctrine:migrations:latest` – Pokazuje najnowszą wersję
- `doctrine:migrations:list` – Lista dostępnych migracji i ich status
- `doctrine:migrations:migrate` – Wykonuje migrację do danej lub najnowszej wersji
- `doctrine:migrations:rollup` – Scalanie migracji w jedną wersję
- `doctrine:migrations:status` – Status migracji
- `doctrine:migrations:sync-metadata-storage` – Aktualizuje metadane migracji
- `doctrine:migrations:up-to-date` – Czy baza jest aktualna?
- `doctrine:migrations:version` – Dodaje/usuwa wersje migracji ręcznie

### 🧩 Importmap
- `importmap:audit` – Sprawdza podatności zależności
- `importmap:install` – Pobiera wymagane assety
- `importmap:outdated` – Lista nieaktualnych paczek JS
- `importmap:remove` – Usuwa paczki JS
- `importmap:require` – Wymaga paczek JS
- `importmap:update` – Aktualizuje paczki JS

### 📬 Mailer
- `mailer:test` – Testuje transporty mailowe przez wysłanie wiadomości

### 🔨 MakerBundle
_(Tworzenie klas i struktur)_

- `make:admin:crud` – Tworzy klasę CRUD EasyAdmin
- `make:admin:dashboard` – Tworzy dashboard EasyAdmin
- `make:auth` – Tworzy Guard Authenticator
- `make:command` – Tworzy nową komendę CLI
- `make:controller` – Tworzy kontroler
- `make:crud` – Tworzy CRUD dla encji Doctrine
- `make:docker:database` – Dodaje bazę do pliku docker-compose
- `make:entity` – Tworzy lub aktualizuje encję Doctrine
- `make:fixtures` – Tworzy klasę z danymi testowymi (fixtures)
- `make:form` – Tworzy klasę formularza
- `make:listener` – Tworzy listener/subscriber
- `make:message` – Tworzy wiadomość + handler
- `make:messenger-middleware` – Tworzy middleware Messengera
- `make:migration` – Tworzy nową migrację
- `make:registration-form` – Tworzy system rejestracji użytkownika
- `make:reset-password` – Generuje reset hasła (z kontrolerem, encją itd.)
- `make:schedule` – Tworzy komponent scheduler
- `make:security:custom` – Tworzy customowy authenticator
- `make:security:form-login` – Generuje form_login
- `make:serializer:encoder` – Tworzy nowy encoder serializer
- `make:serializer:normalizer` – Tworzy nowy normalizer serializer
- `make:stimulus-controller` – Tworzy kontroler Stimulus
- `make:test` – Tworzy klasę testową (unit/functional)
- `make:twig-component` – Tworzy komponent Twig
- `make:twig-extension` – Tworzy rozszerzenie Twig
- `make:user` – Tworzy klasę użytkownika (security)
- `make:validator` – Tworzy walidator i constraint
- `make:voter` – Tworzy klasę votera
- `make:webhook` – Tworzy webhooka

### 📬 Messenger
- `messenger:consume` – Uruchamia odbiorcę wiadomości
- `messenger:failed:remove` – Usuwa wiadomości z kolejki błędów
- `messenger:failed:retry` – Ponawia wiadomości z kolejki błędów
- `messenger:failed:show` – Pokazuje wiadomości z kolejki błędów
- `messenger:setup-transports` – Tworzy infrastrukturę dla transportu
- `messenger:stats` – Statystyki wiadomości w transportach
- `messenger:stop-workers` – Zatrzymuje workerów po obecnym zadaniu

### 🔒 Security i hasła
- `security:hash-password` – Haszuje hasło użytkownika

### 📡 Router
- `router:match` – Debugowanie trasy na podstawie ścieżki

### 🔐 Sekrety
- `secrets:decrypt-to-local` – Deszyfruje wszystkie sekrety do lokalnej skrytki
- `secrets:encrypt-from-local` – Szyfruje lokalne sekrety do vaulta
- `secrets:generate-keys` – Tworzy nowe klucze szyfrujące
- `secrets:list` – Lista sekretów
- `secrets:remove` – Usuwa sekret
- `secrets:set` – Ustawia sekret

### 📡 Serwer i debug
- `server:dump` – Serwer do odbioru `dump()` w jednym miejscu
- `server:log` – Serwer logów w czasie rzeczywistym

### 🌍 Tłumaczenia
- `translation:extract` – Ekstrahuje brakujące klucze tłumaczeń
- `translation:pull` – Pobiera tłumaczenia z dostawcy
- `translation:push` – Wysyła tłumaczenia do dostawcy
