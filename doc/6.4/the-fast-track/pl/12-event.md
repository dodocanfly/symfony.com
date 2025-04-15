## Nasłuchiwanie zdarzeń

Obecny układ strony nie zawiera nagłówka nawigacyjnego, który umożliwiałby powrót do strony głównej lub przełączanie się między konferencjami.

### Dodanie nagłówka strony internetowej

Wszystko, co powinno być wyświetlane na wszystkich stronach internetowych, jak np. nagłówek, powinno być częścią głównego, bazowego układu:

```diff
--- a/templates/base.html.twig
+++ b/templates/base.html.twig
@@ -12,6 +12,15 @@
         {% endblock %}
     </head>
     <body>
+        <header>
+            <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
+            <ul>
+            {% for conference in conferences %}
+                <li><a href="{{ path('conference', { id: conference.id }) }}">{{ conference }}</a></li>
+            {% endfor %}
+            </ul>
+            <hr />
+        </header>
         {% block body %}{% endblock %}
     </body>
 </html>
```

Dodanie tego kodu do układu oznacza, że wszystkie szablony, które go rozszerzają, muszą definiować zmienną `conferences`, która musi zostać utworzona i przekazana z kontrolera.

Ponieważ mamy tylko dwa kontrolery, możesz zrobić coś takiego (nie wprowadzaj tej zmiany do swojego kodu — wkrótce nauczymy się lepszego sposobu):

```diff
--- a/src/Controller/ConferenceController.php
+++ b/src/Controller/ConferenceController.php
@@ -21,12 +21,13 @@ class ConferenceController extends AbstractController
     }

     #[Route('/conference/{id}', name: 'conference')]
-    public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
+    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, ConferenceRepository $conferenceRepository): Response
     {
         $offset = max(0, $request->query->getInt('offset', 0));
         $paginator = $commentRepository->getCommentPaginator($conference, $offset);

         return $this->render('conference/show.html.twig', [
+            'conferences' => $conferenceRepository->findAll(),
             'conference' => $conference,
             'comments' => $paginator,
             'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,
```

Wyobraź sobie konieczność aktualizowania dziesiątek kontrolerów. I robienia tego samego dla każdego nowego. To niezbyt praktyczne. Musi istnieć lepsze rozwiązanie.

Twig oferuje pojęcie zmiennych globalnych. *Zmienna globalna* jest dostępna we wszystkich renderowanych szablonach. Można je zdefiniować w pliku konfiguracyjnym, ale działa to tylko dla wartości statycznych. Aby dodać wszystkie konferencje jako globalną zmienną Twig, utworzymy **nasłuchiwacz zdarzeń (listener)**.

### Odkrywanie zdarzeń w Symfony

Symfony ma wbudowany komponent Event Dispatcher. Dispatcher (dyspozytor) *wywołuje* określone zdarzenia w konkretnych momentach, na które mogą nasłuchiwać tzw. *listenery* (nasłuchiwacze). Listenery to punkty zaczepienia w wewnętrznych mechanizmach frameworka.

Na przykład niektóre zdarzenia pozwalają na ingerencję w cykl życia żądania HTTP. Podczas obsługi żądania dispatcher wywołuje zdarzenia w momencie jego utworzenia, tuż przed wykonaniem kontrolera, gdy odpowiedź jest gotowa do wysłania, lub gdy zostanie rzucony wyjątek. *Listener* może nasłuchiwać jednego lub wielu zdarzeń i wykonywać odpowiednią logikę w zależności od kontekstu zdarzenia.

Zdarzenia to dobrze zdefiniowane punkty rozszerzalności, które czynią framework bardziej elastycznym i uniwersalnym. Wiele komponentów Symfony, takich jak Security, Messenger, Workflow czy Mailer, szeroko z nich korzysta.

Innym przykładem zastosowania zdarzeń i nasłuchiwaczy jest **cykl życia komendy** — możesz utworzyć listener, który wykona kod przed uruchomieniem jakiejkolwiek komendy.

Każda paczka (package) lub pakiet (bundle) może również wywoływać własne zdarzenia, aby umożliwić rozszerzanie swojego działania.

Aby uniknąć konieczności tworzenia pliku konfiguracyjnego z listą zdarzeń, na które listener ma nasłuchiwać, możesz stworzyć *subskrybenta (subscriber)*. Subscriber to listener posiadający statyczną metodę `getSubscribedEvents()`, która zwraca swoją konfigurację. Dzięki temu Symfony automatycznie rejestruje subskrybentów w dispatcherze.

### Implementacja subskrybenta

Znasz już dobrze tę melodię — użyj pakietu `maker-bundle`, aby wygenerować subskrybenta:

```bash
symfony console make:subscriber TwigEventSubscriber
```

Polecenie zapyta, którego zdarzenia chcesz nasłuchiwać. Wybierz zdarzenie `Symfony\Component\HttpKernel\Event\ControllerEvent`, które jest wywoływane tuż przed uruchomieniem kontrolera. To najlepszy moment, aby wstrzyknąć globalną zmienną `conferences`, tak aby Twig miał do niej dostęp podczas renderowania szablonu przez kontroler. Zaktualizuj swojego subskrybenta w następujący sposób:

```diff
--- a/src/EventSubscriber/TwigEventSubscriber.php
+++ b/src/EventSubscriber/TwigEventSubscriber.php
@@ -2,14 +2,25 @@

 namespace App\EventSubscriber;

+use App\Repository\ConferenceRepository;
 use Symfony\Component\EventDispatcher\EventSubscriberInterface;
 use Symfony\Component\HttpKernel\Event\ControllerEvent;
+use Twig\Environment;

 class TwigEventSubscriber implements EventSubscriberInterface
 {
+    private $twig;
+    private $conferenceRepository;
+
+    public function __construct(Environment $twig, ConferenceRepository $conferenceRepository)
+    {
+        $this->twig = $twig;
+        $this->conferenceRepository = $conferenceRepository;
+    }
+
     public function onControllerEvent(ControllerEvent $event): void
     {
-        // ...
+        $this->twig->addGlobal('conferences', $this->conferenceRepository->findAll());
     }

     public static function getSubscribedEvents(): array
```

Od teraz możesz dodawać dowolną liczbę kontrolerów — zmienna `conferences` zawsze będzie dostępna w Twig-u.

> [!NOTE]
> W późniejszym kroku omówimy dużo lepszą alternatywę pod względem wydajności.

### Sortowanie konferencji według roku i miasta

Uporządkowanie listy konferencji według roku może ułatwić przeglądanie. Moglibyśmy stworzyć własną metodę do pobierania i sortowania wszystkich konferencji, ale zamiast tego nadpiszemy domyślną implementację metody `findAll()`, aby mieć pewność, że sortowanie będzie stosowane wszędzie:

```diff
--- a/src/Repository/ConferenceRepository.php
+++ b/src/Repository/ConferenceRepository.php
@@ -21,6 +21,11 @@ class ConferenceRepository extends ServiceEntityRepository
         parent::__construct($registry, Conference::class);
     }

+    public function findAll(): array
+    {
+        return $this->findBy([], ['year' => 'ASC', 'city' => 'ASC']);
+    }
+
     //    /**
     //     * @return Conference[] Returns an array of Conference objects
     //     */
```

Po wykonaniu tego kroku strona powinna wyglądać następująco:

![header](https://symfony.com/doc/6.4en//the-fast-track/_images/header.png)

### Sprawdź również:
- [Przepływ żądania i odpowiedzi](https://symfony.com/doc/current/components/http_kernel.html#the-workflow-of-a-request) w aplikacjach Symfony;
- [Wbudowane zdarzenia HTTP w Symfony](https://symfony.com/doc/current/reference/events.html);
- [Wbudowane zdarzenia konsoli w Symfony](https://symfony.com/doc/current/components/console/events.html).

---

- **Poprzednia strona:** [Rozgałęzianie kodu](11-branch.md)
- **Następna strona:** [Zarządzanie cyklem życia obiektów Doctrine](13-lifecycle.md)
