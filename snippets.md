### Symfony

#### symfony new

##### Klonowanie repozytorium [https://github.com/the-fast-track/book-6.4-1](https://github.com/the-fast-track/book-6.4-1)
```bash
symfony new --version=6.4-1 --book guestbook
```

##### Tworzenie nowej aplikacji Symfony
```bash
symfony new guestbook --version=6.4 --php=8.3 --webapp --docker --cloud
```
- `--version=6.4` - wersja Symfony
- `--php=8.3` - wersja PHP
- `--webapp` - domyślnie tworzona jest aplikacja z minimalną ilością zależności - dla większości projektów webowych zaleca się dodanie pakietu webapp, który zawiera potrzebne pakiety, m.in. Symfony Messenger i PostgreSQL przez Doctrine
- `--docker` - na lokalnym komputerze używamy Dockera do zarządzania usługami, takimi jak PostgreSQL - ta opcja automatycznie dodaje odpowiednie konfiguracje dla Dockera
- `--cloud` - jeśli chcesz wdrożyć projekt na Platform.sh, ta opcja generuje pliki konfiguracyjne niezbędne do tego środowiska

#### symfony book

##### Przejście do kodu z końca kroku 10 / podkroku 10.2
```bash
symfony book:checkout 10
symfony book:checkout 10.2
```

#### symfony server / open

##### Uruchomienie lokalnego serwera w tle
```bash
symfony server:start -d
```

##### Uruchonienie logów lokalnego serwera w konsoli
```bash
symfony server:log
```

##### Otwarcie strony aplikacji w przeglądarce (lokalnie)
```bash
symfony open:local
```

#### symfony cloud

##### Tworzenie nowego zdalnego projektu na Platform.sh
```bash
symfony cloud:project:create --title="Guestbook" --plan=development
```

##### Deployowanie projektu
```bash
symfony cloud:deploy
```

##### Otwarcie strony aplikacji w przeglądarce (zdalnej na Platform.sh)
```bash
symfony cloud:url -1
```

##### Usunięcie projektu z Platform.sh
```bash
symfony cloud:project:delete
```

##### Wyświetlenie w konsoli logów z serwera produkcyjnego Platform.sh
```bash
symfony cloud:log --tail
```

##### Nawiązanie połączenia SSH z kontenerem na Platform.sh
```bash
symfony cloud:ssh
```

##### Otwarcie tunelu SSH między lokalną maszyną a infrastrukturą Platform.sh
```bash
symfony cloud:tunnel:open
symfony tunnel:open
```

##### Wyświetlenie listy tuneli
```bash
symfony cloud:tunnels
symfony tunnels
```

##### Wyświetlenie informacji o tunelu
```bash
symfony cloud:tunnel:info
symfony tunnel:info
```

##### Zamknięcie otwartego tunelu
```bash
symfony cloud:tunnel:close
symfony tunnel:close
```

##### Synchronizacja zmiennych środowiskowych i danych z serwera produkcyjnego Platform.sh do lokalnego ([więcej](symfony-cloud-env-sync.md))
```bash
symfony cloud:env:sync
```

#### symfony var

##### Lista wszystkich zmiennych środowiskowych udostępnianych przez Symfony
```bash
symfony var:export
symfony var:export | sed 's/ /\n/g' | sort
```

##### Eksport zmiennych środowiskowych z serwera Platform.sh połączonego tunelem
```bash
symfony var:expose-from-tunnel
```

#### symfony console

##### Wyświetlenie listy wszystkich generatowów kodu Symfony
```bash
symfony console list make
```

##### Tworzenie encji i repozytorium
```bash
symfony console make:entity Comment
```

##### Tworzenie migracji
```bash
symfony console make:migration
```

##### Uruchamia migracje
```bash
symfony console doctrine:migrations:migrate
```

##### Generowanie panelu EasyAdmin (wymaga instalacji "admin:^4")
```bash
symfony console make:admin:dashboard
```

##### Generowanie CRUD dla encji w EasyAdmin
```bash
symfony console make:admin:crud
```

#### symfony run

##### Zrzut danych z bazy PostgreSQL
```bash
symfony run pg_dump --data-only > dump.sql
```

##### Przywrócenie danych z pliku
```bash
symfony run psql < dump.sql
```

### Composer

##### Instalowanie EasyAdmina
```bash
symfony composer req "admin:^4"
# lub
composer req "easycorp/easyadmin-bundle:4.x-dev"
```

### Git

##### Wyświetlenie różnic w kodzie pomiędzy podanymi krokami (branchami)
```bash
git diff step-10-1...step-10-2
git diff step-9...step-10-1
```

##### Sprawdzanie kiedy dany plik został utworzony lub zmodyfikowany
```bash
git log -- src/Controller/ConferenceController.php
```

### Docker

##### Wyświetlenie logów Docker Compose
```bash
docker compose logs
```
