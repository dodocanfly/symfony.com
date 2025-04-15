# Komponent DependencyInjection

> [!NOTE]
> Komponent DependencyInjection implementuje kompatybilny z [PSR-11](https://www.php-fig.org/psr/psr-11) kontener serwisów, który umożliwia standaryzację i centralizację sposobu tworzenia obiektów w Twojej aplikacji.

Wprowadzenie do wstrzykiwania zależności (Dependency Injection) i kontenerów serwisów znajdziesz w artykule [Service Container](doc/service_container.md).

## Instalacja

```bash
composer require symfony/dependency-injection
```

> [!NOTE]
> Jeśli instalujesz ten komponent poza aplikacją Symfony, musisz dołączyć plik `vendor/autoload.php` w swoim kodzie, aby włączyć mechanizm automatycznego ładowania klas udostępniany przez Composera. Więcej szczegółów znajdziesz w [tym artykule](doc/components/using_components.md).

## Podstawowe użycie

Ten artykuł wyjaśnia, jak używać możliwości komponentu DependencyInjection jako niezależnego elementu w dowolnej aplikacji PHP. Artykuł [Service Container](doc/service_container.md) zawiera informacje o tym, jak używać go w aplikacjach Symfony.

Możesz mieć klasę taką jak poniższa `Mailer`, którą chcesz udostępnić jako serwis.

[...]

Możesz zarejestrować ją w kontenerze jako serwis.

[...]

Ulepszeniem tej klasy, które uczyni ją bardziej elastyczną, byłoby umożliwienie kontenerowi ustawienia używanego transportu. Jeśli zmienisz klasę tak, by transport był przekazywany do konstruktora...

[...]

...wtedy możesz ustawić wybór transportu w kontenerze.

[...]

Klasa ta jest teraz znacznie bardziej elastyczna, ponieważ oddzielono wybór transportu od implementacji i przeniesiono go do kontenera.

Wybrany transport może być czymś, o czym muszą wiedzieć inne serwisy. Możesz uniknąć konieczności zmiany tego w wielu miejscach, definiując to jako parametr w kontenerze i odwołując się do niego jako do argumentu konstruktora serwisu `Mailer`.

[...]

Teraz, gdy serwis `mailer` znajduje się w kontenerze, możesz wstrzykiwać go jako zależność do innych klas. Jeśli masz klasę `NewsletterManager` taką jak ta:

[...]

Podczas definiowania serwisu `newsletter_manager` serwis `mailer` może jeszcze nie istnieć. Użyj klasy `Reference`, aby powiedzieć kontenerowi, że ma wstrzyknąć serwis `mailer` przy inicjalizacji `newsletter_manager`.

[...]

Jeśli `NewsletterManager` nie wymagałby `Mailer`, a jego wstrzyknięcie byłoby tylko opcjonalne, można by zastosować **wstrzykiwanie przez setter**.

[...]

Możesz teraz zdecydować, że nie wstrzykniesz `Mailer` do `NewsletterManager`. Jeśli jednak chcesz to zrobić, kontener może wywołać metodę setter.

[...]

Serwis `newsletter_manager` możesz pobrać z kontenera w następujący sposób:

[...]

---

### Pobieranie nieistniejących serwisów

Domyślnie, gdy próbujesz pobrać serwis, który nie istnieje, zostanie wyrzucony wyjątek. Możesz jednak nadpisać to zachowanie w następujący sposób:

[...]

Oto wszystkie możliwe zachowania:
- `ContainerInterface::EXCEPTION_ON_INVALID_REFERENCE`: wyrzuca wyjątek w czasie kompilacji (domyślne zachowanie);
- `ContainerInterface::RUNTIME_EXCEPTION_ON_INVALID_REFERENCE`: wyrzuca wyjątek w czasie wykonania, przy próbie uzyskania dostępu do brakującego serwisu;
- `ContainerInterface::NULL_ON_INVALID_REFERENCE`: zwraca `null`;
- `ContainerInterface::IGNORE_ON_INVALID_REFERENCE`: ignoruje wywołanie odwołujące się do danego serwisu (np. pomija setter, jeśli serwis nie istnieje);
- `ContainerInterface::IGNORE_ON_UNINITIALIZED_REFERENCE`: ignoruje/zwraca `null` w przypadku niezinicjalizowanych serwisów lub nieprawidłowych referencji.
