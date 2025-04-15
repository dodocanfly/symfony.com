## Nasłuchiwacze encji (Entity Listeners)

Nasłuchiwacze encji, które są serwisami, muszą zostać zarejestrowane przy użyciu *entity listener resolver*. Oprócz adnotacji/atrybutu w klasie encji, należy oznaczyć serwis tagiem `doctrine.orm.entity_listener`, aby został automatycznie dodany do resolvera. Można (opcjonalnie) użyć atrybutu `entity_manager`, aby określić, z którym *entity managerem* powinien być zarejestrowany.

Pełny przykład:

```php
<?php
// User.php

use Doctrine\ORM\Mapping as ORM;
use App\UserListener;

#[ORM\Entity]
#[ORM\EntityListeners([UserListener::class])]
class User
{
    // ....
}
```

```yaml
services:
    App\UserListener:
        tags:
            # Minimal configuration below
            - { name: doctrine.orm.entity_listener }
            # Or, optionally, you can give the entity manager name as below
            #- { name: doctrine.orm.entity_listener, entity_manager: custom }
```

Począwszy od `doctrine/orm` w wersji 2.5 oraz `DoctrineBundle` w wersji 1.5.2, zamiast rejestrować nasłuchiwacz encji bezpośrednio w encji, możesz zadeklarować wszystkie opcje w definicji serwisu:

```yaml
services:
    App\UserListener:
        tags:
            -
                name: doctrine.orm.entity_listener
                event: preUpdate
                entity: App\Entity\User
                # entity_manager attribute is optional
                entity_manager: custom
                # method attribute is optional
                method: validateEmail
```

Atrybut `event` jest wymagany, jeśli nasłuchiwacz encji nie został zarejestrowany w encji. Jeśli nie podasz atrybutu `method`, zostanie użyta domyślna nazwa zdarzenia, na które nasłuchuje.

Począwszy od `DoctrineBundle` w wersji 1.12, jeśli wskazana metoda nie istnieje, ale nasłuchiwacz encji jest wywoływalny (invokable), zostanie użyta metoda `__invoke()`.

Więcej informacji o nasłuchiwaczach encji i resolverze wymaganym przez Symfony znajdziesz tutaj:
[https://www.doctrine-project.org/projects/doctrine-orm/en/latest/reference/events.html#entity-listeners](https://www.doctrine-project.org/projects/doctrine-orm/en/latest/reference/events.html#entity-listeners)

## Leniwe nasłuchiwacze encji (Lazy Entity Listeners)

Możesz użyć atrybutu `lazy` w tagu, aby upewnić się, że serwisy nasłuchiwaczy są instancjonowane tylko wtedy, gdy są faktycznie używane.

```yaml
services:
    App\UserListener:
        tags:
            - { name: doctrine.orm.entity_listener, lazy: true }
```
