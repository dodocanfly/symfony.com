**Przechodzenie na tryb asynchroniczny**

Sprawdzanie spamu podczas obsługi przesyłania formularza może prowadzić do pewnych problemów. Jeśli API Akismet zacznie działać wolno, nasza strona internetowa również będzie działać wolno dla użytkowników. Co gorsza, jeśli dojdzie do przekroczenia limitu czasu lub API Akismet będzie niedostępne, możemy utracić komentarze.

W idealnym przypadku powinniśmy przechowywać przesłane dane bez ich publikowania i natychmiast zwracać odpowiedź. Sprawdzenie pod kątem spamu może wtedy odbywać się poza głównym tokiem działania.

---

**Oznaczanie komentarzy**

Musimy wprowadzić stan dla komentarzy: *przesłany (submitted)*, *spam*, oraz *opublikowany (published)*.

Dodaj właściwość `state` do klasy `Comment`.

Powinniśmy również upewnić się, że domyślnie stan komentarza ustawiany jest na *submitted*.

Utwórz migrację bazy danych.

Zmodyfikuj migrację, aby zaktualizować wszystkie istniejące komentarze i domyślnie ustawić ich stan na *published*.

Wykonaj migrację bazy danych.

Zaktualizuj logikę wyświetlania, aby unikać wyświetlania nieopublikowanych komentarzy na froncie.

Zaktualizuj konfigurację EasyAdmin, aby można było zobaczyć stan komentarza.

Nie zapomnij także zaktualizować testów, ustawiając stan danych testowych (fixtures).

Dla testów kontrolera zasymuluj walidację.

W testach PHPUnit możesz uzyskać dostęp do dowolnego serwisu z kontenera poprzez `self::getContainer()->get()`. To umożliwia także dostęp do serwisów niepublicznych.

---

**Zrozumienie komponentu Messenger**

Zarządzanie kodem asynchronicznym w Symfony to zadanie dla komponentu *Messenger*.

Gdy jakaś logika powinna zostać wykonana asynchronicznie, należy wysłać wiadomość (*message*) do szyny wiadomości (*messenger bus*). Szyna zapisuje wiadomość w kolejce i natychmiast zwraca kontrolę, aby przepływ operacji mógł być jak najszybciej kontynuowany.

Proces *consumer* działa w tle w sposób ciągły, odczytuje nowe wiadomości z kolejki i wykonuje powiązaną logikę. Może on działać na tym samym serwerze co aplikacja webowa albo na osobnym.

Działa to bardzo podobnie do obsługi żądań HTTP, z tą różnicą, że tutaj nie mamy odpowiedzi.

---

**Tworzenie handlera wiadomości**

Wiadomość to klasa danych, która nie powinna zawierać żadnej logiki. Będzie ona serializowana i przechowywana w kolejce, dlatego należy przechowywać tylko „proste”, możliwe do serializacji dane.

Utwórz klasę `CommentMessage`.

W świecie Messenger nie mamy kontrolerów, lecz *message handlers* (obsługiwacze wiadomości).

Utwórz klasę `CommentMessageHandler` w nowej przestrzeni nazw `App\MessageHandler`, która będzie wiedziała jak obsługiwać wiadomości typu `CommentMessage`.

Atrybut `#[AsMessageHandler]` pomaga Symfony automatycznie zarejestrować i skonfigurować klasę jako handler w Messengerze. Zgodnie z konwencją, logika handlera znajduje się w metodzie o nazwie `__invoke()`. Podpowiedź typu `CommentMessage` w argumencie tej metody informuje Messengera, którą klasę obsługiwać.

Zaktualizuj kontroler, aby korzystał z nowego systemu.

Zamiast polegać na sprawdzaczu spamu, teraz wysyłamy wiadomość na szynę wiadomości. Handler decyduje, co z nią zrobić.

Osiągnęliśmy coś nieoczekiwanego — oddzieliliśmy kontroler od logiki sprawdzania spamu i przenieśliśmy ją do nowej klasy, handlera. To idealny przypadek użycia szyny wiadomości. Przetestuj kod — działa. Wszystko nadal wykonywane jest synchronicznie, ale kod jest prawdopodobnie już „lepszy”.

---

**Naprawdę przechodzimy na asynchroniczność**

Domyślnie handlery są wywoływane synchronicznie. Aby przejść na tryb asynchroniczny, musisz jawnie skonfigurować, której kolejki używać dla każdego handlera w pliku konfiguracyjnym `config/packages/messenger.yaml`.

Konfiguracja informuje szynę wiadomości, aby instancje `App\Message\CommentMessage` były wysyłane do kolejki *async*, która jest zdefiniowana przez DSN `MESSENGER_TRANSPORT_DSN`, wskazujący na Doctrine zgodnie z ustawieniem w pliku `.env`. Mówiąc prosto: używamy PostgreSQL jako kolejki dla naszych wiadomości.

Za kulisami Symfony wykorzystuje wbudowany, wydajny, skalowalny i transakcyjny system pub/sub PostgreSQL (LISTEN/NOTIFY). Możesz też przeczytać rozdział o RabbitMQ, jeśli chcesz używać go zamiast PostgreSQL jako brokera wiadomości.

---

**Konsumowanie wiadomości**

Jeśli spróbujesz przesłać nowy komentarz, sprawdzacz spamu nie zostanie już wywołany. Dodaj wywołanie `error_log()` w metodzie `getSpamScore()`, aby to potwierdzić. Zamiast tego, wiadomość czeka w kolejce, gotowa do przetworzenia przez jakiś proces.

Jak możesz się domyślić, Symfony posiada komendę *consumer*. Uruchom ją teraz.

Powinna natychmiast przetworzyć wiadomość wysłaną dla przesłanego komentarza.

Aktywność konsumenta wiadomości jest logowana, ale możesz uzyskać natychmiastową informację zwrotną w konsoli, przekazując flagę `-vv`. Powinieneś nawet być w stanie zauważyć wywołanie API Akismet.

Aby zatrzymać konsumenta, naciśnij Ctrl+C.

---

**Uruchamianie workerów w tle**

Zamiast uruchamiać konsumenta za każdym razem, gdy publikujemy komentarz, i zatrzymywać go zaraz potem, chcemy uruchomić go na stałe bez potrzeby otwierania wielu okien lub zakładek terminala.

Symfony CLI może zarządzać takimi poleceniami działającymi w tle, używając flagi `-d` przy poleceniu `run`.

Uruchom ponownie konsumenta wiadomości, ale tym razem w tle.

Opcja `--watch` informuje Symfony, że polecenie powinno zostać ponownie uruchomione za każdym razem, gdy nastąpi zmiana w systemie plików w katalogach `config/`, `src/`, `templates/` lub `vendor/`.

Nie używaj `-vv`, ponieważ otrzymasz zduplikowane wiadomości w `server:log` (zalogowane i wyświetlone w konsoli).

Jeśli konsument przestanie działać z jakiegoś powodu (limit pamięci, błąd itp.), zostanie automatycznie uruchomiony ponownie. A jeśli zbyt szybko przestanie działać wielokrotnie, Symfony CLI się podda.

Logi są strumieniowane przez `symfony server:log` razem z innymi logami pochodzącymi z PHP, serwera webowego i aplikacji.

Użyj polecenia `server:status`, aby wyświetlić listę wszystkich workerów działających w tle dla bieżącego projektu.

Aby zatrzymać workera, zatrzymaj serwer webowy lub zabij PID podany przez `server:status`.

---

**Ponowne próby nieudanych wiadomości**

A co jeśli Akismet nie działa w momencie przetwarzania wiadomości? Nie ma to wpływu na osoby przesyłające komentarze, ale wiadomość jest utracona i spam nie zostaje sprawdzony.

Messenger posiada mechanizm ponawiania w przypadku wystąpienia wyjątku podczas obsługi wiadomości.

Jeśli podczas obsługi wiadomości wystąpi problem, konsument spróbuje ponowić próbę 3 razy, zanim się podda. Ale zamiast całkowicie porzucić wiadomość, zapisze ją trwale w kolejce *failed*, która korzysta z innej tabeli bazy danych.

Sprawdź nieudane wiadomości i ponów ich przetwarzanie za pomocą następujących poleceń:

[...]

---

**Uruchamianie workerów na Platform.sh**

Aby konsumować wiadomości z PostgreSQL, musimy nieprzerwanie uruchamiać komendę `messenger:consume`. Na Platform.sh jest to rola workera.

Podobnie jak w przypadku Symfony CLI, Platform.sh zarządza restartami i logami.

Aby uzyskać logi z workera, użyj:

[...]

---

**Dalsze kroki:**

- Kurs SymfonyCasts dotyczący komponentu Messenger;  
- Architektura Enterprise Service Bus i wzorzec CQRS;  
- Dokumentacja Symfony Messenger.
