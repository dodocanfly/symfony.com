# Najlepsze praktyki w frameworku Symfony

Ten artykuł opisuje **najlepsze praktyki dotyczące tworzenia aplikacji internetowych z użyciem Symfony**, zgodne z filozofią przyświecającą jego twórcom.

Jeśli nie zgadzasz się z niektórymi z tych zaleceń, potraktuj je jako **punkt wyjścia**, który możesz następnie **dostosować do własnych potrzeb**. Możesz też całkowicie je zignorować i nadal stosować własne dobre praktyki oraz metodyki pracy. Symfony jest na tyle elastyczny, że dopasuje się do Twoich wymagań.

W artykule założono, że masz już doświadczenie w tworzeniu aplikacji z użyciem Symfony. Jeśli nie, najpierw zapoznaj się z sekcją [Pierwsze kroki](https://symfony.com/doc/current/setup.html) w dokumentacji.

> [!TIP]
> Symfony udostępnia przykładową aplikację o nazwie [Symfony Demo](https://github.com/symfony/demo), która stosuje wszystkie te najlepsze praktyki – dzięki niej możesz zobaczyć je w praktyce.

## Tworzenie projektu

### Używanie narzędzia Symfony Binary do tworzenia aplikacji Symfony

Symfony Binary to plik wykonywalny, który pojawia się na Twoim komputerze po [pobraniu Symfony](https://symfony.com/download). Udostępnia on wiele przydatnych funkcji, w tym najprostszy sposób tworzenia nowych aplikacji Symfony:

```bash
symfony new my_project_directory
```

W tle, polecenie to uruchamia odpowiednie polecenie [Composera](https://getcomposer.org), które [tworzy nową aplikację Symfony](https://symfony.com/doc/current/setup.html#creating-symfony-applications) opartą na aktualnej stabilnej wersji.

### Używaj domyślnej struktury katalogów

Jeśli Twój projekt nie wymaga stosowania określonej struktury katalogów wynikającej z przyjętej metodyki pracy, korzystaj z domyślnej struktury katalogów Symfony. Jest ona płaska, intuicyjna i niezależna od samego Symfony.

```
your_project/
├─ assets/
├─ bin/
│  └─ console
├─ config/
│  ├─ packages/
│  ├─ routes/
│  └─ services.yaml
├─ migrations/
├─ public/
│  ├─ build/
│  └─ index.php
├─ src/
│  ├─ Kernel.php
│  ├─ Command/
│  ├─ Controller/
│  ├─ DataFixtures/
│  ├─ Entity/
│  ├─ EventSubscriber/
│  ├─ Form/
│  ├─ Repository/
│  ├─ Security/
│  └─ Twig/
├─ templates/
├─ tests/
├─ translations/
├─ var/
│  ├─ cache/
│  └─ log/
└─ vendor/
```

## Konfiguracja

### Używaj zmiennych środowiskowych do konfiguracji infrastruktury

Wartości tych opcji różnią się w zależności od maszyny (np. między środowiskiem deweloperskim a serwerem produkcyjnym), ale nie wpływają na logikę działania aplikacji.

[Używaj zmiennych środowiskowych w swoim projekcie](https://symfony.com/doc/current/configuration.html#config-env-vars) do definiowania tych opcji i twórz różne pliki `.env`, aby [konfigurować zmienne środowiskowe dla poszczególnych środowisk](https://symfony.com/doc/current/configuration.html#config-dot-env).

### Używaj tajnych danych (Secrets) do przechowywania informacji wrażliwych

Jeśli Twoja aplikacja zawiera wrażliwą konfigurację, taką jak klucz API, powinieneś przechowywać te dane w bezpieczny sposób, korzystając z [systemu zarządzania sekretami w Symfony](https://symfony.com/doc/current/configuration/secrets.html).

### Używaj parametrów do konfiguracji aplikacji

Są to opcje, które wpływają na działanie aplikacji, np. adres nadawcy powiadomień e-mail czy [przełączniki włączające określone funkcje](https://en.wikipedia.org/wiki/Feature_toggle). Ich wartość nie zmienia się w zależności od maszyny, dlatego nie należy definiować ich jako zmienne środowiskowe.

Zamiast tego zdefiniuj te opcje jako [parametry](https://symfony.com/doc/current/configuration.html#configuration-parameters) w pliku `config/services.yaml`. Możesz je [nadpisywać](https://symfony.com/doc/current/configuration.html#configuration-environments) w zależności od środowiska w plikach `config/services_dev.yaml` i `config/services_prod.yaml`.

O ile dana konfiguracja nie jest wykorzystywana wielokrotnie i nie wymaga ścisłej walidacji, *nie* używaj [komponentu Config](https://symfony.com/doc/current/components/config.html) do definiowania tych opcji.

### Używaj krótkich i poprzedzonych prefiksem nazw parametrów

Rozważ stosowanie prefiksu `app`. dla swoich [parametrów](https://symfony.com/doc/current/configuration.html#configuration-parameters), aby uniknąć kolizji z parametrami Symfony oraz pakietów/bibliotek firm trzecich. Następnie użyj jednego lub dwóch słów, aby opisać przeznaczenie parametru:

```yaml
# config/services.yaml
parameters:
    # don't do this: 'dir' is too generic, and it doesn't convey any meaning
    app.dir: '...'
    # do this: short but easy to understand names
    app.contents_dir: '...'
    # it's OK to use dots, underscores, dashes or nothing, but always
    # be consistent and use the same format for all the parameters
    app.dir.contents: '...'
    app.contents-dir: '...'
```

### Używaj stałych do definiowania opcji, które rzadko się zmieniają

Opcje konfiguracyjne, takie jak liczba elementów wyświetlanych na liście, zazwyczaj rzadko ulegają zmianie. Zamiast definiować je jako [parametry konfiguracyjne](https://symfony.com/doc/current/configuration.html#configuration-parameters), zdefiniuj je jako stałe PHP w odpowiednich klasach. Przykład:

```php
// src/Entity/Post.php
namespace App\Entity;

class Post
{
    public const NUMBER_OF_ITEMS = 10;

    // ...
}
```

Główną zaletą stałych jest to, że możesz używać ich wszędzie – także w szablonach Twig i encjach Doctrine – podczas gdy parametry są dostępne tylko w miejscach mających dostęp do [kontenera usług](https://symfony.com/doc/current/service_container.html).

Jedyną istotną wadą używania stałych do tego typu wartości konfiguracyjnych jest to, że ich nadpisywanie w testach jest bardziej skomplikowane.

## Logika biznesowa

### Nie twórz żadnych bundli do organizowania logiki aplikacji

Gdy wydano Symfony 2.0, aplikacje używały [bundli](https://symfony.com/doc/current/bundles.html) do podziału kodu na logiczne funkcje, np. UserBundle, ProductBundle, InvoiceBundle itd. Jednak bundle powinien być tworzony jako element, który można ponownie wykorzystać jako samodzielny komponent oprogramowania.

Jeśli potrzebujesz ponownie wykorzystać jakąś funkcjonalność w swoich projektach, stwórz dla niej osobny bundle (najlepiej w prywatnym repozytorium — nie udostępniaj go publicznie). W pozostałej części kodu aplikacji zamiast bundli używaj przestrzeni nazw (namespaces) w PHP do organizacji kodu.

### Używaj autowiringu do automatycznej konfiguracji usług aplikacji

[Autowiring](https://symfony.com/doc/current/service_container/autowiring.html) to funkcja, która odczytuje deklaracje typów (type-hinty) w konstruktorach (lub innych metodach) i automatycznie przekazuje odpowiednie usługi do tych metod. Dzięki temu nie trzeba ręcznie konfigurować usług, co upraszcza utrzymanie aplikacji.

Stosuj autowiring w połączeniu z [autokonfiguracją](https://symfony.com/doc/current/service_container.html#services-autoconfigure), aby automatycznie dodawać [tagi](https://symfony.com/doc/current/service_container/tags.html) do usług, które ich wymagają — na przykład rozszerzenia Twig, subskrybenci zdarzeń itp.

### Usługi powinny być prywatne, gdy tylko to możliwe

[Ustaw usługi jako prywatne](https://symfony.com/doc/current/service_container.html#container-public), aby uniemożliwić dostęp do nich za pomocą `$container->get()`. Zamiast tego będziesz musiał używać odpowiedniej wstrzykiwania zależności (dependency injection).

### Używaj formatu YAML do konfigurowania własnych usług

Jeśli używasz [domyślnej konfiguracji `services.yaml`](https://symfony.com/doc/current/service_container.html#service-container-services-load-example), większość usług zostanie skonfigurowana automatycznie. Jednak w niektórych przypadkach brzegowych będziesz musiał skonfigurować usługi (lub ich części) ręcznie.

YAML jest rekomendowanym formatem do konfiguracji usług, ponieważ jest przyjazny dla nowicjuszy i zwięzły, ale Symfony obsługuje również konfigurację w formacie XML i PHP.

### Używaj atrybutów do definiowania mapowania encji Doctrine

Encje Doctrine to zwykłe obiekty PHP, które przechowujesz w jakiejś "bazie danych". Doctrine zna Twoje encje tylko dzięki metadanym mapowania skonfigurowanym dla Twoich klas modeli.

Doctrine obsługuje kilka formatów metadanych, ale zaleca się używanie atrybutów PHP, ponieważ są one zdecydowanie najwygodniejszym i najbardziej elastycznym sposobem ustawiania i wyszukiwania informacji o mapowaniu.

```php
#[ORM\Entity]
class Product
{
    #[ORM\Id]
    #[ORM\Column]
    private int $id;
}
```

## Kontrolery

### Spraw, aby Twój kontroler dziedziczył po klasie AbstractController

Symfony oferuje [bazowy kontroler](https://symfony.com/doc/current/controller.html#the-base-controller-classes-services), który zawiera skróty dla najczęstszych potrzeb, takich jak renderowanie szablonów czy sprawdzanie uprawnień bezpieczeństwa.

Rozszerzanie kontrolerów o ten bazowy kontroler łączy Twoją aplikację z Symfony. Ogólnie rzecz biorąc, łączenie (coupling) jest niewłaściwe, ale w tym przypadku może być dopuszczalne, ponieważ kontrolery nie powinny zawierać logiki biznesowej. Kontrolery powinny zawierać tylko kilka linii *kodu łączącego*, dzięki czemu nie łączysz istotnych części swojej aplikacji.

### Używaj atrybutów do konfigurowania routingu, cache'owania i bezpieczeństwa

Używanie atrybutów do routingu, cache'owania i bezpieczeństwa upraszcza konfigurację. Nie musisz przeglądać kilku plików stworzonych w różnych formatach (YAML, XML, PHP): cała konfiguracja jest dokładnie tam, gdzie jej potrzebujesz, i używa tylko jednego formatu.

```php
#[Route('/profile', name: 'user_profile')]
#[IsGranted('ROLE_USER')]
public function profile(): Response
```

### Używaj wstrzykiwania zależności do pobierania usług

Jeśli rozszerzasz bazowy kontroler `AbstractController`, możesz uzyskać dostęp tylko do najczęstszych usług (np. `twig`, `router`, `doctrine` itp.) bezpośrednio z kontenera za pomocą `$this->container->get()`. Zamiast tego musisz używać wstrzykiwania zależności, aby pobierać usługi, [wskazując je w argumentach metod akcji](https://symfony.com/doc/current/controller.html#controller-accessing-services) lub konstruktorach.

### Używaj resolverów wartości encji, jeśli są wygodne

Jeśli używasz [Doctrine](https://symfony.com/doc/current/doctrine.html), możesz *opcjonalnie* użyć [EntityValueResolver](https://symfony.com/doc/current/doctrine.html#doctrine-entity-value-resolver), aby automatycznie zapytać o encję i przekazać ją jako argument do kontrolera. Wyświetli on również stronę 404, jeśli encja nie zostanie znaleziona.

```php
public function show(Product $product)
```

Jeśli logika pobierania encji z zmiennej trasy jest bardziej skomplikowana, zamiast konfigurować `EntityValueResolver`, lepiej wykonać zapytanie Doctrine bezpośrednio w kontrolerze (np. wywołując [metodę repozytorium Doctrine](https://symfony.com/doc/current/doctrine.html)).

## Szablony

### Używaj notacji snake_case dla nazw szablonów i zmiennych

Używaj małych liter i notacji snake_case dla nazw szablonów, katalogów i zmiennych (np. `user_profile` zamiast `userProfile` i `product/edit_form.html.twig` zamiast `Product/EditForm.html.twig`).

### Prefiksuj fragmenty szablonów podkreśleniem

Fragmenty szablonów, zwane również *"częściowymi szablonami"*, umożliwiają [ponowne użycie treści szablonów](https://symfony.com/doc/current/templates.html#templates-reuse-contents). Prefiksuj ich nazwy podkreśleniem, aby lepiej rozróżnić je od pełnych szablonów (np. `_user_metadata.html.twig` lub `_caution_message.html.twig`).

## Formularze

### Definiuj swoje formularze jako klasy PHP

Tworzenie [formularzy w klasach](https://symfony.com/doc/current/forms.html#creating-forms-in-classes) umożliwia ich ponowne użycie w różnych częściach aplikacji. Ponadto, nie tworzenie formularzy w kontrolerach upraszcza kod i ułatwia utrzymanie kontrolerów.

### Dodaj przyciski formularzy w szablonach

Klasy formularzy powinny być niezależne od miejsca, w którym będą używane. Na przykład, przycisk formularza służącego do tworzenia i edytowania przedmiotów powinien zmieniać się z "Dodaj nowy" na "Zapisz zmiany", w zależności od miejsca, w którym jest używany.

Zamiast dodawać przyciski w klasach formularzy lub kontrolerach, zaleca się dodanie przycisków w szablonach. Poprawia to również separację odpowiedzialności, ponieważ stylowanie przycisków (klasa CSS i inne atrybuty) jest definiowane w szablonie, a nie w klasie PHP.

Jednak jeśli tworzysz [formularz z wieloma przyciskami zatwierdzającymi](https://symfony.com/doc/current/form/multiple_buttons.html), powinieneś je zdefiniować w kontrolerze, a nie w szablonie. W przeciwnym razie nie będziesz w stanie sprawdzić, który przycisk został kliknięty podczas obsługi formularza w kontrolerze.

### Definiuj ograniczenia walidacji na obiekcie bazowym

Dołączanie [ograniczeń walidacji](https://symfony.com/doc/current/reference/constraints.html) do pól formularza, a nie do mapowanego obiektu, uniemożliwia ponowne użycie walidacji w innych formularzach lub innych miejscach, gdzie obiekt jest używany.

### Używaj jednej akcji do renderowania i przetwarzania formularza

[Renderowanie formularzy](https://symfony.com/doc/current/forms.html#rendering-forms) i [ich przetwarzanie](https://symfony.com/doc/current/forms.html#processing-forms) to dwie główne czynności podczas obsługi formularzy. Obie są zbyt podobne (w większości przypadków prawie identyczne), więc znacznie prostsze jest pozwolenie, by pojedyncza akcja kontrolera obsługiwała obie te czynności.

## Międzynarodowość

### Używaj formatu XLIFF do swoich plików tłumaczeń

Spośród wszystkich formatów tłumaczeń obsługiwanych przez Symfony (PHP, Qt, `.po`, `.mo`, JSON, CSV, INI itp.), `XLIFF` i `gettext` mają najlepsze wsparcie w narzędziach używanych przez profesjonalnych tłumaczy. Ponieważ XLIFF jest oparty na XML, możesz walidować zawartość plików `XLIFF` w trakcie ich pisania.

Symfony obsługuje również notatki w plikach XLIFF, co czyni je bardziej przyjaznymi dla tłumaczy. Na końcu dobre tłumaczenia opierają się na kontekście, a te notatki XLIFF pozwalają zdefiniować ten kontekst.

### Używaj kluczy do tłumaczeń zamiast ciągów tekstowych

Używanie kluczy upraszcza zarządzanie plikami tłumaczeń, ponieważ możesz zmieniać oryginalne treści w szablonach, kontrolerach i usługach bez konieczności aktualizowania wszystkich plików tłumaczeń.

Klucze powinny zawsze opisywać swoje przeznaczenie, a nie lokalizację. Na przykład, jeśli formularz ma pole z etykietą "Nazwa użytkownika", odpowiedni klucz to `label.username`, a *nie* `edit_form.label.username`.

## Bezpieczeństwo

### Definiuj pojedynczy firewall

Jeśli nie masz dwóch odrębnych systemów uwierzytelniania i użytkowników (np. logowanie formularzowe dla głównej witryny i system tokenów tylko dla API), zaleca się posiadanie tylko jednego firewall'a, aby uprościć konfigurację.

Dodatkowo, powinieneś używać klucza `anonymous` w ramach firewall'a. Jeśli wymagasz, aby użytkownicy byli zalogowani w różnych sekcjach swojej witryny, użyj opcji [access_control](https://symfony.com/doc/current/security/access_control.html).

### Używaj `auto` Password Hasher

[Auto password hasher](https://symfony.com/doc/current/security/passwords.html#reference-security-encoder-auto) automatycznie wybiera najlepszy możliwy encoder/hasher w zależności od Twojej instalacji PHP. Aktualnie domyślnym auto hasherem jest bcrypt.

### Używaj Voters do implementacji precyzyjnych ograniczeń bezpieczeństwa

Jeśli logika bezpieczeństwa jest skomplikowana, powinieneś stworzyć niestandardowe [security voters](https://symfony.com/doc/current/security/voters.html), zamiast definiować długie wyrażenia wewnątrz atrybutu `#[Security]`.

## Zasoby internetowe

### Używaj AssetMapper do zarządzania zasobami internetowymi

Zasoby internetowe to pliki CSS, JavaScript i obrazy, które sprawiają, że frontend Twojej witryny wygląda i działa świetnie. [AssetMapper](https://symfony.com/doc/current/frontend/asset_mapper.html) pozwala pisać nowoczesny JavaScript i CSS bez potrzeby korzystania z bundlera, takiego jak [Webpack](https://webpack.js.org/) (bezpośrednio lub za pośrednictwem [Webpack Encore](https://symfony.com/doc/current/frontend/encore/index.html)).

## Testy

### Przeprowadź test dymny swoich URL-i

W inżynierii oprogramowania [test dymny](https://en.wikipedia.org/wiki/Smoke_testing_(software)) polega na *"wstępnym testowaniu w celu wykrycia prostych awarii wystarczająco poważnych, by odrzucić przyszłą wersję oprogramowania"*. Korzystając z [dostawców danych PHPUnit](https://docs.phpunit.de/en/9.6/writing-tests-for-phpunit.html#data-providers), możesz zdefiniować test funkcjonalny, który sprawdza, czy wszystkie URL-e aplikacji ładują się poprawnie:

```php
// tests/ApplicationAvailabilityFunctionalTest.php
namespace App\Tests;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

class ApplicationAvailabilityFunctionalTest extends WebTestCase
{
    /**
     * @dataProvider urlProvider
     */
    public function testPageIsSuccessful($url): void
    {
        $client = self::createClient();
        $client->request('GET', $url);

        $this->assertResponseIsSuccessful();
    }

    public function urlProvider(): \Generator
    {
        yield ['/'];
        yield ['/posts'];
        yield ['/post/fixture-post-1'];
        yield ['/blog/category/fixture-category'];
        yield ['/archives'];
        // ...
    }
}
```

Dodaj ten test podczas tworzenia aplikacji, ponieważ wymaga on niewielkiego wysiłku, a sprawdza, czy żadna z Twoich stron nie zwraca błędu. Później dodasz bardziej szczegółowe testy dla każdej strony.

### Hard-code'uj URL-e w teście funkcjonalnym

W aplikacjach Symfony zaleca się [generowanie URL-i](https://symfony.com/doc/current/routing.html#routing-generating-urls) za pomocą tras, aby automatycznie aktualizować wszystkie linki, gdy URL ulegnie zmianie. Jednak jeśli publiczny URL się zmieni, użytkownicy nie będą w stanie go przeglądać, chyba że skonfigurujesz przekierowanie na nowy URL.

Dlatego zaleca się używanie surowych URL-i w testach zamiast generowania ich z tras. Kiedy trasa się zmieni, testy zakończą się niepowodzeniem, a Ty będziesz wiedział, że musisz ustawić przekierowanie.
