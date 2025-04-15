## Budowanie interfejsu użytkownika

Wszystko jest już gotowe, aby stworzyć pierwszą wersję interfejsu użytkownika strony internetowej. Nie będziemy się teraz skupiać na wyglądzie — na razie ma po prostu działać.

Pamiętasz, jak musieliśmy stosować mechanizmy ucieczki znaków w kontrolerze dla wielkanocnego jajka, aby uniknąć problemów z bezpieczeństwem? Właśnie dlatego nie będziemy używać PHP do szablonów. Zamiast tego skorzystamy z Twig. Oprócz tego, że [Twig](https://twig.symfony.com) automatycznie zajmuje się ucieczką znaków w wyjściu, oferuje też wiele przydatnych funkcji, z których skorzystamy — jak na przykład dziedziczenie szablonów.

### Użycie Twig do szablonów

Wszystkie strony serwisu będą korzystać z tego samego układu. Podczas instalacji Twig automatycznie utworzony został katalog `templates/` oraz przykładowy szablon układu w pliku `base.html.twig`.

**templates/base.html.twig**
```twig
<!DOCTYPE html>
<html>
    <head>
        <meta charset="UTF-8">
        <title>{% block title %}Welcome!{% endblock %}</title>
        <link rel="icon" href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 128 128%22><text y=%221.2em%22 font-size=%2296%22>⚫️</text></svg>">
        {# Run `composer require symfony/webpack-encore-bundle` to start using Symfony UX #}
        {% block stylesheets %}
            {{ encore_entry_link_tags('app') }}
        {% endblock %}

        {% block javascripts %}
            {{ encore_entry_script_tags('app') }}
        {% endblock %}
    </head>
    <body>
        {% block body %}{% endblock %}
    </body>
</html>
```

Układ (layout) może definiować bloki (block elements), czyli miejsca, w których szablony potomne rozszerzające ten układ mogą dodawać swoją zawartość.

Utwórzmy teraz szablon dla strony głównej projektu w pliku `templates/conference/index.html.twig`:

**templates/conference/index.html.twig**
```twig
{% extends 'base.html.twig' %}

{% block title %}Conference Guestbook{% endblock %}

{% block body %}
    <h2>Give your feedback!</h2>

    {% for conference in conferences %}
        <h4>{{ conference }}</h4>
    {% endfor %}
{% endblock %}
```

Szablon rozszerza `base.html.twig` i nadpisuje bloki `title` oraz `body`.

Notacja `{% %}` w szablonie służy do określania *działań* i *struktury*.

Notacja `{{ }}` jest używana do wyświetlania danych. `{{ conference }}` wyświetla reprezentację obiektu konferencji (czyli wynik działania metody `__toString` na obiekcie `Conference`).

### Użycie Twig w kontrolerze

Zaktualizuj kontroler, aby renderował szablon Twig:

```diff
--- a/src/Controller/ConferenceController.php
+++ b/src/Controller/ConferenceController.php
@@ -2,22 +2,19 @@

 namespace App\Controller;

+use App\Repository\ConferenceRepository;
 use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
 use Symfony\Component\HttpFoundation\Response;
 use Symfony\Component\Routing\Attribute\Route;
+use Twig\Environment;

 class ConferenceController extends AbstractController
 {
     #[Route('/', name: 'homepage')]
-    public function index(): Response
+    public function index(Environment $twig, ConferenceRepository $conferenceRepository): Response
     {
-        return new Response(<<<EOF
-            <html>
-                <body>
-                    <img src="/images/under-construction.gif" />
-                </body>
-            </html>
-            EOF
-        );
+        return new Response($twig->render('conference/index.html.twig', [
+            'conferences' => $conferenceRepository->findAll(),
+        ]));
     }
 }
```

Dzieje się tutaj całkiem sporo.

Aby móc renderować szablon, potrzebujemy obiektu Twig `Environment` (głównego punktu wejścia do pracy z Twig). Zwróć uwagę, że prosimy o instancję Twig, stosując podpowiedź typu (type hinting) w metodzie kontrolera. Symfony jest na tyle inteligentne, że wie, jak wstrzyknąć odpowiedni obiekt.

Potrzebujemy również repozytorium konferencji, aby pobrać wszystkie konferencje z bazy danych.

W kodzie kontrolera metoda `render()` renderuje szablon i przekazuje do niego tablicę zmiennych. Przekazujemy listę obiektów `Conference` jako zmienną `conferences`.

Kontroler to zwykła klasa PHP. Nie musimy nawet rozszerzać klasy `AbstractController`, jeśli chcemy jawnie określać nasze zależności. Można ją usunąć (ale nie rób tego, ponieważ w kolejnych krokach skorzystamy z przydatnych skrótów, które ona oferuje).

### Tworzenie strony dla konferencji

Każda konferencja powinna mieć osobną stronę, na której będą wyświetlane jej komentarze. Dodanie nowej strony polega na dodaniu kontrolera, zdefiniowaniu dla niego trasy (routingu) oraz stworzeniu odpowiedniego szablonu.

Dodaj metodę `show()` w pliku `src/Controller/ConferenceController.php`:

```diff
--- a/src/Controller/ConferenceController.php
+++ b/src/Controller/ConferenceController.php
@@ -2,6 +2,8 @@

 namespace App\Controller;

+use App\Entity\Conference;
+use App\Repository\CommentRepository;
 use App\Repository\ConferenceRepository;
 use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
 use Symfony\Component\HttpFoundation\Response;
@@ -17,4 +19,13 @@ class ConferenceController extends AbstractController
             'conferences' => $conferenceRepository->findAll(),
         ]));
     }
+
+    #[Route('/conference/{id}', name: 'conference')]
+    public function show(Environment $twig, Conference $conference, CommentRepository $commentRepository): Response
+    {
+        return new Response($twig->render('conference/show.html.twig', [
+            'conference' => $conference,
+            'comments' => $commentRepository->findBy(['conference' => $conference], ['createdAt' => 'DESC']),
+        ]));
+    }
 }
```

Ta metoda ma specjalne zachowanie, którego jeszcze nie widzieliśmy. Prosimy o wstrzyknięcie instancji `Conference` do metody. Jednak w bazie danych może znajdować się wiele takich obiektów. Symfony potrafi określić, który dokładnie obiekt chcemy, na podstawie `{id}` przekazanego w ścieżce żądania (gdzie `id` to klucz główny tabeli `conference` w bazie danych).

Pobranie komentarzy powiązanych z daną konferencją można wykonać za pomocą metody `findBy()`, która jako pierwszy argument przyjmuje kryterium wyszukiwania.

Ostatnim krokiem jest utworzenie pliku `templates/conference/show.html.twig`:

**templates/conference/show.html.twig**
```twig
{% extends 'base.html.twig' %}

{% block title %}Conference Guestbook - {{ conference }}{% endblock %}

{% block body %}
    <h2>{{ conference }} Conference</h2>

    {% if comments|length > 0 %}
        {% for comment in comments %}
            {% if comment.photofilename %}
                <img src="{{ asset('uploads/photos/' ~ comment.photofilename) }}" style="max-width: 200px" />
            {% endif %}

            <h4>{{ comment.author }}</h4>
            <small>
                {{ comment.createdAt|format_datetime('medium', 'short') }}
            </small>

            <p>{{ comment.text }}</p>
        {% endfor %}
    {% else %}
        <div>No comments have been posted yet for this conference.</div>
    {% endif %}
{% endblock %}
```

W tym szablonie używamy notacji `|`, aby wywołać *filtry* Twig. Filtr przekształca wartość. `comments|length` zwraca liczbę komentarzy, a `comment.createdAt|format_datetime('medium', 'short')` formatuje datę w czytelnym dla użytkownika formacie.

Spróbuj przejść do „pierwszej” konferencji, używając adresu `/conference/1`, i zwróć uwagę na następujący błąd:

![intl-twig-error](https://symfony.com/doc/6.4en//the-fast-track/_images/intl-twig-error.png)

Błąd pochodzi od filtra `format_datetime`, ponieważ nie jest on częścią rdzenia Twig. Wiadomość o błędzie podpowiada, jaki pakiet należy zainstalować, aby rozwiązać problem:

```bash
symfony composer req "twig/intl-extra:^3"
```

Po zainstalowaniu tego pakietu strona działa poprawnie.

### Łączenie stron ze sobą

Ostatnim krokiem do ukończenia pierwszej wersji interfejsu użytkownika jest dodanie linków do stron konferencji ze strony głównej:

```diff
--- a/templates/conference/index.html.twig
+++ b/templates/conference/index.html.twig
@@ -7,5 +7,8 @@

     {% for conference in conferences %}
         <h4>{{ conference }}</h4>
+        <p>
+            <a href="/conference/{{ conference.id }}">View</a>
+        </p>
     {% endfor %}
 {% endblock %}
```

Jednak ręczne wpisywanie ścieżki (hard-coding) to zły pomysł z kilku powodów. Najważniejszy z nich to fakt, że jeśli zmienisz ścieżkę (np. z `/conference/{id}` na `/conferences/{id}`), wszystkie linki trzeba będzie ręcznie zaktualizować.

Zamiast tego użyj *funkcji* `path()` w Twig i podaj *nazwę trasy* (route):

```diff
--- a/templates/conference/index.html.twig
+++ b/templates/conference/index.html.twig
@@ -8,7 +8,7 @@
     {% for conference in conferences %}
         <h4>{{ conference }}</h4>
         <p>
-            <a href="/conference/{{ conference.id }}">View</a>
+            <a href="{{ path('conference', { id: conference.id }) }}">View</a>
         </p>
     {% endfor %}
 {% endblock %}
```

Funkcja `path()` generuje ścieżkę do strony na podstawie jej nazwy trasy. Wartości parametrów trasy przekazywane są jako mapa Twig.

### Paginacja komentarzy

Przy tysiącach uczestników, możemy spodziewać się sporej liczby komentarzy. Jeśli wyświetlimy je wszystkie na jednej stronie, ta może bardzo szybko urosnąć.

Utwórz metodę `getCommentPaginator()` w repozytorium komentarzy, która zwróci *paginator* komentarzy na podstawie konferencji i offsetu (gdzie zacząć):

```diff
--- a/src/Repository/CommentRepository.php
+++ b/src/Repository/CommentRepository.php
@@ -3,8 +3,10 @@
 namespace App\Repository;

 use App\Entity\Comment;
+use App\Entity\Conference;
 use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
 use Doctrine\Persistence\ManagerRegistry;
+use Doctrine\ORM\Tools\Pagination\Paginator;

 /**
  * @extends ServiceEntityRepository<Comment>
@@ -16,11 +18,27 @@ use Doctrine\Persistence\ManagerRegistry;
  */
 class CommentRepository extends ServiceEntityRepository
 {
+    public const COMMENTS_PER_PAGE = 2;
+
     public function __construct(ManagerRegistry $registry)
     {
         parent::__construct($registry, Comment::class);
     }

+    public function getCommentPaginator(Conference $conference, int $offset): Paginator
+    {
+        $query = $this->createQueryBuilder('c')
+            ->andWhere('c.conference = :conference')
+            ->setParameter('conference', $conference)
+            ->orderBy('c.createdAt', 'DESC')
+            ->setMaxResults(self::COMMENTS_PER_PAGE)
+            ->setFirstResult($offset)
+            ->getQuery()
+        ;
+
+        return new Paginator($query);
+    }
+
     //    /**
     //     * @return Comment[] Returns an array of Comment objects
     //     */
```

Ustawiliśmy maksymalną liczbę komentarzy na stronę na 2, aby ułatwić testowanie.

Aby zarządzać paginacją w szablonie, przekaż paginator Doctrine zamiast kolekcji Doctrine do Twig:

```diff
--- a/src/Controller/ConferenceController.php
+++ b/src/Controller/ConferenceController.php
@@ -6,6 +6,7 @@ use App\Entity\Conference;
 use App\Repository\CommentRepository;
 use App\Repository\ConferenceRepository;
 use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
+use Symfony\Component\HttpFoundation\Request;
 use Symfony\Component\HttpFoundation\Response;
 use Symfony\Component\Routing\Attribute\Route;
 use Twig\Environment;
@@ -21,11 +22,16 @@ class ConferenceController extends AbstractController
     }

     #[Route('/conference/{id}', name: 'conference')]
-    public function show(Environment $twig, Conference $conference, CommentRepository $commentRepository): Response
+    public function show(Request $request, Environment $twig, Conference $conference, CommentRepository $commentRepository): Response
     {
+        $offset = max(0, $request->query->getInt('offset', 0));
+        $paginator = $commentRepository->getCommentPaginator($conference, $offset);
+
         return new Response($twig->render('conference/show.html.twig', [
             'conference' => $conference,
-            'comments' => $commentRepository->findBy(['conference' => $conference], ['createdAt' => 'DESC']),
+            'comments' => $paginator,
+            'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,
+            'next' => min(count($paginator), $offset + CommentRepository::COMMENTS_PER_PAGE),
         ]));
     }
 }
```

Kontroler pobiera `offset` z łańcucha zapytania Request (`$request->query`) jako liczbę całkowitą (metodą `getInt()`), domyślnie ustawiając ją na 0, jeśli nie jest dostępna.

Offsety `previous` i `next` są obliczane na podstawie wszystkich informacji, które mamy z paginatora.

Na koniec, zaktualizuj szablon, aby dodać linki do poprzednich i następnych stron:

```diff
--- a/templates/conference/show.html.twig
+++ b/templates/conference/show.html.twig
@@ -6,6 +6,8 @@
     <h2>{{ conference }} Conference</h2>

     {% if comments|length > 0 %}
+        <div>There are {{ comments|length }} comments.</div>
+
         {% for comment in comments %}
             {% if comment.photofilename %}
                 <img src="{{ asset('uploads/photos/' ~ comment.photofilename) }}" style="max-width: 200px" />
@@ -18,6 +20,13 @@

             <p>{{ comment.text }}</p>
         {% endfor %}
+
+        {% if previous >= 0 %}
+            <a href="{{ path('conference', { id: conference.id, offset: previous }) }}">Previous</a>
+        {% endif %}
+        {% if next < comments|length %}
+            <a href="{{ path('conference', { id: conference.id, offset: next }) }}">Next</a>
+        {% endif %}
     {% else %}
         <div>No comments have been posted yet for this conference.</div>
     {% endif %}
```

Powinieneś teraz móc nawigować po komentarzach za pomocą linków "Previous" i "Next":

![pagination-next](https://symfony.com/doc/6.4en//the-fast-track/_images/pagination-next.png)

![pagination-previous](https://symfony.com/doc/6.4en//the-fast-track/_images/pagination-previous.png)

### Refaktoryzacja kontrolera

Być może zauważyłeś, że obie metody w `ConferenceController` przyjmują środowisko Twig jako argument. Zamiast wstrzykiwać je do każdej metody, skorzystajmy z metody pomocniczej `render()`, dostarczonej przez klasę nadrzędną:

```diff
--- a/src/Controller/ConferenceController.php
+++ b/src/Controller/ConferenceController.php
@@ -9,29 +9,28 @@ use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
 use Symfony\Component\HttpFoundation\Request;
 use Symfony\Component\HttpFoundation\Response;
 use Symfony\Component\Routing\Attribute\Route;
-use Twig\Environment;

 class ConferenceController extends AbstractController
 {
     #[Route('/', name: 'homepage')]
-    public function index(Environment $twig, ConferenceRepository $conferenceRepository): Response
+    public function index(ConferenceRepository $conferenceRepository): Response
     {
-        return new Response($twig->render('conference/index.html.twig', [
+        return $this->render('conference/index.html.twig', [
             'conferences' => $conferenceRepository->findAll(),
-        ]));
+        ]);
     }

     #[Route('/conference/{id}', name: 'conference')]
-    public function show(Request $request, Environment $twig, Conference $conference, CommentRepository $commentRepository): Response
+    public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
     {
         $offset = max(0, $request->query->getInt('offset', 0));
         $paginator = $commentRepository->getCommentPaginator($conference, $offset);

-        return new Response($twig->render('conference/show.html.twig', [
+        return $this->render('conference/show.html.twig', [
             'conference' => $conference,
             'comments' => $paginator,
             'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,
             'next' => min(count($paginator), $offset + CommentRepository::COMMENTS_PER_PAGE),
-        ]));
+        ]);
     }
 }
```

### Sprawdź również:
- [Dokumentacja Twig](https://twig.symfony.com/doc/3.x);
- [Tworzenie i używanie szablonów](https://symfony.com/doc/current/templates.html) w aplikacjach Symfony;
- [Tutorial Twig na SymfonyCasts](https://symfonycasts.com/screencast/symfony/twig-recipe);
- [Funkcje i filtry Twig dostępne tylko w Symfony](https://symfony.com/doc/current/reference/twig_reference.html);
- [Kontroler bazowy `AbstractController`](https://symfony.com/doc/current/controller.html#the-base-controller-classes-services).

---

- **Poprzednia strona:** [Konfigurowanie zaplecza administratora](9-backend.md)
- **Następna strona:** [Rozgałęzianie kodu](11-branch.md)
