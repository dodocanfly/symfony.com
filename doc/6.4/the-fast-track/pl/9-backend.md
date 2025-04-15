## Konfigurowanie zaplecza administratora

Dodawanie nadchodzących konferencji do bazy danych należy do zadań administratorów projektu. Zaplecze administratora to chroniona sekcja strony internetowej, w której administratorzy projektu mogą zarządzać danymi strony, moderować zgłoszenia opinii i nie tylko.

Jak możemy to zrobić szybko? Korzystając z pakietu, który potrafi wygenerować zaplecze administratora na podstawie modelu projektu. EasyAdmin idealnie się do tego nadaje.

### Instalowanie dodatkowych zależności

Chociaż pakiet webapp automatycznie dodał wiele przydatnych pakietów, do niektórych bardziej specyficznych funkcji musimy dodać dodatkowe zależności. Jak możemy dodać więcej zależności? Za pomocą Composera. Oprócz "zwykłych" pakietów Composera, będziemy pracować z dwoma "specjalnymi" rodzajami pakietów:
 
- **Komponenty Symfony**: Pakiety, które implementują funkcje podstawowe i niskopoziomowe abstrakcje, które są potrzebne większości aplikacji (routing, konsola, klient HTTP, mailer, cache, ...);
- **Bundle Symfony**: Pakiety, które dodają funkcje wysokiego poziomu lub zapewniają integrację z bibliotekami zewnętrznymi (bundles są głównie tworzone przez społeczność).

Dodajmy EasyAdmin jako zależność projektu:

```bash
symfony composer req "admin:^4"
# lub
symfony composer req "easycorp/easyadmin-bundle:4.x-dev"
```

`admin` to alias dla pakietu `easycorp/easyadmin-bundle`.

Alias to nie funkcja Composera, ale koncepcja dostarczona przez Symfony, mająca na celu ułatwienie pracy. Aliasy to skróty dla popularnych pakietów Composera. Chcesz ORM dla swojej aplikacji? Wymagaj `orm`. Chcesz rozwijać API? Wymagaj `api`. Te aliasy są automatycznie przekształcane na jeden lub więcej standardowych pakietów Composera. Są to wybory dokonane przez zespół rdzenia Symfony.
Kolejną przydatną funkcją jest to, że zawsze możesz pominąć prefiks symfony. Wymagaj `cache` zamiast `symfony/cache`.

> [!NOTE]
> Pamiętasz, że wcześniej wspomnieliśmy o wtyczce Composera o nazwie `symfony/flex`? Aliasy to jedna z jej funkcji.

### Konfigurowanie EasyAdmin

EasyAdmin automatycznie generuje panel administracyjny dla Twojej aplikacji na podstawie określonych kontrolerów.

Aby rozpocząć pracę z EasyAdmin, wygenerujmy „dashboard administracyjny”, który będzie głównym punktem wejścia do zarządzania danymi strony:

```bash
symfony console make:admin:dashboard
```

Zaakceptowanie domyślnych odpowiedzi spowoduje utworzenie następującego kontrolera:

**src/Controller/Admin/DashboardController.php**
```php
namespace App\Controller\Admin;

use EasyCorp\Bundle\EasyAdminBundle\Config\Dashboard;
use EasyCorp\Bundle\EasyAdminBundle\Config\MenuItem;
use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractDashboardController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class DashboardController extends AbstractDashboardController
{
    /**
     * @Route("/admin", name="admin")
     */
    public function index(): Response
    {
        return parent::index();
    }

    public function configureDashboard(): Dashboard
    {
        return Dashboard::new()
            ->setTitle('Guestbook');
    }

    public function configureMenuItems(): iterable
    {
        yield MenuItem::linkToDashboard('Dashboard', 'fa fa-home');
        // yield MenuItem::linkToCrud('The Label', 'icon class', EntityClass::class);
    }
}
```

Zgodnie z konwencją, wszystkie kontrolery administracyjne są przechowywane w przestrzeni nazw `App\Controller\Admin`.

Dostęp do wygenerowanego zaplecza administracyjnego uzyskasz pod adresem `/admin`, jak skonfigurowano w metodzie `index()`; możesz zmienić ten adres URL na dowolny inny:

![easy-admin-empty](https://symfony.com/doc/6.4en//the-fast-track/_images/easy-admin-empty.png)

Boom! Mamy ładnie wyglądający szkielet panelu administracyjnego, gotowy do dostosowania do naszych potrzeb.

Następnym krokiem jest utworzenie kontrolerów do zarządzania konferencjami i komentarzami.

W kontrolerze dashboardu mogłeś zauważyć metodę `configureMenuItems()`, która zawiera komentarz dotyczący dodawania linków do „CRUD-ów”. **CRUD** to skrót od „Create, Read, Update, Delete” (Tworzenie, Odczyt, Aktualizacja, Usuwanie) — cztery podstawowe operacje, które wykonuje się na każdej encji. Dokładnie tego oczekujemy od panelu administratora. EasyAdmin idzie o krok dalej i zapewnia także funkcje wyszukiwania oraz filtrowania.

Wygenerujmy teraz CRUD dla konferencji:

```bash
symfony console make:admin:crud
```

Wybierz `1`, aby utworzyć interfejs administracyjny dla konferencji i zaakceptuj domyślne odpowiedzi dla pozostałych pytań. Powinien zostać wygenerowany następujący plik:

**src/Controller/Admin/ConferenceCrudController.php**
```php
namespace App\Controller\Admin;

use App\Entity\Conference;
use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractCrudController;

class ConferenceCrudController extends AbstractCrudController
{
    public static function getEntityFqcn(): string
    {
        return Conference::class;
    }

    /*
    public function configureFields(string $pageName): iterable
    {
        return [
            IdField::new('id'),
            TextField::new('title'),
            TextEditorField::new('description'),
        ];
    }
    */
}
```

Zrób to samo dla komentarzy:

```bash
symfony console make:admin:crud
```

Ostatnim krokiem jest podpięcie paneli administracyjnych dla konferencji i komentarzy do dashboardu:

```diff
--- a/src/Controller/Admin/DashboardController.php
+++ b/src/Controller/Admin/DashboardController.php
@@ -2,6 +2,8 @@

 namespace App\Controller\Admin;

+use App\Entity\Comment;
+use App\Entity\Conference;
 use EasyCorp\Bundle\EasyAdminBundle\Config\Dashboard;
 use EasyCorp\Bundle\EasyAdminBundle\Config\MenuItem;
 use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractDashboardController;
@@ -40,7 +42,8 @@ class DashboardController extends AbstractDashboardController

     public function configureMenuItems(): iterable
     {
-        yield MenuItem::linkToDashboard('Dashboard', 'fa fa-home');
-        // yield MenuItem::linkToCrud('The Label', 'fas fa-list', EntityClass::class);
+        yield MenuItem::linktoRoute('Back to the website', 'fas fa-home', 'homepage');
+        yield MenuItem::linkToCrud('Conferences', 'fas fa-map-marker-alt', Conference::class);
+        yield MenuItem::linkToCrud('Comments', 'fas fa-comments', Comment::class);
     }
 }
```

Nadpisaliśmy metodę `configureMenuItems()`, aby dodać pozycje menu z odpowiednimi ikonami dla konferencji i komentarzy oraz dodać link powrotny do strony głównej serwisu.

EasyAdmin udostępnia API, które ułatwia linkowanie do CRUD-ów encji za pomocą metody `MenuItem::linkToCrud()`.

Główna strona dashboardu jest na razie pusta. To miejsce, w którym możesz wyświetlać statystyki lub inne istotne informacje. Ponieważ nie mamy obecnie nic ważnego do pokazania, przekierujmy użytkownika od razu na listę konferencji:

```diff
--- a/src/Controller/Admin/DashboardController.php
+++ b/src/Controller/Admin/DashboardController.php
@@ -7,6 +7,7 @@ use App\Entity\Conference;
 use EasyCorp\Bundle\EasyAdminBundle\Config\Dashboard;
 use EasyCorp\Bundle\EasyAdminBundle\Config\MenuItem;
 use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractDashboardController;
+use EasyCorp\Bundle\EasyAdminBundle\Router\AdminUrlGenerator;
 use Symfony\Component\HttpFoundation\Response;
 use Symfony\Component\Routing\Annotation\Route;

@@ -15,7 +16,10 @@ class DashboardController extends AbstractDashboardController
     #[Route('/admin', name: 'admin')]
     public function index(): Response
     {
-        return parent::index();
+        $routeBuilder = $this->container->get(AdminUrlGenerator::class);
+        $url = $routeBuilder->setController(ConferenceCrudController::class)->generateUrl();
+
+        return $this->redirect($url);

         // Option 1. You can make your dashboard redirect to some common page of your backend
         //
```

Podczas wyświetlania relacji między encjami (np. konferencji powiązanej z komentarzem), EasyAdmin stara się użyć reprezentacji tekstowej konferencji. Domyślnie stosuje konwencję, która wykorzystuje nazwę encji oraz jej klucz główny (np. `Conference #1`), jeśli encja nie definiuje „magicznej” metody `__toString()`. Aby wyświetlanie było bardziej czytelne i znaczące, dodaj taką metodę w klasie `Conference`:

```diff
--- a/src/Entity/Conference.php
+++ b/src/Entity/Conference.php
@@ -35,6 +35,11 @@ class Conference
         $this->comments = new ArrayCollection();
     }

+    public function __toString(): string
+    {
+        return $this->city.' '.$this->year;
+    }
+
     public function getId(): ?int
     {
         return $this->id;
```

Możesz teraz dodawać, modyfikować i usuwać konferencje bezpośrednio z panelu administracyjnego. Pobaw się tym i dodaj przynajmniej jedną konferencję.

![easy-admin](https://symfony.com/doc/6.4en//the-fast-track/_images/easy-admin.png)

### Dostosowywanie EasyAdmin

Domyślny panel administracyjny działa dobrze, ale można go na wiele sposobów dostosować, aby poprawić wygodę korzystania. Wprowadźmy kilka prostych zmian w encji `Comment`, aby pokazać niektóre z dostępnych możliwości:

```diff
--- a/src/Controller/Admin/CommentCrudController.php
+++ b/src/Controller/Admin/CommentCrudController.php
@@ -3,10 +3,17 @@
 namespace App\Controller\Admin;

 use App\Entity\Comment;
+use EasyCorp\Bundle\EasyAdminBundle\Config\Crud;
+use EasyCorp\Bundle\EasyAdminBundle\Config\Filters;
 use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractCrudController;
+use EasyCorp\Bundle\EasyAdminBundle\Field\AssociationField;
+use EasyCorp\Bundle\EasyAdminBundle\Field\DateTimeField;
+use EasyCorp\Bundle\EasyAdminBundle\Field\EmailField;
 use EasyCorp\Bundle\EasyAdminBundle\Field\IdField;
+use EasyCorp\Bundle\EasyAdminBundle\Field\TextareaField;
 use EasyCorp\Bundle\EasyAdminBundle\Field\TextEditorField;
 use EasyCorp\Bundle\EasyAdminBundle\Field\TextField;
+use EasyCorp\Bundle\EasyAdminBundle\Filter\EntityFilter;

 class CommentCrudController extends AbstractCrudController
 {
@@ -15,14 +22,43 @@ class CommentCrudController extends AbstractCrudController
         return Comment::class;
     }

-    /*
+    public function configureCrud(Crud $crud): Crud
+    {
+        return $crud
+            ->setEntityLabelInSingular('Conference Comment')
+            ->setEntityLabelInPlural('Conference Comments')
+            ->setSearchFields(['author', 'text', 'email'])
+            ->setDefaultSort(['createdAt' => 'DESC'])
+        ;
+    }
+
+    public function configureFilters(Filters $filters): Filters
+    {
+        return $filters
+            ->add(EntityFilter::new('conference'))
+        ;
+    }
+
     public function configureFields(string $pageName): iterable
     {
-        return [
-            IdField::new('id'),
-            TextField::new('title'),
-            TextEditorField::new('description'),
-        ];
+        yield AssociationField::new('conference');
+        yield TextField::new('author');
+        yield EmailField::new('email');
+        yield TextareaField::new('text')
+            ->hideOnIndex()
+        ;
+        yield TextField::new('photoFilename')
+            ->onlyOnIndex()
+        ;
+
+        $createdAt = DateTimeField::new('createdAt')->setFormTypeOptions([
+            'years' => range(date('Y'), date('Y') + 5),
+            'widget' => 'single_text',
+        ]);
+        if (Crud::PAGE_EDIT === $pageName) {
+            yield $createdAt->setFormTypeOption('disabled', true);
+        } else {
+            yield $createdAt;
+        }
     }
-    */
 }
```

Aby dostosować sekcję *komentarzy*, wypisanie pól jawnie w metodzie `configureFields()` pozwala nam uporządkować je według własnych preferencji. Niektóre pola są dodatkowo konfigurowane, na przykład ukrywanie pola tekstowego na stronie indeksu.

Dodaj kilka komentarzy bez zdjęć. Na razie ustaw datę ręcznie; kolumnę `createdAt` wypełnimy automatycznie w późniejszym kroku.

![easy-admin-comments](https://symfony.com/doc/6.4en//the-fast-track/_images/easy-admin-comments.png)

Metoda `configureFilters()` określa, które filtry mają być dostępne nad polem wyszukiwania.

![easy-admin-filter](https://symfony.com/doc/6.4en//the-fast-track/_images/easy-admin-filter.png)

Te dostosowania to tylko małe wprowadzenie do możliwości, jakie oferuje EasyAdmin.

Pobaw się panelem administracyjnym — przefiltruj komentarze według konferencji albo wyszukaj komentarze po adresie e-mail, na przykład. Jedynym problemem jest to, że każdy może uzyskać dostęp do panelu. Nie martw się, w kolejnych krokach go zabezpieczymy.

### Sprawdź również:
- [Dokumentacja EasyAdmin](https://symfony.com/bundles/EasyAdminBundle/4.x/index.html);
- [Referencja konfiguracji frameworka Symfony](https://symfony.com/doc/current/reference/configuration/framework.html);
- [Magiczne metody w PHP](https://www.php.net/manual/en/language.oop5.magic.php).

---

- **Poprzednia strona:** [Opis struktury danych](8-doctrine.md)
- **Następna strona:** [Budowanie interfejsu użytkownika](10-twig.md)
