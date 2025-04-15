## Zapobieganie spamowi za pomocą API

Każdy może przesłać opinię. Nawet roboty, spamerzy i inni. Możemy dodać do formularza jakiś „captcha”, aby w pewnym stopniu chronić się przed robotami, albo możemy skorzystać z zewnętrznego API.

Zdecydowałem się użyć darmowej usługi [Akismet](https://akismet.com/), aby pokazać, jak wywoływać API i jak wykonywać te wywołania „poza głównym nurtem” (asynchronicznie).

### Rejestracja w Akismet

Zarejestruj się na stronie [akismet.com](https://akismet.com/), aby założyć darmowe konto i uzyskać klucz API Akismet.

### Zależność od komponentu HTTPClient Symfony

Zamiast korzystać z biblioteki, która abstrahuje API Akismet, wykonamy wszystkie wywołania API bezpośrednio. Samodzielne wykonywanie żądań HTTP jest bardziej wydajne (i pozwala korzystać ze wszystkich narzędzi debugowania Symfony, takich jak integracja z Symfony Profiler).

### Projektowanie klasy do sprawdzania spamu

Utwórz nową klasę w katalogu `src/` o nazwie `SpamChecker`, aby zawrzeć w niej logikę wywoływania API Akismet oraz interpretowania jego odpowiedzi:

**src/SpamChecker.php**
```php
namespace App;

use App\Entity\Comment;
use Symfony\Contracts\HttpClient\HttpClientInterface;

class SpamChecker
{
    private $endpoint;

    public function __construct(
        private HttpClientInterface $client,
        string $akismetKey,
    ) {
        $this->endpoint = sprintf('https://%s.rest.akismet.com/1.1/comment-check', $akismetKey);
    }

    /**
     * @return int Spam score: 0: not spam, 1: maybe spam, 2: blatant spam
     *
     * @throws \RuntimeException if the call did not work
     */
    public function getSpamScore(Comment $comment, array $context): int
    {
        $response = $this->client->request('POST', $this->endpoint, [
            'body' => array_merge($context, [
                'blog' => 'https://guestbook.example.com',
                'comment_type' => 'comment',
                'comment_author' => $comment->getAuthor(),
                'comment_author_email' => $comment->getEmail(),
                'comment_content' => $comment->getText(),
                'comment_date_gmt' => $comment->getCreatedAt()->format('c'),
                'blog_lang' => 'en',
                'blog_charset' => 'UTF-8',
                'is_test' => true,
            ]),
        ]);

        $headers = $response->getHeaders();
        if ('discard' === ($headers['x-akismet-pro-tip'][0] ?? '')) {
            return 2;
        }

        $content = $response->getContent();
        if (isset($headers['x-akismet-debug-help'][0])) {
            throw new \RuntimeException(sprintf('Unable to check for spam: %s (%s).', $content, $headers['x-akismet-debug-help'][0]));
        }

        return 'true' === $content ? 1 : 0;
    }
}
```

Metoda `request()` klienta HTTP wysyła żądanie POST do adresu URL Akismet (`$this->endpoint`) i przekazuje tablicę parametrów.

Metoda `getSpamScore()` zwraca jedną z trzech wartości w zależności od odpowiedzi API:
- `2` – jeśli komentarz to "oczywisty spam";
- `1` – jeśli komentarz może być spamem;
- `0` – jeśli komentarz nie jest spamem (tzw. "ham").

> [!TIP]
> Użyj specjalnego adresu e-mail `akismet-guaranteed-spam@example.com`, aby wymusić wynik jako spam.

### Używanie zmiennych środowiskowych

Klasa `SpamChecker` opiera się na argumencie `$akismetKey`. Podobnie jak w przypadku katalogu do przesyłania plików, możemy wstrzyknąć go za pomocą adnotacji `Autowire`:

```diff
--- a/src/SpamChecker.php
+++ b/src/SpamChecker.php
@@ -3,6 +3,7 @@
 namespace App;

 use App\Entity\Comment;
+use Symfony\Component\DependencyInjection\Attribute\Autowire;
 use Symfony\Contracts\HttpClient\HttpClientInterface;

 class SpamChecker
@@ -11,7 +12,7 @@ class SpamChecker

     public function __construct(
         private HttpClientInterface $client,
-        string $akismetKey,
+        #[Autowire('%env(AKISMET_KEY)%')] string $akismetKey,
     ) {
         $this->endpoint = sprintf('https://%s.rest.akismet.com/1.1/comment-check', $akismetKey);
     }
```

Zdecydowanie nie chcemy na sztywno wpisywać wartości klucza Akismet w kodzie, dlatego używamy zamiast tego zmiennej środowiskowej (`AKISMET_KEY`).

Każdy programista powinien sam ustawić „prawdziwą” zmienną środowiskową lub przechowywać wartość w pliku `.env.local`:

**.env.local**
```env
AKISMET_KEY=abcdef
```

W środowisku produkcyjnym należy zdefiniować „prawdziwą” zmienną środowiskową.

To rozwiązanie działa dobrze, ale zarządzanie wieloma zmiennymi środowiskowymi może stać się uciążliwe. W takim przypadku Symfony oferuje „lepszą” alternatywę do przechowywania sekretów.

### Przechowywanie sekretów

Zamiast używać wielu zmiennych środowiskowych, Symfony może zarządzać „skarbcem” (*vault*), w którym można przechowywać wiele sekretów. Jedną z kluczowych funkcji jest możliwość zapisania skarbca w repozytorium (ale bez klucza do jego otwarcia). Kolejną zaletą jest to, że można zarządzać oddzielnym skarbcem dla każdego środowiska.

Sekrety to tak naprawdę zamaskowane zmienne środowiskowe.

Dodaj klucz Akismet do skarbca:

```bash
symfony console secrets:set AKISMET_KEY
```

```
Please type the secret value:
>

[OK] Secret "AKISMET_KEY" encrypted in "config/secrets/dev/"; you can commit it.
```

Ponieważ uruchamiamy to polecenie po raz pierwszy, Symfony wygenerowało dwa klucze w katalogu `config/secret/dev/`. Następnie zapisało sekret `AKISMET_KEY` w tym samym katalogu.

Dla sekretów wykorzystywanych w środowisku developerskim możesz zdecydować się na zapisanie (commit) skarbca oraz wygenerowanych kluczy w katalogu `config/secret/dev/`.

Sekrety mogą być również nadpisane przez ustawienie zmiennej środowiskowej o tej samej nazwie.

### Sprawdzanie komentarzy pod kątem spamu

Prostym sposobem na wykrycie spamu podczas dodawania nowego komentarza jest wywołanie klasy sprawdzającej spam przed zapisaniem danych do bazy:

```diff
--- a/src/Controller/ConferenceController.php
+++ b/src/Controller/ConferenceController.php
@@ -7,6 +7,7 @@ use App\Entity\Conference;
 use App\Form\CommentType;
 use App\Repository\CommentRepository;
 use App\Repository\ConferenceRepository;
+use App\SpamChecker;
 use Doctrine\ORM\EntityManagerInterface;
 use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
 use Symfony\Component\DependencyInjection\Attribute\Autowire;
@@ -34,6 +35,7 @@ class ConferenceController extends AbstractController
         Request $request,
         Conference $conference,
         CommentRepository $commentRepository,
+        SpamChecker $spamChecker,
         #[Autowire('%photo_dir%')] string $photoDir,
     ): Response {
         $comment = new Comment();
@@ -48,6 +50,17 @@ class ConferenceController extends AbstractController
             }

             $this->entityManager->persist($comment);
+
+            $context = [
+                'user_ip' => $request->getClientIp(),
+                'user_agent' => $request->headers->get('user-agent'),
+                'referrer' => $request->headers->get('referer'),
+                'permalink' => $request->getUri(),
+            ];
+            if (2 === $spamChecker->getSpamScore($comment, $context)) {
+                throw new \RuntimeException('Blatant spam, go away!');
+            }
+
             $this->entityManager->flush();

             return $this->redirectToRoute('conference', ['slug' => $conference->getSlug()]);
```

Sprawdź, czy wszystko działa poprawnie.

### Zarządzanie sekretami w środowisku produkcyjnym

W środowisku produkcyjnym Platform.sh umożliwia ustawianie *wrażliwych zmiennych środowiskowych*:

```bash
symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:AKISMET_KEY --value=abcdef
```

Jednak – jak wspomniano wcześniej – korzystanie z *sekretów Symfony* może być lepszym rozwiązaniem. Nie ze względu na większe bezpieczeństwo, ale na łatwiejsze zarządzanie nimi w zespole projektowym. Wszystkie sekrety są przechowywane w repozytorium, a jedyną zmienną środowiskową, którą trzeba zarządzać w środowisku produkcyjnym, jest klucz deszyfrujący. Dzięki temu każdy członek zespołu może dodawać sekrety produkcyjne, nawet jeśli nie ma dostępu do serwerów produkcyjnych. Konfiguracja jest jednak nieco bardziej złożona.

Najpierw wygeneruj parę kluczy do użytku produkcyjnego:

```bash
symfony console secrets:generate-keys --env=prod
```

> [!NOTE]
> Na Linuksie i podobnych systemach użyj zamiast `--env=prod` zmiennej środowiskowej `APP_RUNTIME_ENV=prod`, aby uniknąć kompilowania aplikacji w trybie produkcyjnym:
> ```bash
> APP_RUNTIME_ENV=prod symfony console secrets:generate-keys
> ```

Następnie ponownie dodaj sekret Akismet do skarbca produkcyjnego, tym razem z jego produkcyjną wartością:

```bash
symfony console secrets:set AKISMET_KEY --env=prod
```

Ostatnim krokiem jest przesłanie klucza deszyfrującego do Platform.sh, ustawiając go jako wrażliwą zmienną środowiskową:

```bash
symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`
```

Możesz dodać i zatwierdzić wszystkie pliki – klucz deszyfrujący został automatycznie dodany do `.gitignore`, więc nigdy nie zostanie przypadkowo zapisany w repozytorium. Dla większego bezpieczeństwa możesz usunąć go ze swojego lokalnego komputera, ponieważ został już wdrożony.

```bash
rm -f config/secrets/prod/prod.decrypt.private.php
```

### Sprawdź również:
- [Dokumentacja komponentu HttpClient](https://symfony.com/doc/current/components/http_client.html);
- [Procesory zmiennych środowiskowych](https://symfony.com/doc/current/configuration/env_var_processors.html);
- [Ściąga do klienta HTTP Symfony](https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/httpclient_en_43.pdf) (Symfony HttpClient Cheat Sheet).

---

- **Poprzednia strona:** [Zabezpieczanie panelu administracyjnego](15-security.md)
- **Następna strona:** [Testowanie](17-tests.md)
