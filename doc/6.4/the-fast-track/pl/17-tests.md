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

Zainstaluj paczkę Doctrine Fixtures.

Podczas instalacji utworzono katalog `src/DataFixtures/` z przykładową klasą gotową do dostosowania. Dodaj na razie dwie konferencje i jeden komentarz.

Podczas ładowania fikstur wszystkie dane są usuwane — w tym użytkownik admin. Aby tego uniknąć, dodajmy go do fikstur.

Nie pamiętasz, jakiego serwisu użyć do konkretnego zadania? Użyj `debug:autowiring` z jakimś słowem kluczowym.

### Ładowanie fikstur

Załaduj fikstury dla środowiska/bazy testowej:

### Przeglądanie strony w testach funkcjonalnych

Jak widzieliśmy, klient HTTP w testach symuluje przeglądarkę — możemy więc poruszać się po stronie jak przy użyciu przeglądarki bezgłowej.

Dodaj test, który klika na stronę konferencji ze strony głównej:

Opis testu „po ludzku”:
- Idziemy na stronę główną;
- Metoda `request()` zwraca instancję `Crawler`, która pomaga znaleźć elementy na stronie (linki, formularze, itp.);
- Dzięki selektorowi CSS sprawdzamy, czy są dwie konferencje na stronie głównej;
- Klikamy na link „View” (ponieważ nie można kliknąć więcej niż jeden link na raz, Symfony automatycznie wybiera pierwszy znaleziony);
- Sprawdzamy tytuł strony, odpowiedź HTTP i nagłówek `<h2>`, by upewnić się, że jesteśmy na właściwej stronie;
- Na koniec sprawdzamy, że jest jeden komentarz. `div:contains()` nie jest poprawnym selektorem CSS, ale Symfony ma miłe dodatki zapożyczone z jQuery.

Zamiast klikać po tekście (np. „View”), można też wybrać link za pomocą selektora CSS.

Sprawdź, czy test przechodzi (zielony).

### Wysyłanie formularza w teście funkcjonalnym

Chcesz wejść na wyższy poziom? Spróbuj dodać nowy komentarz ze zdjęciem do konferencji z poziomu testu, symulując wysyłkę formularza. Ambitne? Zobacz kod — nie jest bardziej skomplikowany niż to, co już pisaliśmy.

Aby wysłać formularz przez `submitForm()`, znajdź nazwy inputów przez narzędzia deweloperskie przeglądarki lub panel Symfony Profiler. Zwróć uwagę na sprytne ponowne użycie obrazka „w budowie”!

Uruchom ponownie testy, by sprawdzić, czy wszystko przeszło.

Chcesz sprawdzić wynik w przeglądarce? Zatrzymaj serwer i uruchom go ponownie w środowisku testowym.

### Ponowne ładowanie fikstur

Jeśli uruchomisz testy ponownie, mogą się nie powieść. W bazie jest teraz więcej komentarzy, więc asercja sprawdzająca ich liczbę się nie powiedzie. Trzeba zresetować stan bazy przed każdym testem — poprzez ponowne ładowanie fikstur.

### Automatyzowanie workflow za pomocą Makefile

Zapamiętywanie komend do uruchomienia testów jest uciążliwe. Warto je udokumentować. Ale dokumentacja to ostateczność — lepiej zautomatyzować codzienne czynności. To będzie dokumentacją, ułatwi pracę innym deweloperom i przyspieszy działanie.

Makefile to jedno z rozwiązań.

W regule Makefile wcięcie musi być pojedynczym znakiem tabulacji (nie spacjami!).

Zwróć uwagę na flagę `-n` w komendzie Doctrine — to globalna flaga Symfony, która sprawia, że komenda nie jest interaktywna.

Chcesz uruchomić testy? Użyj `make tests`.

### Resetowanie bazy po każdym teście

Resetowanie bazy między uruchomieniami testów to dobre podejście. Ale jeszcze lepiej, gdy testy są całkowicie niezależne. Jeden test nie powinien polegać na efektach poprzednich. Zmiana kolejności nie powinna zmieniać wyniku. Obecnie tak nie jest.

Przenieś test `testConferencePage` po `testCommentSubmission`.

Testy się wywalają.

Aby resetować bazę między testami, zainstaluj `DoctrineTestBundle`.

Trzeba będzie potwierdzić instalację recepty (to nie „oficjalnie” wspierana paczka).

I gotowe. Wszelkie zmiany w bazie wykonane przez testy są teraz automatycznie cofane po zakończeniu testu.

Testy powinny znów przechodzić.

### Użycie prawdziwej przeglądarki w testach funkcjonalnych

Testy funkcjonalne używają specjalnej przeglądarki, która wywołuje Symfony bezpośrednio. Ale możesz też użyć prawdziwej przeglądarki i prawdziwej warstwy HTTP dzięki Symfony Panther.

Możesz pisać testy korzystające z Google Chrome, wprowadzając kilka zmian.

Zmienna `SYMFONY_PROJECT_DEFAULT_ROUTE_URL` zawiera adres lokalnego serwera.

### Wybór odpowiedniego rodzaju testu

Stworzyliśmy trzy rodzaje testów. Choć tylko test jednostkowy był generowany przez `maker`, można było wygenerować też inne.

`Maker bundle` obsługuje generowanie następujących typów testów, w zależności od potrzeb:
- `TestCase`: Podstawowe testy PHPUnit;
- `KernelTestCase`: Podstawowe testy z dostępem do usług Symfony;
- `WebTestCase`: Testy symulujące przeglądarkę, ale bez JavaScriptu;
- `ApiTestCase`: Testy API;
- `PantherTestCase`: Testy end-to-end, z użyciem prawdziwej przeglądarki lub klienta HTTP i prawdziwego serwera.

### Uruchamianie testów black-box z Blackfire

Innym sposobem uruchamiania testów funkcjonalnych jest użycie `Blackfire player`. Poza tym, co oferują testy funkcjonalne, pozwala też na testy wydajnościowe.

Więcej w sekcji o wydajności.

### Dalsze kroki:
- Lista asercji zdefiniowanych przez Symfony dla testów funkcjonalnych;
- Dokumentacja PHPUnit;
- Biblioteka Faker do generowania realistycznych danych testowych;
- Dokumentacja komponentu CssSelector;
- Biblioteka Symfony Panther do testów przeglądarkowych i crawl’owania;
- Dokumentacja Make/Makefile.

--- 

Chcesz, żebym coś skrócił, podsumował albo zrobił checklistę do działania?
