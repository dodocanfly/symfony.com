## Konfigurowanie bazy danych

Strona internetowa *Conference Guestbook* służy do zbierania opinii podczas konferencji. Musimy przechowywać komentarze uczestników konferencji w trwałej pamięci.

Komentarz najlepiej opisać za pomocą ustalonej struktury danych: autor, jego adres e-mail, treść opinii oraz opcjonalne zdjęcie. Tego rodzaju dane najlepiej przechowywać w tradycyjnej relacyjnej bazie danych.

Jako silnika bazy danych użyjemy **PostgreSQL** .

### Dodawanie PostgreSQL do Docker Compose

Na naszej lokalnej maszynie zdecydowaliśmy się używać Dockera do zarządzania usługami. Wygenerowany plik `compose.yaml` zawiera już PostgreSQL jako usługę:

**compose.yaml**
```yaml
###> doctrine/doctrine-bundle ###
database:
    image: postgres:${POSTGRES_VERSION:-16}-alpine
    environment:
        POSTGRES_DB: ${POSTGRES_DB:-app}
        # Koniecznie zmień hasło w środowisku produkcyjnym
        POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-ChangeMe}
        POSTGRES_USER: ${POSTGRES_USER:-app}
volumes:
    - db-data:/var/lib/postgresql/data:rw
    # Możesz zamiast tego użyć katalogu powiązanego z systemem plików hosta, co utrudni przypadkowe usunięcie wolumenu i utratę danych!
    # - ./docker/db/data:/var/lib/postgresql/data:rw
###< doctrine/doctrine-bundle ###
```

To spowoduje zainstalowanie serwera PostgreSQL i skonfiguruje kilka zmiennych środowiskowych, które kontrolują nazwę bazy danych oraz dane uwierzytelniające. Ich konkretne wartości nie mają większego znaczenia na tym etapie.

Eksponujemy również port PostgreSQL (5432) z kontenera na hosta lokalnego. Dzięki temu możemy uzyskać dostęp do bazy danych z naszej maszyny:

**compose.override.yaml** 
```yaml
###> doctrine/doctrine-bundle ###
database:
    ports:
    - "5432"
###< doctrine/doctrine-bundle ###
```

> [!NOTE]
> Rozszerzenie `pdo_pgsql` powinno zostać zainstalowane podczas wcześniejszego etapu konfiguracji PHP.

### Uruchamianie Docker Compose

Uruchom Docker Compose w tle (z flagą `-d`):

```bash
docker compose up -d
```

Poczekaj chwilę, aby baza danych mogła się uruchomić, a następnie sprawdź, czy wszystko działa poprawnie:

```bash
docker compose ps
```

Przykładowy wynik:

```
Name                      Command              State            Ports
---------------------------------------------------------------------------------------
guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp
```

Jeśli nie ma uruchomionych kontenerów lub w kolumnie **State**  nie widnieje **Up** , sprawdź logi Docker Compose:

```bash
docker compose logs
```

### Dostęp do lokalnej bazy danych

Używanie narzędzia wiersza poleceń `psql` może się czasem okazać przydatne. Trzeba jednak pamiętać o danych uwierzytelniających i nazwie bazy danych. Mniej oczywiste jest to, że musisz też znać lokalny port, na którym baza działa na hoście. Docker wybiera losowy port, aby umożliwić pracę nad kilkoma projektami korzystającymi z PostgreSQL jednocześnie (lokalny port znajduje się w wyniku polecenia `docker compose ps`).

Jeśli uruchomisz `psql` za pomocą Symfony CLI, nie musisz niczego pamiętać.
Symfony CLI automatycznie wykrywa usługi Dockera uruchomione dla projektu i udostępnia zmienne środowiskowe, których `psql` potrzebuje do połączenia się z bazą danych.
Dzięki tym konwencjom dostęp do bazy danych przez `symfony run` jest znacznie łatwiejszy:

```bash
symfony run psql
```

> [!NOTE]
> Jeśli nie masz zainstalowanego polecenia `psql` na swoim hoście lokalnym, możesz je również uruchomić przez Docker Compose:
> ```bash
> docker compose exec database psql app app
> ```

### Zrzut i przywracanie danych bazy danych

Użyj polecenia `pg_dump`, aby wykonać zrzut danych z bazy:

```bash
symfony run pg_dump --data-only > dump.sql
```

Aby przywrócić dane:

```bash
symfony run psql < dump.sql
```

### Dodawanie PostgreSQL do Platform.sh

W infrastrukturze produkcyjnej na Platform.sh dodanie usługi takiej jak PostgreSQL powinno zostać wykonane w pliku `.platform/services.yaml`, co zostało już zrobione za pomocą przepisu pakietu webapp:

**.platform/services.yaml** 

```yaml
database:
    type: postgresql:16
    disk: 1024
```

Usługa `database` to baza danych PostgreSQL (ta sama wersja co dla Dockera), którą chcemy udostępnić z dyskiem o pojemności 1 GB.
Musimy również „połączyć” bazę danych z kontenerem aplikacji, co opisane jest w pliku `.platform.app.yaml`:
**.platform.app.yaml** 

```yaml
relationships:
    database: "database:postgresql"
```

Usługa bazy danych typu `postgresql` jest odwoływana jako `database` w kontenerze aplikacji.
Sprawdź, czy rozszerzenie `pdo_pgsql` jest już zainstalowane dla środowiska wykonawczego PHP:
**.platform.app.yaml** 

```yaml
runtime:
    extensions:
        # inne rozszerzenia
        - pdo_pgsql
        # inne rozszerzenia
```

### Dostęp do bazy danych Platform.sh

PostgreSQL działa teraz zarówno lokalnie przez Dockera, jak i w środowisku produkcyjnym na Platform.sh.

Jak już widzieliśmy, uruchomienie polecenia `symfony run psql` automatycznie łączy się z bazą danych hostowaną przez Dockera dzięki zmiennym środowiskowym udostępnianym przez `symfony run`.

Jeśli chcesz połączyć się z PostgreSQL działającym na kontenerach produkcyjnych, możesz otworzyć tunel SSH między lokalną maszyną a infrastrukturą Platform.sh:

```bash
symfony cloud:tunnel:open
symfony var:expose-from-tunnel
```

Domyślnie usługi Platform.sh nie są udostępniane jako zmienne środowiskowe na lokalnej maszynie. Musisz to zrobić jawnie, uruchamiając polecenie `var:expose-from-tunnel`. Dlaczego? Połączenie z produkcyjną bazą danych to operacja obarczona ryzykiem — możesz przypadkowo wpłynąć na prawdziwe dane.
Teraz możesz połączyć się z zdalną bazą PostgreSQL za pomocą `symfony run psql`, tak jak wcześniej:

```bash
symfony run psql
```

Po zakończeniu nie zapomnij zamknąć tunelu:

```bash
symfony cloud:tunnel:close
```

> [!TIP]
> Aby uruchomić zapytania SQL na produkcyjnej bazie danych bez wchodzenia do powłoki, możesz też użyć polecenia ``symfony sql``

### Udostępnianie zmiennych środowiskowych

Docker Compose i Platform.sh współpracują z Symfony bezproblemowo dzięki zmiennym środowiskowym.

Aby sprawdzić wszystkie zmienne środowiskowe udostępniane przez Symfony, wykonaj polecenie:

```bash
symfony var:export
```

Przykładowy wynik:

```ini
PGHOST=127.0.0.1  
PGPORT=32781  
PGDATABASE=app  
PGUSER=app  
PGPASSWORD=!ChangeMe!
```

Zmienne środowiskowe zaczynające się od `PG*` są odczytywane przez narzędzie `psql`. A co z pozostałymi?
Gdy tunel do Platform.sh jest otwarty i użyto `var:expose-from-tunnel`, polecenie `var:export` zwraca zdalne zmienne środowiskowe:

```bash
symfony cloud:tunnel:open
symfony var:expose-from-tunnel
symfony var:export
symfony cloud:tunnel:close
```

### Opisywanie infrastruktury

Możesz jeszcze tego nie zauważyłeś, ale przechowywanie konfiguracji infrastruktury w plikach razem z kodem jest bardzo pomocne. Docker i Platform.sh używają plików konfiguracyjnych do opisywania infrastruktury projektu. Gdy nowa funkcja wymaga dodatkowej usługi, zmiany w kodzie i infrastrukturze są częścią tego samego zestawu zmian (patcha).

### Sprawdź również:
- [Usługi Platform.sh](https://symfony.com/doc/current/cloud/services/intro.html#available-services)
- [Tunel Platform.sh](https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service)
- [Dokumentację PostgreSQL](https://www.postgresql.org/docs)
- [Polecenia `docker compose`](https://docs.docker.com/compose/reference)

---

- **Poprzednia strona:** [Poprzednia strona...](6-6th.md)
- **Następna strona:** [Opis struktury danych](8-doctrine.md)
