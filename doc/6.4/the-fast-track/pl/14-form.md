## Akceptowanie opinii za pomocą formularzy

Czas pozwolić uczestnikom konferencji wyrazić swoją opinię. Będą oni przekazywać swoje komentarze za pomocą formularza HTML.

### Tworzenie klasy formularza

Użyj pakietu Maker, aby wygenerować klasę formularza:

```bash
symfony console make:form CommentType Comment
```

```
created: src/Form/CommentType.php


 Success!


Next: Add fields to your form and start using it.
Find the documentation at https://symfony.com/doc/current/forms.html
```

Klasa `App\Form\CommentType` definiuje formularz dla encji `App\Entity\Comment`:

**src/Form/CommentType.php**
```php
namespace App\Form;

use App\Entity\Comment;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

class CommentType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options)
    {
        $builder
            ->add('author')
            ->add('text')
            ->add('email')
            ->add('createdAt')
            ->add('photoFilename')
            ->add('conference')
        ;
    }

    public function configureOptions(OptionsResolver $resolver)
    {
        $resolver->setDefaults([
            'data_class' => Comment::class,
        ]);
    }
}
```

[Typ formularza](https://symfony.com/doc/current/forms.html#form-types) opisuje *pola formularza* powiązane z modelem. Odpowiada za konwersję danych między przesłanymi wartościami a właściwościami klasy modelu. Domyślnie Symfony korzysta z metadanych encji `Comment` — takich jak metadane Doctrine — aby odgadnąć konfigurację dla każdego pola. Na przykład pole typu `text` renderowane jest jako `textarea`, ponieważ w bazie danych odpowiada mu kolumna o większym rozmiarze.

### Wyświetlanie formularza

Aby wyświetlić formularz użytkownikowi, utwórz go w kontrolerze i przekaż do szablonu:

```diff
--- a/src/Controller/ConferenceController.php
+++ b/src/Controller/ConferenceController.php
@@ -2,7 +2,9 @@

 namespace App\Controller;

+use App\Entity\Comment;
 use App\Entity\Conference;
+use App\Form\CommentType;
 use App\Repository\CommentRepository;
 use App\Repository\ConferenceRepository;
 use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
@@ -23,6 +25,9 @@ class ConferenceController extends AbstractController
     #[Route('/conference/{slug}', name: 'conference')]
     public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
     {
+        $comment = new Comment();
+        $form = $this->createForm(CommentType::class, $comment);
+
         $offset = max(0, $request->query->getInt('offset', 0));
         $paginator = $commentRepository->getCommentPaginator($conference, $offset);

@@ -31,6 +36,7 @@ class ConferenceController extends AbstractController
             'comments' => $paginator,
             'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,
             'next' => min(count($paginator), $offset + CommentRepository::COMMENTS_PER_PAGE),
+            'comment_form' => $form,
         ]);
     }
 }
```

Nigdy nie powinno się bezpośrednio instancjonować typu formularza. Zamiast tego należy używać metody `createForm()`. Metoda ta jest częścią `AbstractController` i ułatwia tworzenie formularzy.

Wyświetlanie formularza w szablonie można zrealizować przy pomocy funkcji Twig `form`:

```diff
--- a/templates/conference/show.html.twig
+++ b/templates/conference/show.html.twig
@@ -30,4 +30,8 @@
     {% else %}
         <div>No comments have been posted yet for this conference.</div>
     {% endif %}
+
+    <h2>Add your own feedback</h2>
+
+    {{ form(comment_form) }}
 {% endblock %}
```

Po odświeżeniu strony konferencji w przeglądarce, zauważysz, że każde pole formularza wyświetla odpowiedni widget HTML (typ danych pochodzi z modelu):

![form](https://symfony.com/doc/6.4en//the-fast-track/_images/form.png)

Funkcja `form()` generuje kod HTML formularza na podstawie wszystkich informacji zdefiniowanych w klasie formularza. Dodaje również atrybut `enctype="multipart/form-data"` do znacznika `<form>`, jeśli formularz zawiera pole do przesyłania plików. Dodatkowo zajmuje się wyświetlaniem komunikatów o błędach, jeśli wystąpią jakieś problemy podczas przesyłania formularza. Wszystko można dostosować poprzez nadpisanie domyślnych szablonów, ale w tym projekcie nie będzie to konieczne.

### Dostosowywanie typu formularza

Nawet jeśli pola formularza są konfigurowane na podstawie odpowiadających im właściwości modelu, możesz dostosować domyślną konfigurację bezpośrednio w klasie typu formularza:

```diff
--- a/src/Form/CommentType.php
+++ b/src/Form/CommentType.php
@@ -6,26 +6,32 @@ use App\Entity\Comment;
 use App\Entity\Conference;
 use Symfony\Bridge\Doctrine\Form\Type\EntityType;
 use Symfony\Component\Form\AbstractType;
+use Symfony\Component\Form\Extension\Core\Type\EmailType;
+use Symfony\Component\Form\Extension\Core\Type\FileType;
+use Symfony\Component\Form\Extension\Core\Type\SubmitType;
 use Symfony\Component\Form\FormBuilderInterface;
 use Symfony\Component\OptionsResolver\OptionsResolver;
+use Symfony\Component\Validator\Constraints\Image;

 class CommentType extends AbstractType
 {
     public function buildForm(FormBuilderInterface $builder, array $options): void
     {
         $builder
-            ->add('author')
-            ->add('text')
-            ->add('email')
-            ->add('createdAt', null, [
-                'widget' => 'single_text',
+            ->add('author', null, [
+                'label' => 'Your name',
             ])
-            ->add('photoFilename')
-            ->add('conference', EntityType::class, [
-                'class' => Conference::class,
-                'choice_label' => 'id',
+            ->add('text')
+            ->add('email', EmailType::class)
+            ->add('photo', FileType::class, [
+                'required' => false,
+                'mapped' => false,
+                'constraints' => [
+                    new Image(['maxSize' => '1024k'])
+                ],
             ])
-        ;
+            ->add('submit', SubmitType::class)
+       ;
     }

     public function configureOptions(OptionsResolver $resolver): void
```

Zwróć uwagę, że dodaliśmy przycisk *submit* (co pozwala nam dalej korzystać z prostego wyrażenia `{{ form(comment_form) }}` w szablonie).

Niektóre pola nie mogą być skonfigurowane automatycznie, jak np. `photoFilename`. Encja `Comment` przechowuje jedynie nazwę pliku zdjęcia, ale formularz musi obsłużyć samo przesyłanie pliku. Aby to umożliwić, dodaliśmy pole `photo` jako **niemapowane** — nie będzie ono powiązane z żadną właściwością w klasie `Comment`. Obsłużymy je ręcznie, aby zaimplementować konkretną logikę (na przykład zapisanie przesłanego zdjęcia na dysku).

Jako przykład dostosowania, zmodyfikowaliśmy także domyślne etykiety dla niektórych pól.

![form-customized](https://symfony.com/doc/6.4en//the-fast-track/_images/form-customized.png)

### Walidacja modeli

Typ formularza odpowiada za konfigurację wyglądu formularza po stronie frontendowej (przy użyciu walidacji HTML5). Poniżej znajduje się wygenerowany formularz HTML:

```html
<form name="comment_form" method="post" enctype="multipart/form-data">
    <div id="comment_form">
        <div >
            <label for="comment_form_author" class="required">Your name</label>
            <input type="text" id="comment_form_author" name="comment_form[author]" required="required" maxlength="255" />
        </div>
        <div >
            <label for="comment_form_text" class="required">Text</label>
            <textarea id="comment_form_text" name="comment_form[text]" required="required"></textarea>
        </div>
        <div >
            <label for="comment_form_email" class="required">Email</label>
            <input type="email" id="comment_form_email" name="comment_form[email]" required="required" />
        </div>
        <div >
            <label for="comment_form_photo">Photo</label>
            <input type="file" id="comment_form_photo" name="comment_form[photo]" />
        </div>
        <div >
            <button type="submit" id="comment_form_submit" name="comment_form[submit]">Submit</button>
        </div>
        <input type="hidden" id="comment_form__token" name="comment_form[_token]" value="DwqsEanxc48jofxsqbGBVLQBqlVJ_Tg4u9-BL1Hjgac" />
    </div>
</form>
```

Formularz używa pola `email` dla adresu e-mail w komentarzu i oznacza większość pól jako wymagane (`required`). Zwróć uwagę, że formularz zawiera również ukryte pole `_token`, które chroni go przed [atakami CSRF](https://owasp.org/www-community/attacks/csrf).

Jednak jeśli walidacja HTML zostanie pominięta (np. przez wysłanie danych przy pomocy klienta HTTP, który nie egzekwuje zasad walidacji, jak cURL), nieprawidłowe dane mogą trafić na serwer.

Dlatego musimy dodać również odpowiednie ograniczenia walidacyjne w modelu danych `Comment`:

```diff
--- a/src/Entity/Comment.php
+++ b/src/Entity/Comment.php
@@ -5,6 +5,7 @@ namespace App\Entity;
 use App\Repository\CommentRepository;
 use Doctrine\DBAL\Types\Types;
 use Doctrine\ORM\Mapping as ORM;
+use Symfony\Component\Validator\Constraints as Assert;

 #[ORM\Entity(repositoryClass: CommentRepository::class)]
 #[ORM\HasLifecycleCallbacks]
@@ -16,12 +17,16 @@ class Comment
     private ?int $id = null;

     #[ORM\Column(length: 255)]
+    #[Assert\NotBlank]
     private ?string $author = null;

     #[ORM\Column(type: Types::TEXT)]
+    #[Assert\NotBlank]
     private ?string $text = null;

     #[ORM\Column(length: 255)]
+    #[Assert\NotBlank]
+    #[Assert\Email]
     private ?string $email = null;

     #[ORM\Column]
```

### Obsługa formularza

Kod, który napisaliśmy do tej pory, wystarcza do wyświetlenia formularza.

Teraz powinniśmy obsłużyć jego wysyłkę oraz zapis danych do bazy danych w kontrolerze:

```diff
--- a/src/Controller/ConferenceController.php
+++ b/src/Controller/ConferenceController.php
@@ -7,6 +7,7 @@ use App\Entity\Conference;
 use App\Form\CommentType;
 use App\Repository\CommentRepository;
 use App\Repository\ConferenceRepository;
+use Doctrine\ORM\EntityManagerInterface;
 use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
 use Symfony\Component\HttpFoundation\Request;
 use Symfony\Component\HttpFoundation\Response;
@@ -14,6 +15,11 @@ use Symfony\Component\Routing\Attribute\Route;

 class ConferenceController extends AbstractController
 {
+    public function __construct(
+        private EntityManagerInterface $entityManager,
+    ) {
+    }
+
     #[Route('/', name: 'homepage')]
     public function index(ConferenceRepository $conferenceRepository): Response
     {
@@ -27,6 +33,15 @@ class ConferenceController extends AbstractController
     {
         $comment = new Comment();
         $form = $this->createForm(CommentType::class, $comment);
+        $form->handleRequest($request);
+        if ($form->isSubmitted() && $form->isValid()) {
+            $comment->setConference($conference);
+
+            $this->entityManager->persist($comment);
+            $this->entityManager->flush();
+
+            return $this->redirectToRoute('conference', ['slug' => $conference->getSlug()]);
+        }

         $offset = max(0, $request->query->getInt('offset', 0));
         $paginator = $commentRepository->getCommentPaginator($conference, $offset);
```

Gdy formularz zostanie wysłany, obiekt `Comment` jest aktualizowany na podstawie przesłanych danych.

Konferencja jest ustawiana na tę samą, która znajduje się w adresie URL (usunięto ją z formularza).

Jeśli formularz nie jest poprawny, wyświetlamy stronę ponownie — ale tym razem formularz zawiera przesłane wartości oraz komunikaty o błędach, dzięki czemu użytkownik widzi, co należy poprawić.

Przetestuj formularz. Powinien działać poprawnie, a dane powinny zostać zapisane w bazie danych (możesz to sprawdzić w panelu administracyjnym). Jest jednak jeden problem: *zdjęcia*. Nie działają, ponieważ jeszcze nie zaimplementowaliśmy ich obsługi w kontrolerze.

### Przesyłanie plików

Przesłane zdjęcia powinny być przechowywane na lokalnym dysku, w miejscu dostępnym dla frontendu, tak aby można było je wyświetlić na stronie konferencji. Będziemy je zapisywać w katalogu `public/uploads/photos`.

Ponieważ nie chcemy na sztywno wpisywać ścieżki do katalogu w kodzie, musimy zapisać ją globalnie w konfiguracji. Kontener Symfony oprócz usług potrafi przechowywać także *parametry* — czyli wartości skalarne, które pomagają w konfiguracji usług:

```diff
--- a/config/services.yaml
+++ b/config/services.yaml
@@ -4,6 +4,7 @@
 # Put parameters here that don't need to change on each machine where the app is deployed
 # https://symfony.com/doc/current/best_practices.html#use-parameters-for-application-configuration
 parameters:
+    photo_dir: "%kernel.project_dir%/public/uploads/photos"

 services:
     # default configuration for services in *this* file
```

Widzieliśmy już wcześniej, jak usługi są automatycznie wstrzykiwane do argumentów konstruktora. Parametry kontenera można natomiast jawnie wstrzykiwać przy użyciu atrybutu `Autowire`.

Teraz mamy już wszystko, czego potrzeba, aby zaimplementować logikę zapisywania przesłanego pliku w jego docelowej lokalizacji:

```diff
--- a/src/Controller/ConferenceController.php
+++ b/src/Controller/ConferenceController.php
@@ -9,6 +9,7 @@ use App\Repository\CommentRepository;
 use App\Repository\ConferenceRepository;
 use Doctrine\ORM\EntityManagerInterface;
 use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
+use Symfony\Component\DependencyInjection\Attribute\Autowire;
 use Symfony\Component\HttpFoundation\Request;
 use Symfony\Component\HttpFoundation\Response;
 use Symfony\Component\Routing\Attribute\Route;
@@ -29,13 +30,22 @@ class ConferenceController extends AbstractController
     }

     #[Route('/conference/{slug}', name: 'conference')]
-    public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
-    {
+    public function show(
+        Request $request,
+        Conference $conference,
+        CommentRepository $commentRepository,
+        #[Autowire('%photo_dir%')] string $photoDir,
+    ): Response {
         $comment = new Comment();
         $form = $this->createForm(CommentType::class, $comment);
         $form->handleRequest($request);
         if ($form->isSubmitted() && $form->isValid()) {
             $comment->setConference($conference);
+            if ($photo = $form['photo']->getData()) {
+                $filename = bin2hex(random_bytes(6)).'.'.$photo->guessExtension();
+                $photo->move($photoDir, $filename);
+                $comment->setPhotoFilename($filename);
+            }

             $this->entityManager->persist($comment);
             $this->entityManager->flush();
```

Aby obsłużyć przesyłanie zdjęć, tworzymy losową nazwę pliku. Następnie przenosimy przesłany plik do ostatecznego katalogu (czyli katalogu zdjęć). Na końcu zapisujemy nazwę pliku w obiekcie `Comment`.

Spróbuj przesłać plik PDF zamiast zdjęcia. Powinieneś zobaczyć działające komunikaty o błędach. Wygląd formularza jest na razie dość surowy, ale nie martw się — wszystko wkrótce nabierze estetyki, gdy przejdziemy do projektowania wyglądu strony. W przypadku formularzy wystarczy zmienić jedną linię konfiguracji, aby ostylować wszystkie elementy formularza.

### Debugowanie formularzy

Gdy formularz zostanie wysłany i coś nie działa poprawnie, skorzystaj z panelu *Form* w Symfony Profilerze. Zawiera on szczegółowe informacje o formularzu, jego opcjach, przesłanych danych oraz sposobie ich wewnętrznej konwersji. Jeśli formularz zawiera jakiekolwiek błędy, również zostaną tam wyświetlone.

Typowy przebieg pracy z formularzem wygląda następująco:  
- Formularz jest wyświetlany na stronie;  
- Użytkownik wysyła formularz za pomocą żądania POST;  
- Serwer przekierowuje użytkownika na inną stronę lub tę samą stronę.

Ale jak uzyskać dostęp do profilera po *pomyślnym* przesłaniu formularza? Ponieważ następuje natychmiastowe przekierowanie, nie widzimy pasku debugowania dla żądania POST. Żaden problem — na stronie, na którą nastąpiło przekierowanie, najedź kursorem na zielony fragment z kodem „200” w lewym dolnym rogu. Powinien pojawić się czerwony wpis „302” z linkiem do profilu (w nawiasie).

![form-wdt](https://symfony.com/doc/6.4en//the-fast-track/_images/form-wdt.png)

Kliknij ten link, aby przejść do profilu żądania POST, a następnie otwórz panel *Form*.

![form-profiler](https://symfony.com/doc/6.4en//the-fast-track/_images/form-profiler.png)

### Wyświetlanie przesłanych zdjęć w panelu administracyjnym

Obecnie panel administracyjny wyświetla tylko nazwę pliku zdjęcia, ale chcemy, aby pokazywał rzeczywiste zdjęcie:

```diff
--- a/src/Controller/Admin/CommentCrudController.php
+++ b/src/Controller/Admin/CommentCrudController.php
@@ -10,6 +10,7 @@ use EasyCorp\Bundle\EasyAdminBundle\Field\AssociationField;
 use EasyCorp\Bundle\EasyAdminBundle\Field\DateTimeField;
 use EasyCorp\Bundle\EasyAdminBundle\Field\EmailField;
 use EasyCorp\Bundle\EasyAdminBundle\Field\IdField;
+use EasyCorp\Bundle\EasyAdminBundle\Field\ImageField;
 use EasyCorp\Bundle\EasyAdminBundle\Field\TextareaField;
 use EasyCorp\Bundle\EasyAdminBundle\Field\TextEditorField;
 use EasyCorp\Bundle\EasyAdminBundle\Field\TextField;
@@ -47,7 +48,9 @@ class CommentCrudController extends AbstractCrudController
         yield TextareaField::new('text')
             ->hideOnIndex()
         ;
-        yield TextField::new('photoFilename')
+        yield ImageField::new('photoFilename')
+            ->setBasePath('/uploads/photos')
+            ->setLabel('Photo')
             ->onlyOnIndex()
         ;
```

### Wykluczanie przesłanych zdjęć z systemu Git

Jeszcze nie wykonuj commit'a! Nie chcemy przechowywać przesłanych obrazów w repozytorium Git. Dodaj katalog `/public/uploads` do pliku `.gitignore`:

```diff
--- a/.gitignore
+++ b/.gitignore
@@ -1,3 +1,4 @@
+/public/uploads

 ###> symfony/framework-bundle ###
 /.env.local
```

### Przechowywanie przesłanych plików na serwerach produkcyjnych

Ostatnim krokiem jest zapisanie przesłanych plików na serwerach produkcyjnych. Dlaczego musimy zrobić coś specjalnego? Ponieważ większość nowoczesnych platform chmurowych z różnych powodów używa kontenerów tylko do odczytu. Platform.sh nie jest wyjątkiem.

Nie wszystko jest jednak tylko do odczytu w projekcie Symfony. Staramy się generować jak najwięcej cache’u podczas budowania kontenera (czyli w fazie "cache warmup"), ale Symfony nadal potrzebuje zapisywać dane w niektórych miejscach, takich jak pamięć podręczna użytkownika, logi, sesje (jeśli są przechowywane na dysku) itp.

Spójrz na plik `.platform.app.yaml` — znajduje się tam już zapisujący punkt montowania dla katalogu `var/`. `var/` to domyślny katalog, do którego Symfony zapisuje dane (cache, logi itd.).

Dodajmy nowy punkt montowania dla przesłanych zdjęć:

```diff
--- a/.platform.app.yaml
+++ b/.platform.app.yaml
@@ -35,6 +35,7 @@ web:

 mounts:
     "/var": { source: local, source_path: var }
+    "/public/uploads": { source: local, source_path: uploads }


 relationships:
```

Teraz możesz wdrożyć kod, a zdjęcia będą zapisywane w katalogu `public/uploads/`, tak jak w wersji lokalnej.

### Sprawdź również:
- [Kurs „Forms” na SymfonyCasts](https://symfonycasts.com/screencast/symfony-forms);
- Jak [dostosować renderowanie formularzy Symfony w HTML](https://symfony.com/doc/current/form/form_customization.html);
- [Walidacja formularzy Symfony](https://symfony.com/doc/current/forms.html#validating-forms);
- [Dokumentacja typów formularzy Symfony](https://symfony.com/doc/current/reference/forms/types.html);
- Dokumentacja [FlysystemBundle](https://github.com/thephpleague/flysystem-bundle/blob/master/docs/1-getting-started.md) – integracja z wieloma usługami chmurowymi, takimi jak AWS S3, Azure i Google Cloud Storage;
- [Dokumentacja parametrów konfiguracyjnych Symfony](https://symfony.com/doc/current/configuration.html#configuration-parameters);
- [Ograniczenia walidacyjne Symfony](https://symfony.com/doc/current/validation.html#basic-constraints);
- [Ściąga do formularzy Symfony](https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony2/how_symfony2_forms_works_en.pdf).

---

- **Poprzednia strona:** [Zarządzanie cyklem życia obiektów Doctrine](13-lifecycle.md)
- **Następna strona:** [Zabezpieczanie panelu administracyjnego](15-security.md)
