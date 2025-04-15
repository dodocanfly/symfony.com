# Komponent DependencyInjection

> [!NOTE]
> Komponent DependencyInjection implementuje kompatybilny z [PSR-11](https://www.php-fig.org/psr/psr-11) kontener serwisów, który umożliwia standaryzację i centralizację sposobu tworzenia obiektów w Twojej aplikacji.

Wprowadzenie do wstrzykiwania zależności (Dependency Injection) i kontenerów serwisów znajdziesz w artykule [Service Container](../service_container.md).

## Instalacja

```bash
composer require symfony/dependency-injection
```

> [!NOTE]
> Jeśli instalujesz ten komponent poza aplikacją Symfony, musisz dołączyć plik `vendor/autoload.php` w swoim kodzie, aby włączyć mechanizm automatycznego ładowania klas udostępniany przez Composera. Więcej szczegółów znajdziesz w [tym artykule](using_components.md).

## Podstawowe użycie

> [!NOTE]
> Ten artykuł wyjaśnia, jak używać możliwości komponentu DependencyInjection jako niezależnego elementu w dowolnej aplikacji PHP. Artykuł [Service Container](../service_container.md) zawiera informacje o tym, jak używać go w aplikacjach Symfony.

Możesz mieć klasę taką jak poniższa `Mailer`, którą chcesz udostępnić jako serwis:

```php
class Mailer
{
    private string $transport;

    public function __construct()
    {
        $this->transport = 'sendmail';
    }

    // ...
}
```

Możesz zarejestrować ją w kontenerze jako serwis:

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;

$container = new ContainerBuilder();
$container->register('mailer', 'Mailer');
```

Ulepszeniem tej klasy, które uczyni ją bardziej elastyczną, byłoby umożliwienie kontenerowi ustawienia używanego transportu. Jeśli zmienisz klasę tak, by transport był przekazywany do konstruktora:

```php
class Mailer
{
    public function __construct(
        private string $transport,
    ) {
    }

    // ...
}
```

wtedy możesz ustawić wybór transportu w kontenerze:

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;

$container = new ContainerBuilder();
$container
    ->register('mailer', 'Mailer')
    ->addArgument('sendmail');
```

Klasa ta jest teraz znacznie bardziej elastyczna, ponieważ oddzielono wybór transportu od implementacji i przeniesiono go do kontenera.

Wybrany transport może być czymś, o czym muszą wiedzieć inne serwisy. Możesz uniknąć konieczności zmiany tego w wielu miejscach, definiując to jako parametr w kontenerze i odwołując się do niego jako do argumentu konstruktora serwisu `Mailer`:

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;

$container = new ContainerBuilder();
$container->setParameter('mailer.transport', 'sendmail');
$container
    ->register('mailer', 'Mailer')
    ->addArgument('%mailer.transport%');
```

Teraz, gdy serwis `mailer` znajduje się w kontenerze, możesz wstrzykiwać go jako zależność do innych klas. Jeśli masz klasę `NewsletterManager` taką jak ta:

```php
class NewsletterManager
{
    public function __construct(
        private \Mailer $mailer,
    ) {
    }

    // ...
}
```

Podczas definiowania serwisu `newsletter_manager` serwis `mailer` może jeszcze nie istnieć. Użyj klasy `Reference`, aby powiedzieć kontenerowi, że ma wstrzyknąć serwis `mailer` przy inicjalizacji menedżera newsletterów:

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Reference;

$container = new ContainerBuilder();

$container->setParameter('mailer.transport', 'sendmail');
$container
    ->register('mailer', 'Mailer')
    ->addArgument('%mailer.transport%');

$container
    ->register('newsletter_manager', 'NewsletterManager')
    ->addArgument(new Reference('mailer'));
```

Jeśli `NewsletterManager` nie wymagałby `Mailer`, a jego wstrzyknięcie byłoby tylko opcjonalne, można by zastosować wstrzykiwanie przez setter:

```php
class NewsletterManager
{
    private \Mailer $mailer;

    public function setMailer(\Mailer $mailer): void
    {
        $this->mailer = $mailer;
    }

    // ...
}
```

Możesz teraz zdecydować, że nie wstrzykniesz `Mailer` do `NewsletterManager`. Jeśli jednak chcesz to zrobić, kontener może wywołać metodę setter:

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Reference;

$container = new ContainerBuilder();

$container->setParameter('mailer.transport', 'sendmail');
$container
    ->register('mailer', 'Mailer')
    ->addArgument('%mailer.transport%');

$container
    ->register('newsletter_manager', 'NewsletterManager')
    ->addMethodCall('setMailer', [new Reference('mailer')]);
```

Serwis `newsletter_manager` możesz pobrać z kontenera w następujący sposób:

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;

$container = new ContainerBuilder();

// ...

$newsletterManager = $container->get('newsletter_manager');
```

### Pobieranie nieistniejących serwisów

Domyślnie, gdy próbujesz pobrać serwis, który nie istnieje, zostanie wyrzucony wyjątek. Możesz jednak nadpisać to zachowanie w następujący sposób:

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\ContainerInterface;

$containerBuilder = new ContainerBuilder();

// ...

// the second argument is optional and defines what to do when the service doesn't exist
$newsletterManager = $containerBuilder->get('newsletter_manager', ContainerInterface::EXCEPTION_ON_INVALID_REFERENCE);
```

Oto wszystkie możliwe zachowania:
- `ContainerInterface::EXCEPTION_ON_INVALID_REFERENCE`: wyrzuca wyjątek w czasie kompilacji (domyślne zachowanie);
- `ContainerInterface::RUNTIME_EXCEPTION_ON_INVALID_REFERENCE`: wyrzuca wyjątek w czasie wykonania, przy próbie uzyskania dostępu do brakującego serwisu;
- `ContainerInterface::NULL_ON_INVALID_REFERENCE`: zwraca `null`;
- `ContainerInterface::IGNORE_ON_INVALID_REFERENCE`: ignoruje wywołanie odwołujące się do danego serwisu (np. pomija setter, jeśli serwis nie istnieje);
- `ContainerInterface::IGNORE_ON_UNINITIALIZED_REFERENCE`: ignoruje/zwraca `null` w przypadku niezinicjalizowanych serwisów lub nieprawidłowych referencji.

## Unikanie uzależnienia kodu od kontenera

Chociaż możesz bezpośrednio pobierać serwisy z kontenera, najlepiej ograniczyć to do minimum. Na przykład, w klasie `NewsletterManager` wstrzyknąłeś serwis `mailer` zamiast pobierać go bezpośrednio z kontenera. Można by było wstrzyknąć cały kontener i pobrać z niego serwis `mailer`, ale wtedy klasa stałaby się zależna od konkretnego kontenera, co utrudniłoby jej ponowne użycie w innym kontekście.

W pewnym momencie i tak będziesz musiał pobrać serwis z kontenera, ale powinno to mieć miejsce jak najrzadziej i jedynie w punktach wejścia Twojej aplikacji.

## Konfigurowanie kontenera za pomocą plików konfiguracyjnych

Oprócz definiowania serwisów w PHP, jak pokazano wcześniej, możesz również używać plików konfiguracyjnych. Dzięki temu możesz korzystać z XML lub YAML do definiowania serwisów, zamiast robić to w czystym PHP. W przypadku aplikacji większych niż najprostsze, sensowne jest uporządkowanie definicji serwisów poprzez przeniesienie ich do jednego lub kilku plików konfiguracyjnych. Aby to umożliwić, musisz zainstalować również [komponent Config](config.md).

Wczytywanie pliku konfiguracyjnego XML:

```php
use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\XmlFileLoader;

$container = new ContainerBuilder();
$loader = new XmlFileLoader($container, new FileLocator(__DIR__));
$loader->load('services.xml');
```

Wczytywanie pliku konfiguracyjnego YAML:

```php
use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;

$container = new ContainerBuilder();
$loader = new YamlFileLoader($container, new FileLocator(__DIR__));
$loader->load('services.yaml');
```

> [!NOTE]
> Jeśli chcesz korzystać z plików YAML, musisz także zainstalować [komponent Yaml](yaml.html).

> [!TIP]
> Jeśli Twoja aplikacja używa niestandardowych rozszerzeń plików (np. pliki XML mają rozszerzenie `.config`), możesz przekazać typ pliku jako drugi, opcjonalny parametr metody `load()`:
> ```php
> // ...
> $loader->load('services.config', 'xml');
> ```

Jeśli chcesz korzystać z PHP do definiowania serwisów, możesz przenieść te definicje do osobnego pliku konfiguracyjnego i wczytać go w podobny sposób:

```php
use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\PhpFileLoader;

$container = new ContainerBuilder();
$loader = new PhpFileLoader($container, new FileLocator(__DIR__));
$loader->load('services.php');
```

Możesz teraz skonfigurować serwisy `newsletter_manager` oraz `mailer` przy użyciu plików konfiguracyjnych:
```yaml
parameters:
    # ...
    mailer.transport: sendmail

services:
    mailer:
        class:     Mailer
        arguments: ['%mailer.transport%']
    newsletter_manager:
        class:     NewsletterManager
        calls:
            - [setMailer, ['@mailer']]
```

## Dowiedz się więcej: [link](https://symfony.com/doc/current/components/dependency_injection.html#learn-more)
- Kompilowanie kontenera
- Przebieg budowania kontenera
- Jak tworzyć aliasy serwisów i oznaczać serwisy jako prywatne
- Automatyczne definiowanie zależności serwisów (Autowiring)
- Wywoływanie metod serwisów i wstrzykiwanie przez settery
- Jak pracować z przepustkami kompilatora (Compiler Passes)
- Jak konfigurować serwis za pomocą konfiguratora
- Jak debugować kontener serwisów i wyświetlać listę serwisów
- Jak pracować z obiektami definicji serwisów
- Jak wstrzykiwać wartości na podstawie złożonych wyrażeń
- Używanie fabryk do tworzenia serwisów
- Jak importować pliki/zasoby konfiguracyjne
- Typy wstrzykiwania zależności
- Serwisy ładowane „leniwe” (Lazy Services)
- Jak uczynić argumenty/referencje serwisów opcjonalnymi
- Jak zarządzać wspólnymi zależnościami za pomocą serwisów nadrzędnych
- Jak pobrać obiekt Request z kontenera serwisów
- Zamknięcia serwisowe (Service Closures)
- Jak dekorować serwisy
- Subskrybenci i lokalizatory serwisów
- Jak definiować serwisy nieshared (tworzone za każdym razem)
- Jak wstrzykiwać instancje do kontenera
- Jak pracować z tagami serwisów
