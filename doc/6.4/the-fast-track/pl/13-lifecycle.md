## Zarządzanie cyklem życia obiektów Doctrine

Podczas tworzenia nowego komentarza dobrze by było, gdyby data `createdAt` była automatycznie ustawiana na bieżącą datę i godzinę.

Doctrine oferuje różne sposoby manipulowania obiektami i ich właściwościami w trakcie ich cyklu życia (przed utworzeniem wiersza w bazie danych, po jego aktualizacji itp.).

### Definiowanie callbacków cyklu życia

Gdy dane zachowanie nie wymaga żadnych usług i powinno być stosowane tylko do jednego rodzaju encji, należy zdefiniować callback bezpośrednio w klasie encji:

```diff
--- a/src/Controller/Admin/CommentCrudController.php
+++ b/src/Controller/Admin/CommentCrudController.php
@@ -57,8 +57,6 @@ class CommentCrudController extends AbstractCrudController
         ]);
         if (Crud::PAGE_EDIT === $pageName) {
             yield $createdAt->setFormTypeOption('disabled', true);
-        } else {
-            yield $createdAt;
         }
     }
 }
--- a/src/Entity/Comment.php
+++ b/src/Entity/Comment.php
@@ -7,6 +7,7 @@ use Doctrine\DBAL\Types\Types;
 use Doctrine\ORM\Mapping as ORM;

 #[ORM\Entity(repositoryClass: CommentRepository::class)]
+#[ORM\HasLifecycleCallbacks]
 class Comment
 {
     #[ORM\Id]
@@ -86,6 +87,12 @@ class Comment
         return $this;
     }

+    #[ORM\PrePersist]
+    public function setCreatedAtValue()
+    {
+        $this->createdAt = new \DateTimeImmutable();
+    }
+
     public function getConference(): ?Conference
     {
         return $this->conference;
```

*Zdarzenie* `ORM\PrePersist` jest wywoływane, gdy obiekt jest zapisywany w bazie danych po raz pierwszy. W tym momencie wywoływana jest metoda `setCreatedAtValue()`, a bieżąca data i godzina zostaje użyta jako wartość właściwości `createdAt`.

### Dodawanie slugów do konferencji

Adresy URL konferencji nie są zbyt czytelne: `/conference/1`. Co ważniejsze, zależą od szczegółów implementacyjnych (ujawniany jest klucz główny z bazy danych).

A co powiesz na używanie adresów URL takich jak `/conference/paris-2020`? Wygląda to znacznie lepiej. `paris-2020` to tak zwany *slug* konferencji.

Dodaj nową właściwość `slug` do encji konferencji (nienullowalny string o długości 255 znaków):

```bash
symfony console make:entity Conference
```

Utwórz plik migracji, aby dodać nową kolumnę:

```bash
symfony console make:migration
```

I wykonaj tę nową migrację:

```bash
symfony console doctrine:migrations:migrate
```

Pojawił się błąd? To spodziewane. Dlaczego? Ponieważ określiliśmy, że slug nie może być `NULL`, ale istniejące rekordy w tabeli `conference` otrzymają wartość `NULL` podczas wykonywania migracji. Naprawmy to, modyfikując migrację:

```diff
--- a/migrations/Version00000000000000.php
+++ b/migrations/Version00000000000000.php
@@ -20,7 +20,9 @@ final class Version00000000000000 extends AbstractMigration
     public function up(Schema $schema): void
     {
         // this up() migration is auto-generated, please modify it to your needs
-        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255) NOT NULL');
+        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255)');
+        $this->addSql("UPDATE conference SET slug=CONCAT(LOWER(city), '-', year)");
+        $this->addSql('ALTER TABLE conference ALTER COLUMN slug SET NOT NULL');
     }

     public function down(Schema $schema): void
```

Sztuczka polega na tym, aby najpierw dodać kolumnę jako dopuszczającą `NULL`, następnie ustawić wartość `slug` na nienullowalną, a na końcu zmienić kolumnę tak, by nie akceptowała `NULL`.

> [!NOTE]
> W prawdziwym projekcie użycie `CONCAT(LOWER(city), '-', year)` może nie wystarczyć. W takim przypadku należałoby skorzystać z „prawdziwego” *Sluggera*.

Migracja powinna teraz przebiec bez problemów:

```bash
symfony console doctrine:migrations:migrate
```

Ponieważ aplikacja wkrótce będzie wykorzystywać slugi do odnajdywania konferencji, zmodyfikujmy encję `Conference`, aby upewnić się, że wartości `slug` są unikalne w bazie danych:

```diff
--- a/src/Entity/Conference.php
+++ b/src/Entity/Conference.php
@@ -6,8 +6,10 @@ use App\Repository\ConferenceRepository;
 use Doctrine\Common\Collections\ArrayCollection;
 use Doctrine\Common\Collections\Collection;
 use Doctrine\ORM\Mapping as ORM;
+use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;

 #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
+#[UniqueEntity('slug')]
 class Conference
 {
     #[ORM\Id]
@@ -30,7 +32,7 @@ class Conference
     #[ORM\OneToMany(targetEntity: Comment::class, mappedBy: 'conference', orphanRemoval: true)]
     private Collection $comments;

-    #[ORM\Column(length: 255)]
+    #[ORM\Column(length: 255, unique: true)]
     private ?string $slug = null;

     public function __construct()
```

Jak zapewne się domyślasz, ponownie musimy wykonać „taniec migracyjny”:

```bash
symfony console make:migration
symfony console doctrine:migrations:migrate
```

### Generowanie slugów

Generowanie sluga, który dobrze wygląda w adresie URL (gdzie wszystkie znaki inne niż ASCII muszą być zakodowane), to trudne zadanie – szczególnie dla języków innych niż angielski. Jak na przykład zamienić `é` na `e`?

Zamiast wymyślać koło na nowo, skorzystajmy z komponentu Symfony `String`, który ułatwia manipulację tekstem i udostępnia narzędzie do tworzenia *slugów*.

Dodaj metodę `computeSlug()` do klasy `Conference`, która będzie generować slug na podstawie danych konferencji:

```diff
--- a/src/Entity/Conference.php
+++ b/src/Entity/Conference.php
@@ -7,6 +7,7 @@ use Doctrine\Common\Collections\ArrayCollection;
 use Doctrine\Common\Collections\Collection;
 use Doctrine\ORM\Mapping as ORM;
 use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
+use Symfony\Component\String\Slugger\SluggerInterface;

 #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
 #[UniqueEntity('slug')]
@@ -50,6 +51,13 @@ class Conference
         return $this->id;
     }

+    public function computeSlug(SluggerInterface $slugger)
+    {
+        if (!$this->slug || '-' === $this->slug) {
+            $this->slug = (string) $slugger->slug((string) $this)->lower();
+        }
+    }
+
     public function getCity(): ?string
     {
         return $this->city;
```

Metoda `computeSlug()` generuje slug tylko wtedy, gdy bieżący slug jest pusty lub ustawiony na specjalną wartość `-`. Dlaczego potrzebujemy tej specjalnej wartości `-`? Ponieważ podczas dodawania konferencji w panelu administracyjnym slug jest wymagany. Potrzebujemy więc niepustej wartości, która jednocześnie sygnalizuje aplikacji, że slug powinien zostać wygenerowany automatycznie.

### Definiowanie złożonego callbacka cyklu życia

Podobnie jak właściwość `createdAt`, pole `slug` powinno być ustawiane automatycznie za każdym razem, gdy konferencja jest aktualizowana — poprzez wywołanie metody `computeSlug()`.

Jednakże, ponieważ metoda ta zależy od implementacji `SluggerInterface`, nie możemy po prostu dodać zdarzenia `prePersist` jak wcześniej (nie mamy sposobu, aby wstrzyknąć sluggera).

Zamiast tego utwórzmy nasłuchiwacz (listener) encji Doctrine:

**src/EntityListener/ConferenceEntityListener.php**
```php
namespace App\EntityListener;

use App\Entity\Conference;
use Doctrine\Persistence\Event\LifecycleEventArgs;
use Symfony\Component\String\Slugger\SluggerInterface;

class ConferenceEntityListener
{
    public function __construct(
        private SluggerInterface $slugger,
    ) {
    }

    public function prePersist(Conference $conference, LifecycleEventArgs $event)
    {
        $conference->computeSlug($this->slugger);
    }

    public function preUpdate(Conference $conference, LifecycleEventArgs $event)
    {
        $conference->computeSlug($this->slugger);
    }
}
```

Zwróć uwagę, że slug jest aktualizowany zarówno podczas tworzenia nowej konferencji (`prePersist()`), jak i przy każdej jej aktualizacji (`preUpdate()`).

### Konfigurowanie usługi w kontenerze

Do tej pory nie mówiliśmy o jednym z kluczowych komponentów Symfony – *kontenerze wstrzykiwania zależności* (*dependency injection container*). Kontener ten odpowiada za zarządzanie usługami: tworzy je i wstrzykuje tam, gdzie są potrzebne.

*Usługa* to „globalny” obiekt, który udostępnia funkcjonalność (np. mailer, logger, slugger itd.), w przeciwieństwie do *obiektów danych* (np. instancji encji Doctrine).

Rzadko kiedy trzeba bezpośrednio korzystać z kontenera — automatycznie wstrzykuje on obiekty usług wszędzie tam, gdzie są potrzebne: np. w kontrolerach, gdy używasz *type hintingu* w argumentach metod.

Jeśli zastanawiałeś się, w jaki sposób nasłuchiwacz zdarzeń został zarejestrowany w poprzednim kroku, to teraz masz odpowiedź: zrobił to kontener. Gdy klasa implementuje określone interfejsy, kontener „wie”, że trzeba ją zarejestrować w odpowiedni sposób.

W naszym przypadku, ponieważ klasa nie implementuje żadnego interfejsu ani nie rozszerza żadnej klasy bazowej, Symfony nie wie, jak ją automatycznie skonfigurować. Możemy jednak użyć atrybutu, by poinformować kontener Symfony, jak ją powiązać:

```diff
--- a/src/EntityListener/ConferenceEntityListener.php
+++ b/src/EntityListener/ConferenceEntityListener.php
@@ -3,9 +3,13 @@
 namespace App\EntityListener;

 use App\Entity\Conference;
+use Doctrine\Bundle\DoctrineBundle\Attribute\AsEntityListener;
+use Doctrine\ORM\Events;
 use Doctrine\Persistence\Event\LifecycleEventArgs;
 use Symfony\Component\String\Slugger\SluggerInterface;

+#[AsEntityListener(event: Events::prePersist, entity: Conference::class)]
+#[AsEntityListener(event: Events::preUpdate, entity: Conference::class)]
 class ConferenceEntityListener
 {
     public function __construct(
```

> [!NOTE]
> Nie myl nasłuchiwaczy zdarzeń Doctrine z tymi w Symfony. Mimo że wyglądają podobnie, pod spodem używają zupełnie innej infrastruktury.

### Używanie slugów w aplikacji

Spróbuj dodać więcej konferencji w panelu administracyjnym albo zmień miasto lub rok w już istniejącej — slug nie zostanie zaktualizowany, chyba że użyjesz specjalnej wartości `-`.

Ostatnią zmianą jest zaktualizowanie kontrolerów i szablonów tak, aby korzystały ze `sluga` konferencji zamiast jej `identyfikatora` w trasach:

```diff
--- a/src/Controller/ConferenceController.php
+++ b/src/Controller/ConferenceController.php
@@ -20,7 +20,7 @@ class ConferenceController extends AbstractController
         ]);
     }

-    #[Route('/conference/{id}', name: 'conference')]
+    #[Route('/conference/{slug}', name: 'conference')]
     public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
     {
         $offset = max(0, $request->query->getInt('offset', 0));
--- a/templates/base.html.twig
+++ b/templates/base.html.twig
@@ -16,7 +16,7 @@
             <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
             <ul>
             {% for conference in conferences %}
-                <li><a href="{{ path('conference', { id: conference.id }) }}">{{ conference }}</a></li>
+                <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
             {% endfor %}
             </ul>
             <hr />
--- a/templates/conference/index.html.twig
+++ b/templates/conference/index.html.twig
@@ -8,7 +8,7 @@
     {% for conference in conferences %}
         <h4>{{ conference }}</h4>
         <p>
-            <a href="{{ path('conference', { id: conference.id }) }}">View</a>
+            <a href="{{ path('conference', { slug: conference.slug }) }}">View</a>
         </p>
     {% endfor %}
 {% endblock %}
--- a/templates/conference/show.html.twig
+++ b/templates/conference/show.html.twig
@@ -22,10 +22,10 @@
         {% endfor %}

         {% if previous >= 0 %}
-            <a href="{{ path('conference', { id: conference.id, offset: previous }) }}">Previous</a>
+            <a href="{{ path('conference', { slug: conference.slug, offset: previous }) }}">Previous</a>
         {% endif %}
         {% if next < comments|length %}
-            <a href="{{ path('conference', { id: conference.id, offset: next }) }}">Next</a>
+            <a href="{{ path('conference', { slug: conference.slug, offset: next }) }}">Next</a>
         {% endif %}
     {% else %}
         <div>No comments have been posted yet for this conference.</div>
```

Dostęp do stron konferencji powinien teraz odbywać się za pomocą jej sluga:

![slug](https://symfony.com/doc/6.4en//the-fast-track/_images/slug.png)

### Sprawdź również:
- [System zdarzeń Doctrine](../../../7.2/doctrine/events.md) (callbacki cyklu życia i nasłuchiwacze, entity listeners oraz subskrybenci cyklu życia);
- [Dokumentacja komponentu String](../../../7.2/components/string.md);
- [Kontener usług](../../../7.2/service_container.md);
- [Ściąga z usług Symfony](https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/services_en_42.pdf) (Symfony Services Cheat Sheet).

---

- **Poprzednia strona:** [Nasłuchiwanie zdarzeń](12-event.md)
- **Następna strona:** [Akceptowanie opinii za pomocą formularzy](14-form.md)
