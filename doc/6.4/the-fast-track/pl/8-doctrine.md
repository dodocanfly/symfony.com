## Opis struktury danych

Aby obsługiwać bazę danych z poziomu PHP, będziemy korzystać z Doctrine — zestawu bibliotek, które pomagają programistom zarządzać bazami danych: Doctrine DBAL (warstwa abstrakcji bazy danych), Doctrine ORM (biblioteka umożliwiająca manipulację zawartością bazy danych za pomocą obiektów PHP) oraz Doctrine Migrations.

### Konfiguracja Doctrine ORM

Skąd Doctrine wie, jak połączyć się z bazą danych? Przepis Doctrine dodaje plik konfiguracyjny `config/packages/doctrine.yaml`, który kontroluje jego działanie. Główne ustawienie to DSN bazy danych — ciąg znaków zawierający wszystkie informacje o połączeniu: dane uwierzytelniające, host, port itd. Domyślnie Doctrine szuka zmiennej środowiskowej `DATABASE_URL`.

Prawie wszystkie zainstalowane pakiety mają swoją konfigurację w katalogu `config/packages/`. W większości przypadków domyślne ustawienia zostały starannie dobrane, aby działały dla większości aplikacji.

### Zrozumienie konwencji zmiennych środowiskowych w Symfony

Możesz ręcznie zdefiniować `DATABASE_URL` w pliku `.env` lub `.env.local`. W rzeczywistości, dzięki przepisowi pakietu, znajdziesz przykładową zmienną `DATABASE_URL` w pliku `.env`. Jednak ponieważ port lokalny PostgreSQL udostępniany przez Dockera może się zmieniać, takie podejście bywa uciążliwe. Istnieje lepsze rozwiązanie.

Zamiast na sztywno wpisywać `DATABASE_URL` do pliku, możemy poprzedzać wszystkie polecenia słowem `symfony`. Dzięki temu Symfony automatycznie wykryje usługi uruchomione przez Dockera i/lub Platform.sh (gdy tunel jest otwarty) i ustawi zmienne środowiskowe automatycznie.

Docker Compose i Platform.sh współpracują bezproblemowo z Symfony właśnie dzięki tym zmiennym środowiskowym.

Sprawdź wszystkie dostępne zmienne środowiskowe, wykonując polecenie:

```bash
symfony var:export
```

Przykład:

```bash
DATABASE_URL=postgres://app:!ChangeMe!@127.0.0.1:32781/app?sslmode=disable&charset=utf8
# ...
```

Pamiętasz nazwę usługi bazy danych używaną w konfiguracjach Dockera i Platform.sh? Nazwy tych usług są wykorzystywane jako prefiksy do definiowania zmiennych środowiskowych, takich jak `DATABASE_URL`. Jeśli twoje usługi są nazwane zgodnie z konwencjami Symfony, nie jest wymagana żadna dodatkowa konfiguracja.

> [!NOTE]
> Bazy danych nie są jedynymi usługami, które korzystają z konwencji Symfony. Dotyczy to również np. systemu Mailer (poprzez zmienną `MAILER_DSN`).

### Zmiana domyślnej wartości `DATABASE_URL` w pliku .env

Wciąż musimy zmienić plik `.env`, aby ustawić domyślną wartość `DATABASE_URL` na użycie PostgreSQL:

```diff
--- a/.env
+++ b/.env
@@ -26,7 +26,7 @@ APP_SECRET=ce2ae8138936039d22afb20f4596fe97
 # DATABASE_URL="sqlite:///%kernel.project_dir%/var/data.db"
 # DATABASE_URL="mysql://app:!ChangeMe!@127.0.0.1:3306/app?serverVersion=8.0.32&charset=utf8mb4"
 # DATABASE_URL="mysql://app:!ChangeMe!@127.0.0.1:3306/app?serverVersion=10.11.2-MariaDB&charset=utf8mb4"
-DATABASE_URL="postgresql://app:!ChangeMe!@127.0.0.1:5432/app?serverVersion=16&charset=utf8"
+DATABASE_URL="postgresql://127.0.0.1:5432/db?serverVersion=16&charset=utf8"
```

Dlaczego te informacje muszą być zduplikowane w dwóch różnych miejscach? Ponieważ na niektórych platformach chmurowych podczas *budowania aplikacji* adres URL bazy danych może jeszcze nie być znany, ale Doctrine musi wiedzieć, jaki silnik bazy danych będzie użyty, aby zbudować swoją konfigurację. Dlatego host, nazwa użytkownika i hasło nie są w tym momencie istotne.

### Tworzenie klas encji

Konferencję można opisać kilkoma właściwościami:

- *Miasto*, w którym organizowana jest konferencja;
- *Rok* konferencji;
- Flaga *isInternational*, określająca czy konferencja jest lokalna, czy międzynarodowa (SymfonyLive vs SymfonyCon).

Pakiet Maker może pomóc w wygenerowaniu klasy (klasy encji), która będzie reprezentować konferencję.

Nadszedł czas, aby wygenerować encję `Conference`:

```bash
symfony console make:entity Conference
```

To polecenie działa interaktywnie — poprowadzi cię przez proces dodawania wszystkich wymaganych pól. Użyj następujących odpowiedzi (większość z nich to wartości domyślne, więc możesz po prostu naciskać Enter):
 
- `city`, `string`, `255`, `no`;
- `year`, `string`, `4`, `no`;
- `isInternational`, `boolean`, `no`.

Przykładowy wynik polecenia: 

```
created: src/Entity/Conference.php
created: src/Repository/ConferenceRepository.php

Entity generated! Now let's add some fields!
You can always add more fields later manually or by re-running this command.

New property name (press <return> to stop adding fields):
> city

Field type (enter ? to see all types) [string]:
>

Field length [255]:
>

Can this field be null in the database (nullable) (yes/no) [no]:
>

updated: src/Entity/Conference.php

Add another property? Enter the property name (or press <return> to stop adding fields):
> year

Field type (enter ? to see all types) [string]:
>

Field length [255]:
> 4

Can this field be null in the database (nullable) (yes/no) [no]:
>

updated: src/Entity/Conference.php

Add another property? Enter the property name (or press <return> to stop adding fields):
> isInternational

Field type (enter ? to see all types) [boolean]:
>

Can this field be null in the database (nullable) (yes/no) [no]:
>

updated: src/Entity/Conference.php

Add another property? Enter the property name (or press <return> to stop adding fields):
>



 Success!


Next: When you're ready, create a migration with make:migration
```

Klasa `Conference` została zapisana w przestrzeni nazw `App\Entity`.

Polecenie wygenerowało również klasę repozytorium Doctrine: `App\Repository\ConferenceRepository`.

Fragment wygenerowanego kodu:

**src/Entity/Conference.php**
```php
namespace App\Entity;

use App\Repository\ConferenceRepository;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: ConferenceRepository::class)]
class Conference
{
    #[ORM\Column(type: 'integer')]
    #[ORM\Id, ORM\GeneratedValue()]
    private $id;

    #[ORM\Column(type: 'string', length: 255)]
    private $city;

    // ...

    public function getCity(): ?string
    {
        return $this->city;
    }

    public function setCity(string $city): self
    {
        $this->city = $city;

        return $this;
    }

    // ...
}
```

Zwróć uwagę, że sama klasa to zwykła klasa PHP bez żadnych oznak działania Doctrine. Atrybuty (atrybuty PHP 8+) są używane do dodania metadanych, które Doctrine wykorzystuje do mapowania klasy na odpowiadającą jej tabelę w bazie danych.

Doctrine automatycznie dodało właściwość `id` do przechowywania klucza głównego w tabeli. Ten klucz (`ORM\Id()`) jest automatycznie generowany (`ORM\GeneratedValue()`) zgodnie z wybraną strategią, zależną od silnika bazy danych.

Teraz wygeneruj klasę encji dla komentarzy do konferencji:

```bash
symfony console make:entity Comment
```

Wprowadź następujące odpowiedzi:

- `author`, `string`, `255`, `no`;
- `text`, `text`, `no`;
- `email`, `string`, `255`, `no`;
- `createdAt`, `datetime_immutable`, `no`.

### Łączenie encji

Dwie encje — `Conference` i `Comment` — powinny zostać ze sobą powiązane. Konferencja może mieć zero lub więcej komentarzy, co nazywamy relacją *jeden-do-wielu*  (*one-to-many*).

Użyj ponownie polecenia `make:entity`, aby dodać tę relację do klasy `Conference`:

```bash
symfony console make:entity Conference
```

```
Your entity already exists! So let's add some new fields!

New property name (press <return> to stop adding fields):
> comments

Field type (enter ? to see all types) [string]:
> OneToMany

What class should this entity be related to?:
> Comment

A new property will also be added to the Comment class...

New field name inside Comment [conference]:
>

Is the Comment.conference property allowed to be null (nullable)? (yes/no) [yes]:
> no

Do you want to activate orphanRemoval on your relationship?
A Comment is "orphaned" when it is removed from its related Conference.
e.g. $conference->removeComment($comment)

NOTE: If a Comment may *change* from one Conference to another, answer "no".

Do you want to automatically delete orphaned App\Entity\Comment objects (orphanRemoval)? (yes/no) [no]:
> yes

updated: src/Entity/Conference.php
updated: src/Entity/Comment.php
```

W ten sposób Symfony automatycznie skonfiguruje relację jeden-do-wielu między konferencją a komentarzami, aktualizując obie klasy.

> [!TIP]
> Jeśli wpiszesz `?` jako odpowiedź dla typu pola, otrzymasz listę wszystkich obsługiwanych typów:
> ```
> Główne typy
> - `string`
> - `text`
> - `boolean`
> - `integer` (lub `smallint`, `bigint`)
> - `float`
> 
> Relacje / Powiązania
> - `relation` (kreator pomoże zbudować relację)
> - `ManyToOne`
> - `OneToMany`
> - `ManyToMany`
> - `OneToOne`
> 
> Typy tablic/obiektów
> - `array` (lub `simple_array`)
> - `json`
> - `object`
> - `binary`
> - `blob`
>
> Typy daty/czasu
> - `datetime` (lub `datetime_immutable`)
> - `datetimetz` (lub `datetimetz_immutable`)
> - `date` (lub `date_immutable`)
> - `time` (lub `time_immutable`)
> - `dateinterval`
>
> Inne typy
> - `decimal`
> - `guid`
> - `json_array`
> ```

Spójrz na pełną różnicę (*diff*) w klasach encji po dodaniu relacji:

```diff
--- a/src/Entity/Comment.php
+++ b/src/Entity/Comment.php
@@ -36,6 +36,12 @@ class Comment
      */
     private $createdAt;

+    #[ORM\ManyToOne(inversedBy: 'comments')]
+    #[ORM\JoinColumn(nullable: false)]
+    private Conference $conference;
+
     public function getId(): ?int
     {
         return $this->id;
@@ -88,4 +94,16 @@ class Comment

         return $this;
     }
+
+    public function getConference(): ?Conference
+    {
+        return $this->conference;
+    }
+
+    public function setConference(?Conference $conference): self
+    {
+        $this->conference = $conference;
+
+        return $this;
+    }
 }
--- a/src/Entity/Conference.php
+++ b/src/Entity/Conference.php
@@ -2,6 +2,8 @@

 namespace App\Entity;

+use Doctrine\Common\Collections\ArrayCollection;
+use Doctrine\Common\Collections\Collection;
 use Doctrine\ORM\Mapping as ORM;

 /**
@@ -31,6 +33,16 @@ class Conference
      */
     private $isInternational;

+    #[ORM\OneToMany(targetEntity: Comment::class, mappedBy: "conference", orphanRemoval: true)]
+    private $comments;
+
+    public function __construct()
+    {
+        $this->comments = new ArrayCollection();
+    }
+
     public function getId(): ?int
     {
         return $this->id;
@@ -71,4 +83,35 @@ class Conference

         return $this;
     }
+
+    /**
+     * @return Collection<int, Comment>
+     */
+    public function getComments(): Collection
+    {
+        return $this->comments;
+    }
+
+    public function addComment(Comment $comment): self
+    {
+        if (!$this->comments->contains($comment)) {
+            $this->comments[] = $comment;
+            $comment->setConference($this);
+        }
+
+        return $this;
+    }
+
+    public function removeComment(Comment $comment): self
+    {
+        if ($this->comments->contains($comment)) {
+            $this->comments->removeElement($comment);
+            // set the owning side to null (unless already changed)
+            if ($comment->getConference() === $this) {
+                $comment->setConference(null);
+            }
+        }
+
+        return $this;
+    }
 }
```

Wszystko, co potrzebne do zarządzania relacją, zostało wygenerowane automatycznie. Po wygenerowaniu kod staje się Twoją własnością — możesz go dowolnie modyfikować i dostosowywać.

### Dodawanie kolejnych właściwości

Właśnie zdałem sobie sprawę, że zapomnieliśmy dodać jednej właściwości w encji `Comment`: uczestnicy mogą chcieć dołączyć zdjęcie z konferencji, aby zilustrować swoją opinię.

Uruchom ponownie `make:entity` i dodaj kolumnę/właściwość `photoFilename` typu `string`, ale pozwól, aby mogła być pusta (null), ponieważ dodanie zdjęcia jest opcjonalne:

```bash
symfony console make:entity Comment
```

### Migracja bazy danych

Model projektu został teraz w pełni opisany przez dwie wygenerowane klasy.

Następnie musimy utworzyć tabele w bazie danych odpowiadające tym encjom PHP.

Doctrine Migrations doskonale nadaje się do tego zadania. Został już zainstalowany jako część zależności `orm`.

Migracja to klasa opisująca zmiany niezbędne do zaktualizowania schematu bazy danych z obecnego stanu do nowego, zdefiniowanego przez atrybuty encji. Ponieważ baza danych jest obecnie pusta, migracja powinna składać się z utworzenia dwóch tabel.

Zobaczmy, co wygeneruje Doctrine:

```bash
symfony console make:migration
```

Zwróć uwagę na nazwę wygenerowanego pliku w wyniku działania komendy (coś w stylu `migrations/Version20191019083640.php`):

**migrations/Version20191019083640.php**
```php
namespace DoctrineMigrations;

use Doctrine\DBAL\Schema\Schema;
use Doctrine\Migrations\AbstractMigration;

final class Version00000000000000 extends AbstractMigration
{
    public function up(Schema $schema): void
    {
        // this up() migration is auto-generated, please modify it to your needs
        $this->addSql('CREATE SEQUENCE comment_id_seq INCREMENT BY 1 MINVALUE 1 START 1');
        $this->addSql('CREATE SEQUENCE conference_id_seq INCREMENT BY 1 MINVALUE 1 START 1');
        $this->addSql('CREATE TABLE comment (id INT NOT NULL, conference_id INT NOT NULL, author VARCHAR(255) NOT NULL, text TEXT NOT NULL, email VARCHAR(255) NOT NULL, created_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL, photo_filename VARCHAR(255) DEFAULT NULL, PRIMARY KEY(id))');
        $this->addSql('CREATE INDEX IDX_9474526C604B8382 ON comment (conference_id)');
        $this->addSql('CREATE TABLE conference (id INT NOT NULL, city VARCHAR(255) NOT NULL, year VARCHAR(4) NOT NULL, is_international BOOLEAN NOT NULL, PRIMARY KEY(id))');
        $this->addSql('ALTER TABLE comment ADD CONSTRAINT FK_9474526C604B8382 FOREIGN KEY (conference_id) REFERENCES conference (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
    }

    public function down(Schema $schema): void
    {
        // ...
    }
}
```

Ten plik zawiera wszystkie potrzebne instrukcje SQL do utworzenia tabel i relacji między nimi.

### Aktualizacja lokalnej bazy danych

Teraz możesz uruchomić wygenerowaną migrację, aby zaktualizować lokalny schemat bazy danych:

```bash
symfony console doctrine:migrations:migrate
```

Lokalny schemat bazy danych jest teraz zaktualizowany i gotowy do przechowywania danych.

### Aktualizacja bazy danych produkcyjnej

Kroki potrzebne do migracji bazy danych produkcyjnej są takie same, jak te, które już znasz: zatwierdź zmiany i wdroż.

Podczas wdrażania projektu, Platform.sh aktualizuje kod, ale także uruchamia migrację bazy danych, jeśli taka istnieje (wykrywa, czy polecenie `doctrine:migrations:migrate` jest dostępne).

### Sprawdź również:
- [Bazy danych i Doctrine ORM](https://symfony.com/doc/current/doctrine.html) w aplikacjach Symfony;
- [Samouczek SymfonyCasts dotyczący Doctrine](https://symfonycasts.com/screencast/symfony-doctrine/install);
- [Praca z powiązaniami/relacjami Doctrine](https://symfony.com/doc/current/doctrine/associations.html);
- [Dokumentacja DoctrineMigrationsBundle](https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html).

---

- **Poprzednia strona:** [Konfigurowanie bazy danych](7-database.md)
- **Następna strona:** [Konfigurowanie zaplecza administratora](9-backend.md)
