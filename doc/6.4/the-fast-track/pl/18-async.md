## Przechodzenie na tryb asynchroniczny

Sprawdzanie spamu podczas obsługi przesyłania formularza może prowadzić do pewnych problemów. Jeśli API Akismet zacznie działać wolno, nasza strona internetowa również będzie działać wolno dla użytkowników. Co gorsza, jeśli dojdzie do przekroczenia limitu czasu lub API Akismet będzie niedostępne, możemy utracić komentarze.

W idealnym przypadku powinniśmy przechowywać przesłane dane bez ich publikowania i natychmiast zwracać odpowiedź. Sprawdzenie pod kątem spamu może wtedy odbywać się poza głównym tokiem działania.

### Oznaczanie komentarzy

Musimy wprowadzić `stan` dla komentarzy:  `submitted` (przesłany), `spam` oraz `published` (opublikowany).

Dodaj właściwość `state` do klasy `Comment`:

```bash
symfony console make:entity Comment
```

Powinniśmy również upewnić się, że domyślnie stan komentarza (`state`) ustawiany jest na `submitted`:

```diff
--- a/src/Entity/Comment.php
+++ b/src/Entity/Comment.php
@@ -39,8 +39,8 @@ class Comment
     #[ORM\Column(length: 255, nullable: true)]
     private ?string $photoFilename = null;

-    #[ORM\Column(length: 255)]
-    private ?string $state = null;
+    #[ORM\Column(length: 255, options: ['default' => 'submitted'])]
+    private ?string $state = 'submitted';

     public function getId(): ?int
     {
```

Utwórz migrację bazy danych:

```bash
symfony console make:migration
```

Zmodyfikuj migrację, aby zaktualizować wszystkie istniejące komentarze i domyślnie ustawić ich stan na `published`:

```diff
--- a/migrations/Version00000000000000.php
+++ b/migrations/Version00000000000000.php
@@ -21,6 +21,7 @@ final class Version00000000000000 extends AbstractMigration
     {
         // this up() migration is auto-generated, please modify it to your needs
         $this->addSql('ALTER TABLE comment ADD state VARCHAR(255) DEFAULT \'submitted\' NOT NULL');
+        $this->addSql("UPDATE comment SET state='published'");
     }

     public function down(Schema $schema): void
```

Wykonaj migrację bazy danych:

```bash
symfony console doctrine:migrations:migrate
```

Zaktualizuj logikę wyświetlania, aby unikać wyświetlania nieopublikowanych komentarzy na froncie:

```diff
--- a/src/Repository/CommentRepository.php
+++ b/src/Repository/CommentRepository.php
@@ -29,7 +29,9 @@ class CommentRepository extends ServiceEntityRepository
     {
         $query = $this->createQueryBuilder('c')
             ->andWhere('c.conference = :conference')
+            ->andWhere('c.state = :state')
             ->setParameter('conference', $conference)
+            ->setParameter('state', 'published')
             ->orderBy('c.createdAt', 'DESC')
             ->setMaxResults(self::COMMENTS_PER_PAGE)
             ->setFirstResult($offset)
```

Zaktualizuj konfigurację EasyAdmin, aby można było zobaczyć stan komentarza:

```diff
--- a/src/Controller/Admin/CommentCrudController.php
+++ b/src/Controller/Admin/CommentCrudController.php
@@ -53,6 +53,7 @@ class CommentCrudController extends AbstractCrudController
             ->setLabel('Photo')
             ->onlyOnIndex()
         ;
+        yield TextField::new('state');

         $createdAt = DateTimeField::new('createdAt')->setFormTypeOptions([
             'years' => range(date('Y'), date('Y') + 5),
```

Nie zapomnij także zaktualizować testów, ustawiając stan (`state`) danych testowych (fixtures):

```diff
--- a/src/DataFixtures/AppFixtures.php
+++ b/src/DataFixtures/AppFixtures.php
@@ -35,8 +35,16 @@ class AppFixtures extends Fixture
         $comment1->setAuthor('Fabien');
         $comment1->setEmail('fabien@example.com');
         $comment1->setText('This was a great conference.');
+        $comment1->setState('published');
         $manager->persist($comment1);

+        $comment2 = new Comment();
+        $comment2->setConference($amsterdam);
+        $comment2->setAuthor('Lucas');
+        $comment2->setEmail('lucas@example.com');
+        $comment2->setText('I think this one is going to be moderated.');
+        $manager->persist($comment2);
+
         $admin = new Admin();
         $admin->setRoles(['ROLE_ADMIN']);
         $admin->setUsername('admin');
```

Dla testów kontrolera zasymuluj walidację:

```diff
--- a/tests/Controller/ConferenceControllerTest.php
+++ b/tests/Controller/ConferenceControllerTest.php
@@ -2,6 +2,8 @@

 namespace App\Tests\Controller;

+use App\Repository\CommentRepository;
+use Doctrine\ORM\EntityManagerInterface;
 use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

 class ConferenceControllerTest extends WebTestCase
@@ -22,10 +24,16 @@ class ConferenceControllerTest extends WebTestCase
         $client->submitForm('Submit', [
             'comment[author]' => 'Fabien',
             'comment[text]' => 'Some feedback from an automated functional test',
-            'comment[email]' => 'me@automat.ed',
+            'comment[email]' => $email = 'me@automat.ed',
             'comment[photo]' => dirname(__DIR__, 2).'/public/images/under-construction.gif',
         ]);
         $this->assertResponseRedirects();
+
+        // simulate comment validation
+        $comment = self::getContainer()->get(CommentRepository::class)->findOneByEmail($email);
+        $comment->setState('published');
+        self::getContainer()->get(EntityManagerInterface::class)->flush();
+
         $client->followRedirect();
         $this->assertSelectorExists('div:contains("There are 2 comments")');
     }
```

W testach PHPUnit możesz uzyskać dostęp do dowolnego serwisu z kontenera poprzez `self::getContainer()->get()`. To umożliwia także dostęp do serwisów niepublicznych.

### Zrozumienie komponentu Messenger

Zarządzanie kodem asynchronicznym w Symfony to zadanie dla komponentu *Messenger*:

```bash
symfony composer req doctrine-messenger
```

Gdy jakaś logika powinna zostać wykonana asynchronicznie, należy wysłać wiadomość (*message*) do szyny wiadomości (*messenger bus*). Szyna zapisuje wiadomość w kolejce (*queue*) i natychmiast zwraca kontrolę, aby przepływ operacji mógł być jak najszybciej kontynuowany.

Proces *consumer* działa w tle w sposób ciągły, odczytuje nowe wiadomości z kolejki i wykonuje powiązaną logikę. Może on działać na tym samym serwerze co aplikacja webowa albo na osobnym.

Działa to bardzo podobnie do obsługi żądań HTTP, z tą różnicą, że tutaj nie mamy odpowiedzi.

### Tworzenie handlera wiadomości

Wiadomość to klasa danych, która nie powinna zawierać żadnej logiki. Będzie ona serializowana i przechowywana w kolejce, dlatego należy przechowywać tylko „proste”, możliwe do serializacji dane.

Utwórz klasę `CommentMessage`:

**src/Message/CommentMessage.php**
```php
namespace App\Message;

class CommentMessage
{
    public function __construct(
        private int $id,
        private array $context = [],
    ) {
    }

    public function getId(): int
    {
        return $this->id;
    }

    public function getContext(): array
    {
        return $this->context;
    }
}
```

W świecie Messenger nie mamy kontrolerów, lecz *message handlers* (obsługiwacze wiadomości).

Utwórz klasę `CommentMessageHandler` w nowej przestrzeni nazw `App\MessageHandler`, która będzie wiedziała jak obsługiwać wiadomości typu `CommentMessage`:

**src/MessageHandler/CommentMessageHandler.php**
```php
namespace App\MessageHandler;

use App\Message\CommentMessage;
use App\Repository\CommentRepository;
use App\SpamChecker;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
class CommentMessageHandler
{
    public function __construct(
        private EntityManagerInterface $entityManager,
        private SpamChecker $spamChecker,
        private CommentRepository $commentRepository,
    ) {
    }

    public function __invoke(CommentMessage $message)
    {
        $comment = $this->commentRepository->find($message->getId());
        if (!$comment) {
            return;
        }

        if (2 === $this->spamChecker->getSpamScore($comment, $message->getContext())) {
            $comment->setState('spam');
        } else {
            $comment->setState('published');
        }

        $this->entityManager->flush();
    }
}
```

Atrybut `#[AsMessageHandler]` pomaga Symfony automatycznie zarejestrować i skonfigurować klasę jako handler w Messengerze. Zgodnie z konwencją, logika handlera znajduje się w metodzie o nazwie `__invoke()`. Podpowiedź typu `CommentMessage` w argumencie tej metody informuje Messengera, którą klasę obsługiwać.

Zaktualizuj kontroler, aby korzystał z nowego systemu:

```diff
--- a/src/Controller/ConferenceController.php
+++ b/src/Controller/ConferenceController.php
@@ -5,20 +5,22 @@ namespace App\Controller;
 use App\Entity\Comment;
 use App\Entity\Conference;
 use App\Form\CommentType;
+use App\Message\CommentMessage;
 use App\Repository\CommentRepository;
 use App\Repository\ConferenceRepository;
-use App\SpamChecker;
 use Doctrine\ORM\EntityManagerInterface;
 use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
 use Symfony\Component\DependencyInjection\Attribute\Autowire;
 use Symfony\Component\HttpFoundation\Request;
 use Symfony\Component\HttpFoundation\Response;
+use Symfony\Component\Messenger\MessageBusInterface;
 use Symfony\Component\Routing\Attribute\Route;

 class ConferenceController extends AbstractController
 {
     public function __construct(
         private EntityManagerInterface $entityManager,
+        private MessageBusInterface $bus,
     ) {
     }

@@ -35,7 +37,6 @@ class ConferenceController extends AbstractController
         Request $request,
         Conference $conference,
         CommentRepository $commentRepository,
-        SpamChecker $spamChecker,
         #[Autowire('%photo_dir%')] string $photoDir,
     ): Response {
         $comment = new Comment();
@@ -50,6 +51,7 @@ class ConferenceController extends AbstractController
             }

             $this->entityManager->persist($comment);
+            $this->entityManager->flush();

             $context = [
                 'user_ip' => $request->getClientIp(),
@@ -57,11 +59,7 @@ class ConferenceController extends AbstractController
                 'referrer' => $request->headers->get('referer'),
                 'permalink' => $request->getUri(),
             ];
-            if (2 === $spamChecker->getSpamScore($comment, $context)) {
-                throw new \RuntimeException('Blatant spam, go away!');
-            }
-
-            $this->entityManager->flush();
+            $this->bus->dispatch(new CommentMessage($comment->getId(), $context));

             return $this->redirectToRoute('conference', ['slug' => $conference->getSlug()]);
         }
```

Zamiast polegać na sprawdzaczu spamu (Spam Checker), teraz wysyłamy wiadomość na szynę wiadomości. Handler decyduje, co z nią zrobić.

Osiągnęliśmy coś nieoczekiwanego — oddzieliliśmy kontroler od logiki sprawdzania spamu i przenieśliśmy ją do nowej klasy, handlera. To idealny przypadek użycia szyny wiadomości. Przetestuj kod — działa. Wszystko nadal wykonywane jest synchronicznie, ale kod jest prawdopodobnie już „lepszy”.

### Przechodzimy na prawdziwą asynchroniczność

Domyślnie handlery są wywoływane synchronicznie. Aby przejść na tryb asynchroniczny, musisz jawnie skonfigurować, której kolejki używać dla każdego handlera w pliku konfiguracyjnym `config/packages/messenger.yaml`:

```diff
--- a/config/packages/messenger.yaml
+++ b/config/packages/messenger.yaml
@@ -26,4 +26,4 @@ framework:
             Symfony\Component\Notifier\Message\SmsMessage: async

             # Route your messages to the transports
-            # 'App\Message\YourMessage': async
+            App\Message\CommentMessage: async
```

Konfiguracja informuje szynę wiadomości, aby instancje `App\Message\CommentMessage` były wysyłane do kolejki *async*, która jest zdefiniowana przez DSN `MESSENGER_TRANSPORT_DSN`, wskazujący na Doctrine zgodnie z ustawieniem w pliku `.env`. Mówiąc prosto: używamy PostgreSQL jako kolejki dla naszych wiadomości.

> [!TIP]
> Za kulisami Symfony wykorzystuje wbudowany, wydajny, skalowalny i transakcyjny system pub/sub PostgreSQL (`LISTEN`/`NOTIFY`). Możesz też przeczytać rozdział o RabbitMQ, jeśli chcesz używać go zamiast PostgreSQL jako brokera wiadomości.

### Konsumowanie wiadomości

Jeśli spróbujesz przesłać nowy komentarz, sprawdzacz spamu nie zostanie już wywołany. Dodaj wywołanie `error_log()` w metodzie `getSpamScore()`, aby to potwierdzić. Zamiast tego, wiadomość czeka w kolejce, gotowa do przetworzenia przez jakiś proces.

Jak możesz się domyślić, Symfony posiada odpowiednią komendę *consume*. Uruchom ją teraz:

```bash
symfony console messenger:consume async -vv
```

Powinna natychmiast przetworzyć wiadomość wysłaną dla przesłanego komentarza:

```
[OK] Consuming messages from transports "async".

 // The worker will automatically exit once it has received a stop signal via the messenger:stop-workers command.

 // Quit the worker with CONTROL-C.

11:30:20 INFO      [messenger] Received message App\Message\CommentMessage ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage"]
11:30:20 INFO      [http_client] Request: "POST https://80cea32be1f6.rest.akismet.com/1.1/comment-check"
11:30:20 INFO      [http_client] Response: "200 https://80cea32be1f6.rest.akismet.com/1.1/comment-check"
11:30:20 INFO      [messenger] Message App\Message\CommentMessage handled by App\MessageHandler\CommentMessageHandler::__invoke ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage","handler" => "App\MessageHandler\CommentMessageHandler::__invoke"]
11:30:20 INFO      [messenger] App\Message\CommentMessage was handled successfully (acknowledging to transport). ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage"]
```

Aktywność konsumenta wiadomości jest logowana, ale możesz uzyskać natychmiastową informację zwrotną w konsoli, przekazując flagę `-vv`. Powinieneś nawet być w stanie zauważyć wywołanie API Akismet.

Aby zatrzymać konsumenta, naciśnij `Ctrl+C`.

### Uruchamianie workerów w tle

Zamiast uruchamiać konsumenta za każdym razem, gdy publikujemy komentarz, i zatrzymywać go zaraz potem, chcemy uruchomić go na stałe bez potrzeby otwierania wielu okien lub zakładek terminala.

Symfony CLI może zarządzać takimi poleceniami działającymi w tle, używając flagi `-d` przy poleceniu `run`.

Uruchom ponownie konsumenta wiadomości, ale tym razem w tle:

```bash
symfony run -d --watch=config,src,templates,vendor/composer/installed.json symfony console messenger:consume async -vv
```

Opcja `--watch` informuje Symfony, że polecenie powinno zostać ponownie uruchomione za każdym razem, gdy nastąpi zmiana w systemie plików w katalogach `config/`, `src/`, `templates/` lub `vendor/`.

> [!NOTE]
> Nie używaj `-vv`, ponieważ otrzymasz zduplikowane wiadomości w `server:log` (zalogowane i wyświetlone w konsoli).

Jeśli konsument przestanie działać z jakiegoś powodu (limit pamięci, błąd itp.), zostanie automatycznie uruchomiony ponownie. A jeśli zbyt szybko przestanie działać wielokrotnie, Symfony CLI się podda.

Logi są strumieniowane przez `symfony server:log` razem z innymi logami pochodzącymi z PHP, serwera webowego i aplikacji:

```bash
symfony server:log
```

Użyj polecenia `server:status`, aby wyświetlić listę wszystkich workerów działających w tle dla bieżącego projektu:

```bash
symfony server:status

Web server listening on https://127.0.0.1:8000
  Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)
```

Aby zatrzymać workera, zatrzymaj serwer webowy lub zabij PID podany przez `server:status`:

```bash
kill 15774
```

### Ponowne próby nieudanych wiadomości

A co jeśli Akismet nie działa w momencie przetwarzania wiadomości? Nie ma to wpływu na osoby przesyłające komentarze, ale wiadomość jest utracona i spam nie zostaje sprawdzony.

Messenger posiada mechanizm ponawiania w przypadku wystąpienia wyjątku podczas obsługi wiadomości:

**config/packages/messenger.yaml**
```yaml
framework:
    messenger:
        failure_transport: failed

        transports:
            # https://symfony.com/doc/current/messenger.html#transport-configuration
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                options:
                    use_notify: true
                    check_delayed_interval: 1000
                retry_strategy:
                    max_retries: 3
                    multiplier: 2
            failed: 'doctrine://default?queue_name=failed'
            # sync: 'sync://'
```

Jeśli podczas obsługi wiadomości wystąpi problem, konsument spróbuje ponowić próbę 3 razy, zanim się podda. Ale zamiast całkowicie porzucić wiadomość, zapisze ją trwale w kolejce `failed`, która korzysta z innej tabeli bazy danych.

Sprawdź nieudane wiadomości i ponów ich przetwarzanie za pomocą następujących poleceń:

```bash
symfony console messenger:failed:show

symfony console messenger:failed:retry
```

### Uruchamianie workerów na Platform.sh

Aby konsumować wiadomości z PostgreSQL, musimy nieprzerwanie uruchamiać komendę `messenger:consume`. Na Platform.sh jest to rola *workera*:

**.platform.app.yaml**
```yaml
workers:
    messenger:
        commands:
            # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
            start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async
```

Podobnie jak w przypadku Symfony CLI, Platform.sh zarządza restartami i logami.

Aby uzyskać logi z workera, użyj:

```bash
symfony cloud:logs --worker=messages all
```

### Sprawdź również:
- [Kurs SymfonyCasts dotyczący komponentu Messenger](https://symfonycasts.com/screencast/messenger);
- Architektura [Enterprise Service Bus](https://en.wikipedia.org/wiki/Enterprise_service_bus) i [wzorzec CQRS](https://martinfowler.com/bliki/CQRS.html);
- [Dokumentacja Symfony Messenger](https://symfony.com/doc/current/messenger.html).

---

- **Poprzednia strona:** [Testowanie](17-tests.md)
- **Następna strona:** [Podejmowanie decyzji z pomocą Workflow](19-workflow.md)
