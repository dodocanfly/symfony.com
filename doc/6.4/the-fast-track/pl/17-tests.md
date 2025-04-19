## Testowanie

W miarę dodawania coraz większej liczby funkcji do aplikacji, nadchodzi właściwy moment, by porozmawiać o testowaniu.

*Ciekawostka*: podczas pisania testów w tym rozdziale znalazłem błąd.

Symfony korzysta z PHPUnit do testów jednostkowych. Zainstalujmy go:

```bash
symfony composer req phpunit --dev
```

### Pisanie testów jednostkowych

`SpamChecker` to pierwsza klasa, dla której napiszemy testy. Wygeneruj test jednostkowy:

```bash
symfony console make:test TestCase SpamCheckerTest
```

Testowanie klasy SpamChecker jest wyzwaniem, ponieważ zdecydowanie nie chcemy łączyć się z API Akismeta. Będziemy więc je *mockować*.

Napiszmy pierwszy test dla sytuacji, gdy API zwraca błąd:

```diff
--- a/tests/SpamCheckerTest.php
+++ b/tests/SpamCheckerTest.php
@@ -2,12 +2,26 @@

 namespace App\Tests;

+use App\Entity\Comment;
+use App\SpamChecker;
 use PHPUnit\Framework\TestCase;
+use Symfony\Component\HttpClient\MockHttpClient;
+use Symfony\Component\HttpClient\Response\MockResponse;
+use Symfony\Contracts\HttpClient\ResponseInterface;

 class SpamCheckerTest extends TestCase
 {
-    public function testSomething(): void
+    public function testSpamScoreWithInvalidRequest(): void
     {
-        $this->assertTrue(true);
+        $comment = new Comment();
+        $comment->setCreatedAtValue();
+        $context = [];
+
+        $client = new MockHttpClient([new MockResponse('invalid', ['response_headers' => ['x-akismet-debug-help: Invalid key']])]);
+        $checker = new SpamChecker($client, 'abcde');
+
+        $this->expectException(\RuntimeException::class);
+        $this->expectExceptionMessage('Unable to check for spam: invalid (Invalid key).');
+        $checker->getSpamScore($comment, $context);
     }
 }
```

Klasa `MockHttpClient` pozwala na mockowanie dowolnego serwera HTTP. Przyjmuje tablicę instancji `MockResponse`, które zawierają oczekiwane ciało odpowiedzi oraz nagłówki.

Następnie wywołujemy metodę `getSpamScore()` i sprawdzamy, czy zostanie rzucony wyjątek za pomocą metody `expectException()` z PHPUnit.

Uruchom testy, aby upewnić się, że przechodzą:

```bash
symfony php bin/phpunit
```

Dodajmy testy dla „szczęśliwej ścieżki”:

```diff
--- a/tests/SpamCheckerTest.php
+++ b/tests/SpamCheckerTest.php
@@ -24,4 +24,32 @@ class SpamCheckerTest extends TestCase
         $this->expectExceptionMessage('Unable to check for spam: invalid (Invalid key).');
         $checker->getSpamScore($comment, $context);
     }
+
+    /**
+     * @dataProvider provideComments
+     */
+    public function testSpamScore(int $expectedScore, ResponseInterface $response, Comment $comment, array $context)
+    {
+        $client = new MockHttpClient([$response]);
+        $checker = new SpamChecker($client, 'abcde');
+
+        $score = $checker->getSpamScore($comment, $context);
+        $this->assertSame($expectedScore, $score);
+    }
+
+    public static function provideComments(): iterable
+    {
+        $comment = new Comment();
+        $comment->setCreatedAtValue();
+        $context = [];
+
+        $response = new MockResponse('', ['response_headers' => ['x-akismet-pro-tip: discard']]);
+        yield 'blatant_spam' => [2, $response, $comment, $context];
+
+        $response = new MockResponse('true');
+        yield 'spam' => [1, $response, $comment, $context];
+
+        $response = new MockResponse('false');
+        yield 'ham' => [0, $response, $comment, $context];
+    }
 }
```

Dostawcy danych (data providers) w PHPUnit pozwalają na wielokrotne użycie tej samej logiki testu dla różnych przypadków.

### Pisanie testów funkcjonalnych dla kontrolerów

Testowanie kontrolerów różni się nieco od testowania „zwykłych” klas PHP, ponieważ chcemy je uruchamiać w kontekście żądania HTTP.

Utwórz test funkcjonalny dla kontrolera `Conference`:

**tests/Controller/ConferenceControllerTest.php**
```php
namespace App\Tests\Controller;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

class ConferenceControllerTest extends WebTestCase
{
    public function testIndex()
    {
        $client = static::createClient();
        $client->request('GET', '/');

        $this->assertResponseIsSuccessful();
        $this->assertSelectorTextContains('h2', 'Give your feedback');
    }
}
```

Zamiast `PHPUnit\Framework\TestCase`, używamy `Symfony\Bundle\FrameworkBundle\Test\WebTestCase` jako klasy bazowej — daje nam to wygodną abstrakcję dla testów funkcjonalnych.

Zmiennej `$client` używamy do symulowania przeglądarki. Zamiast wykonywać żądania HTTP do serwera, od razu wywołuje aplikację Symfony. Ta strategia ma kilka zalet: jest znacznie szybsza (bez round-tripów klient-serwer), a także pozwala testom sprawdzać stan usług po każdym żądaniu HTTP.

Pierwszy test sprawdza, czy strona główna zwraca odpowiedź HTTP 200.

Asercje takie jak `assertResponseIsSuccessful` są dodatkiem do PHPUnit i upraszczają pracę. Symfony oferuje ich całkiem sporo.

> [!TIP]
> Użyliśmy `/` jako adresu URL zamiast generowania go przez router — celowo, ponieważ testowanie adresów URL widocznych dla użytkownika końcowego to też część testów. Zmiana ścieżki trasy sprawi, że test się „wywali” — co przypomni Ci, że warto ustawić przekierowanie ze starego adresu na nowy (np. dla SEO czy linkujących zewnętrznych stron).

### Konfigurowanie środowiska testowego

Domyślnie testy PHPUnit uruchamiane są w środowisku `test`, zdefiniowanym w pliku konfiguracyjnym PHPUnit:

**phpunit.xml.dist**
```xml
<phpunit>
    <php>
        <ini name="error_reporting" value="-1" />
        <server name="APP_ENV" value="test" force="true" />
        <server name="SHELL_VERBOSITY" value="-1" />
        <server name="SYMFONY_PHPUNIT_REMOVE" value="" />
        <server name="SYMFONY_PHPUNIT_VERSION" value="8.5" />
        ...
```

Aby testy działały, musimy ustawić sekret `AKISMET_KEY` w środowisku testowym (`test`):

```bash
symfony console secrets:set AKISMET_KEY --env=test
```

### Praca z testową bazą danych

Jak już widzieliśmy, Symfony CLI automatycznie ustawia zmienną `DATABASE_URL`. Gdy `APP_ENV` to `test` (jak przy uruchamianiu PHPUnit), nazwa bazy zmienia się z `app` na `app_test`, dzięki czemu testy mają swoją własną bazę:

**config/packages/doctrine.yaml**
```yaml
when@test:
    doctrine:
        dbal:
            # "TEST_TOKEN" is typically set by ParaTest
            dbname_suffix: '_test%env(default::TEST_TOKEN)%'
```

To bardzo ważne, ponieważ testy wymagają stabilnych danych i zdecydowanie nie chcemy nadpisywać danych z bazy deweloperskiej.

Zanim uruchomimy testy, musimy „zainicjalizować” testową (`test`) bazę danych (utworzyć ją i uruchomić migracje):

```bash
symfony console doctrine:database:create --env=test
symfony console doctrine:migrations:migrate -n --env=test
```

> [!NOTE]
> Na Linuksie i podobnych systemach można użyć `APP_ENV=test` zamiast `--env=test`:
> ```bash
> APP_ENV=test symfony console doctrine:database:create
> ```

Jeśli teraz uruchomisz testy, PHPUnit nie będzie już korzystać z bazy deweloperskiej. Aby uruchomić tylko nowe testy, podaj ścieżkę do klasy testowej:

```bash
symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php
```

> [!TIP]
> Gdy test się nie powiedzie, warto podejrzeć obiekt odpowiedzi. Uzyskaj go przez `$client->getResponse()` i `echo`, aby zobaczyć jak wygląda.

### Definiowanie „fikstur”

Aby testować listę komentarzy, paginację i wysyłanie formularza, musimy mieć dane w bazie. I chcemy, by były takie same przy każdym uruchomieniu testów. Do tego właśnie służą fikstury.

Zainstaluj paczkę Doctrine Fixtures:

```bash
symfony composer req orm-fixtures --dev
```

Podczas instalacji utworzono katalog `src/DataFixtures/` z przykładową klasą gotową do dostosowania. Dodaj na razie dwie konferencje i jeden komentarz:

```diff
--- a/src/DataFixtures/AppFixtures.php
+++ b/src/DataFixtures/AppFixtures.php
@@ -2,6 +2,8 @@

 namespace App\DataFixtures;

+use App\Entity\Comment;
+use App\Entity\Conference;
 use Doctrine\Bundle\FixturesBundle\Fixture;
 use Doctrine\Persistence\ObjectManager;

@@ -9,8 +11,24 @@ class AppFixtures extends Fixture
 {
     public function load(ObjectManager $manager): void
     {
-        // $product = new Product();
-        // $manager->persist($product);
+        $amsterdam = new Conference();
+        $amsterdam->setCity('Amsterdam');
+        $amsterdam->setYear('2019');
+        $amsterdam->setInternational(true);
+        $manager->persist($amsterdam);
+
+        $paris = new Conference();
+        $paris->setCity('Paris');
+        $paris->setYear('2020');
+        $paris->setInternational(false);
+        $manager->persist($paris);
+
+        $comment1 = new Comment();
+        $comment1->setConference($amsterdam);
+        $comment1->setAuthor('Fabien');
+        $comment1->setEmail('fabien@example.com');
+        $comment1->setText('This was a great conference.');
+        $manager->persist($comment1);

         $manager->flush();
     }
```

Podczas ładowania fikstur wszystkie dane są usuwane — w tym użytkownik admin. Aby tego uniknąć, dodajmy go do fikstur:

```diff
--- a/src/DataFixtures/AppFixtures.php
+++ b/src/DataFixtures/AppFixtures.php
@@ -2,13 +2,20 @@

 namespace App\DataFixtures;

+use App\Entity\Admin;
 use App\Entity\Comment;
 use App\Entity\Conference;
 use Doctrine\Bundle\FixturesBundle\Fixture;
 use Doctrine\Persistence\ObjectManager;
+use Symfony\Component\PasswordHasher\Hasher\PasswordHasherFactoryInterface;

 class AppFixtures extends Fixture
 {
+    public function __construct(
+        private PasswordHasherFactoryInterface $passwordHasherFactory,
+    ) {
+    }
+
     public function load(ObjectManager $manager): void
     {
         $amsterdam = new Conference();
@@ -30,6 +37,12 @@ class AppFixtures extends Fixture
         $comment1->setText('This was a great conference.');
         $manager->persist($comment1);

+        $admin = new Admin();
+        $admin->setRoles(['ROLE_ADMIN']);
+        $admin->setUsername('admin');
+        $admin->setPassword($this->passwordHasherFactory->getPasswordHasher(Admin::class)->hash('admin'));
+        $manager->persist($admin);
+
         $manager->flush();
     }
 }
```

> [!TIP]
> Nie pamiętasz, jakiego serwisu użyć do konkretnego zadania? Użyj `debug:autowiring` z jakimś słowem kluczowym.
> ```bash
> symfony console debug:autowiring hasher
> ```

### Ładowanie fikstur

Załaduj fikstury dla środowiska/bazy testowej:

```bash
symfony console doctrine:fixtures:load --env=test
```

### Przeglądanie strony w testach funkcjonalnych

Jak widzieliśmy, klient HTTP w testach symuluje przeglądarkę — możemy więc poruszać się po stronie jak przy użyciu przeglądarki bezgłowej.

Dodaj test, który klika na stronę konferencji ze strony głównej:

```diff
--- a/tests/Controller/ConferenceControllerTest.php
+++ b/tests/Controller/ConferenceControllerTest.php
@@ -14,4 +14,19 @@ class ConferenceControllerTest extends WebTestCase
         $this->assertResponseIsSuccessful();
         $this->assertSelectorTextContains('h2', 'Give your feedback');
     }
+
+    public function testConferencePage()
+    {
+        $client = static::createClient();
+        $crawler = $client->request('GET', '/');
+
+        $this->assertCount(2, $crawler->filter('h4'));
+
+        $client->clickLink('View');
+
+        $this->assertPageTitleContains('Amsterdam');
+        $this->assertResponseIsSuccessful();
+        $this->assertSelectorTextContains('h2', 'Amsterdam 2019');
+        $this->assertSelectorExists('div:contains("There are 1 comments")');
+    }
 }
```

Opiszmy, co dokładnie dzieje się w tym teście, w prostym języku:
- Tak jak w pierwszym teście, przechodzimy na stronę główną;
- Metoda `request()` zwraca instancję klasy `Crawler`, która pomaga znaleźć elementy na stronie (takie jak linki, formularze lub cokolwiek, do czego można się odwołać za pomocą selektorów CSS lub XPath);
- Dzięki selektorowi CSS sprawdzamy, że na stronie głównej znajdują się dwie konferencje;
- Następnie klikamy w link "View" (ponieważ nie można kliknąć więcej niż jednego linku jednocześnie, Symfony automatycznie wybiera pierwszy, jaki znajdzie);
- Sprawdzamy tytuł strony, odpowiedź HTTP oraz nagłówek `<h2>` na stronie, aby upewnić się, że znajdujemy się na właściwej stronie (moglibyśmy również sprawdzić, czy pasuje odpowiednia trasa — route);
- Na koniec sprawdzamy, że na stronie znajduje się dokładnie jeden komentarz. `div:contains()` nie jest poprawnym selektorem CSS, ale Symfony zawiera kilka przydatnych rozszerzeń, zapożyczonych z jQuery.

Zamiast klikać po tekście (np. „View”), można też wybrać link za pomocą selektora CSS:

```php
$client->click($crawler->filter('h4 + p a')->link());
```

Sprawdź, czy test przechodzi (zielony):

```bash
symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php
```

### Wysyłanie formularza w teście funkcjonalnym

Chcesz wejść na wyższy poziom? Spróbuj dodać nowy komentarz ze zdjęciem do konferencji z poziomu testu, symulując wysyłkę formularza. Ambitne? Zobacz kod — nie jest bardziej skomplikowany niż to, co już pisaliśmy:

```diff
--- a/tests/Controller/ConferenceControllerTest.php
+++ b/tests/Controller/ConferenceControllerTest.php
@@ -29,4 +29,19 @@ class ConferenceControllerTest extends WebTestCase
         $this->assertSelectorTextContains('h2', 'Amsterdam 2019');
         $this->assertSelectorExists('div:contains("There are 1 comments")');
     }
+
+    public function testCommentSubmission()
+    {
+        $client = static::createClient();
+        $client->request('GET', '/conference/amsterdam-2019');
+        $client->submitForm('Submit', [
+            'comment[author]' => 'Fabien',
+            'comment[text]' => 'Some feedback from an automated functional test',
+            'comment[email]' => 'me@automat.ed',
+            'comment[photo]' => dirname(__DIR__, 2).'/public/images/under-construction.gif',
+        ]);
+        $this->assertResponseRedirects();
+        $client->followRedirect();
+        $this->assertSelectorExists('div:contains("There are 2 comments")');
+    }
 }
```

Aby wysłać formularz przez `submitForm()`, znajdź nazwy inputów przez narzędzia deweloperskie przeglądarki lub panel Symfony Profiler. Zwróć uwagę na sprytne ponowne użycie obrazka „w budowie”!

Uruchom ponownie testy, by sprawdzić, czy wszystko przeszło:

```bash
symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php
```

Chcesz sprawdzić wynik w przeglądarce? Zatrzymaj serwer i uruchom go ponownie w środowisku testowym:

```bash
symfony server:stop
APP_ENV=test symfony server:start -d
```

![tests-add-comment](https://symfony.com/doc/6.4en//the-fast-track/_images/tests-add-comment.png)

### Ponowne ładowanie fikstur

Jeśli uruchomisz testy ponownie, mogą się nie powieść. W bazie jest teraz więcej komentarzy, więc asercja sprawdzająca ich liczbę się nie powiedzie. Trzeba zresetować stan bazy przed każdym testem — poprzez ponowne ładowanie fikstur:

```bash
symfony console doctrine:fixtures:load --env=test
symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php
```

### Automatyzowanie workflow za pomocą Makefile

Zapamiętywanie komend do uruchomienia testów jest uciążliwe. Warto je udokumentować. Ale dokumentacja to ostateczność — lepiej zautomatyzować codzienne czynności. To będzie dokumentacją, ułatwi pracę innym deweloperom i przyspieszy działanie.

Użycie `Makefile` to jedno z rozwiązań.

**Makefile**
```Makefile
SHELL := /bin/bash

tests:
	symfony console doctrine:database:drop --force --env=test || true
	symfony console doctrine:database:create --env=test
	symfony console doctrine:migrations:migrate -n --env=test
	symfony console doctrine:fixtures:load -n --env=test
	symfony php bin/phpunit $(MAKECMDGOALS)
.PHONY: tests
```

> [!WARNING]
> W regule Makefile wcięcie **musi** być pojedynczym znakiem tabulacji (nie spacjami!).

Zwróć uwagę na flagę `-n` w komendzie Doctrine — to globalna flaga Symfony, która sprawia, że komenda nie jest interaktywna.

Chcesz uruchomić testy? Użyj `make tests`:

```bash
make tests
```

### Resetowanie bazy po każdym teście

Resetowanie bazy danych po każdym uruchomieniu testów jest przydatne, ale posiadanie całkowicie niezależnych testów jest jeszcze lepsze. Nie chcemy, aby jeden test polegał na wynikach poprzednich testów. Zmiana kolejności testów nie powinna wpływać na ich wynik. Jak się zaraz przekonamy, w tym momencie tak jednak nie jest.

Przenieś test `testConferencePage` po `testCommentSubmission`:

```diff
--- a/tests/Controller/ConferenceControllerTest.php
+++ b/tests/Controller/ConferenceControllerTest.php
@@ -15,21 +15,6 @@ class ConferenceControllerTest extends WebTestCase
         $this->assertSelectorTextContains('h2', 'Give your feedback');
     }

-    public function testConferencePage()
-    {
-        $client = static::createClient();
-        $crawler = $client->request('GET', '/');
-
-        $this->assertCount(2, $crawler->filter('h4'));
-
-        $client->clickLink('View');
-
-        $this->assertPageTitleContains('Amsterdam');
-        $this->assertResponseIsSuccessful();
-        $this->assertSelectorTextContains('h2', 'Amsterdam 2019');
-        $this->assertSelectorExists('div:contains("There are 1 comments")');
-    }
-
     public function testCommentSubmission()
     {
         $client = static::createClient();
@@ -44,4 +29,19 @@ class ConferenceControllerTest extends WebTestCase
         $client->followRedirect();
         $this->assertSelectorExists('div:contains("There are 2 comments")');
     }
+
+    public function testConferencePage()
+    {
+        $client = static::createClient();
+        $crawler = $client->request('GET', '/');
+
+        $this->assertCount(2, $crawler->filter('h4'));
+
+        $client->clickLink('View');
+
+        $this->assertPageTitleContains('Amsterdam');
+        $this->assertResponseIsSuccessful();
+        $this->assertSelectorTextContains('h2', 'Amsterdam 2019');
+        $this->assertSelectorExists('div:contains("There are 1 comments")');
+    }
 }
```

Testy się wywalają.

Aby zresetować bazę danych pomiędzy testami, zainstaluj pakiet `DoctrineTestBundle`:

```bash
symfony composer req "dama/doctrine-test-bundle:^8" --dev
```

Będziesz musiał potwierdzić wykonanie przepisu (ponieważ nie jest to pakiet „oficjalnie” wspierany):

```
Symfony operations: 1 recipe (a5c79a9ff21bc3ae26d9bb25f1262ed7)
  -  WARNING  dama/doctrine-test-bundle (>=4.0): From github.com/symfony/recipes-contrib:master
    The recipe for this package comes from the "contrib" repository, which is open to community contributions.
    Review the recipe at https://github.com/symfony/recipes-contrib/tree/master/dama/doctrine-test-bundle/4.0

    Do you want to execute this recipe?
    [y] Yes
    [n] No
    [a] Yes for all packages, only for the current installation session
    [p] Yes permanently, never ask again for this project
    (defaults to n): p
```

I gotowe. Wszelkie zmiany dokonane w testach są teraz automatycznie wycofywane na końcu każdego testu.

Testy powinny znowu przechodzić pomyślnie:

```bash
make tests
```

### Użycie prawdziwej przeglądarki w testach funkcjonalnych

Testy funkcjonalne używają specjalnej przeglądarki, która wywołuje Symfony bezpośrednio. Ale możesz też użyć prawdziwej przeglądarki i prawdziwej warstwy HTTP dzięki Symfony Panther:

```bash
symfony composer req panther --dev
```

Możesz pisać testy korzystające z Google Chrome, wprowadzając kilka zmian:

```diff
--- a/tests/Controller/ConferenceControllerTest.php
+++ b/tests/Controller/ConferenceControllerTest.php
@@ -2,13 +2,13 @@

 namespace App\Tests\Controller;

-use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
+use Symfony\Component\Panther\PantherTestCase;

-class ConferenceControllerTest extends WebTestCase
+class ConferenceControllerTest extends PantherTestCase
 {
     public function testIndex()
     {
-        $client = static::createClient();
+        $client = static::createPantherClient(['external_base_uri' => rtrim($_SERVER['SYMFONY_PROJECT_DEFAULT_ROUTE_URL'], '/')]);
         $client->request('GET', '/');

         $this->assertResponseIsSuccessful();
```

Zmienna `SYMFONY_PROJECT_DEFAULT_ROUTE_URL` zawiera adres lokalnego serwera.

### Wybór odpowiedniego rodzaju testu

Stworzyliśmy trzy rodzaje testów. Choć tylko test jednostkowy był generowany przez `maker`, można było wygenerować też inne.

```bash
symfony console make:test WebTestCase Controller\\ConferenceController

symfony console make:test PantherTestCase Controller\\ConferenceController
```

`Maker bundle` obsługuje generowanie następujących typów testów, w zależności od potrzeb:
- `TestCase`: Podstawowe testy PHPUnit;
- `KernelTestCase`: Podstawowe testy z dostępem do usług Symfony;
- `WebTestCase`: Testy symulujące przeglądarkę, ale bez JavaScriptu;
- `ApiTestCase`: Testy API;
- `PantherTestCase`: Testy end-to-end, z użyciem prawdziwej przeglądarki lub klienta HTTP i prawdziwego serwera.

### Uruchamianie testów funkcjonalnych Black-Box z Blackfire

Innym sposobem uruchamiania testów funkcjonalnych jest użycie [Blackfire player](https://blackfire.io/player). Poza tym, co oferują testy funkcjonalne, pozwala też na testy wydajnościowe.

Więcej w sekcji o [wydajności](29-performance.md).

### Sprawdź również:
- [Lista asercji zdefiniowanych przez Symfony](https://symfony.com/doc/current/testing/functional_tests_assertions.html) dla testów funkcjonalnych;
- [Dokumentacja PHPUnit](https://phpunit.de/documentation.html);
- [Biblioteka Faker](https://github.com/FakerPHP/Faker) do generowania realistycznych danych testowych;
- [Dokumentacja komponentu CssSelector](https://symfony.com/doc/current/components/css_selector.html);
- Biblioteka [Symfony Panther](https://github.com/symfony/panther) do testów przeglądarkowych i crawl’owania;
- [Dokumentacja Make/Makefile](https://www.gnu.org/software/make/manual/make.html).

---

- **Poprzednia strona:** [Zapobieganie spamowi za pomocą API](16-spam.md)
- **Następna strona:** [Przechodzenie na tryb asynchroniczny](18-async.md)
