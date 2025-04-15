## Zabezpieczanie panelu administracyjnego

Interfejs panelu administracyjnego powinien być dostępny wyłącznie dla zaufanych osób. Zabezpieczenie tej części witryny można zrealizować za pomocą komponentu Symfony Security.

### Tworzenie encji użytkownika

Chociaż użytkownicy nie będą mogli zakładać własnych kont na stronie, stworzymy w pełni funkcjonalny system uwierzytelniania dla administratora. Będziemy zatem mieć tylko jednego użytkownika — administratora strony.

Pierwszym krokiem jest zdefiniowanie encji `User`. Aby uniknąć nieporozumień, nazwijmy ją `Admin`.

Aby zintegrować encję `Admin` z systemem uwierzytelniania Symfony Security, musi ona spełniać określone wymagania. Na przykład, musi zawierać właściwość `password`.

Zamiast używać tradycyjnego polecenia `make:entity`, użyj dedykowanego polecenia `make:user`, aby utworzyć encję `Admin`:

```bash
symfony console make:user Admin
```

Odpowiedz na pytania interaktywne: chcemy użyć Doctrine do przechowywania administratorów (`yes`), używać nazwy użytkownika (`username`) jako unikalnej nazwy wyświetlanej administratorów, oraz każdy użytkownik będzie posiadał hasło (`tak`).

Wygenerowana klasa zawiera takie metody jak `getRoles()`, `eraseCredentials()` i kilka innych, wymaganych przez system uwierzytelniania Symfony.

Jeśli chcesz dodać więcej właściwości do użytkownika `Admin`, użyj komendy `make:entity`.

Oprócz wygenerowania encji `Admin`, komenda ta zaktualizowała również konfigurację bezpieczeństwa, aby powiązać encję z systemem uwierzytelniania:

```diff
--- a/config/packages/security.yaml
+++ b/config/packages/security.yaml
@@ -5,14 +5,18 @@ security:
         Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface: 'auto'
     # https://symfony.com/doc/current/security.html#loading-the-user-the-user-provider
     providers:
-        users_in_memory: { memory: null }
+        # used to reload user from session & other features (e.g. switch_user)
+        app_user_provider:
+            entity:
+                class: App\Entity\Admin
+                property: username
     firewalls:
         dev:
             pattern: ^/(_(profiler|wdt)|css|images|js)/
             security: false
         main:
             lazy: true
-            provider: users_in_memory
+            provider: app_user_provider

             # activate different ways to authenticate
             # https://symfony.com/doc/current/security.html#the-firewall
```

Pozostawiamy Symfony wybór najlepszego dostępnego algorytmu do haszowania haseł (co będzie się zmieniać w czasie).

Czas wygenerować migrację i zastosować ją do bazy danych:

```bash
symfony console make:migration
symfony console doctrine:migrations:migrate -n
```

### Generowanie hasła dla użytkownika Admin

Nie będziemy tworzyć dedykowanego systemu do zakładania kont administratorów. Ponownie – będziemy mieć tylko jednego administratora. Login będzie brzmiał `admin`, a my musimy wygenerować skrót (hash) hasła.

Wybierz dowolne hasło i uruchom poniższe polecenie, aby wygenerować jego hash:

```bash
symfony console security:hash-password
```

```
Symfony Password Hash Utility
=============================

 Type in your password to be hashed:
 >

 ------------------ ---------------------------------------------------------------------------------------------------
  Key                Value
 ------------------ ---------------------------------------------------------------------------------------------------
  Hasher used        Symfony\Component\PasswordHasher\Hasher\MigratingPasswordHasher
  Password hash      $argon2id$v=19$m=65536,t=4,p=1$BQG+jovPcunctc30xG5PxQ$TiGbx451NKdo+g9vLtfkMy4KjASKSOcnNxjij4gTX1s
 ------------------ ---------------------------------------------------------------------------------------------------

 ! [NOTE] Self-salting hasher used: the hasher generated its own built-in salt.


 [OK] Password hashing succeeded
```

### Tworzenie użytkownika Admin

Wstaw użytkownika administratora za pomocą zapytania SQL:

```bash
symfony run psql -c "INSERT INTO admin (id, username, roles, password) \
  VALUES (nextval('admin_id_seq'), 'admin', '[\"ROLE_ADMIN\"]', \
  '\$argon2id\$v=19\$m=65536,t=4,p=1\$BQG+jovPcunctc30xG5PxQ\$TiGbx451NKdo+g9vLtfkMy4KjASKSOcnNxjij4gTX1s')"
```

Zwróć uwagę na znak `$` w haśle — wszystkie trzeba poprzedzić znakiem `\` (escape)!

### Konfigurowanie uwierzytelniania (Authentication) w Symfony

Skoro mamy już użytkownika *admin*, możemy zabezpieczyć panel administracyjny. Symfony obsługuje kilka strategii uwierzytelniania — my skorzystamy z klasycznego, popularnego *logowania przez formularz*.

Uruchom polecenie, które zaktualizuje konfigurację bezpieczeństwa, wygeneruje szablon logowania i utworzy odpowiedni *authenticator*:

```bash
symfony console make:security:form-login
```

Nazwij kontroler `SecurityController` i potwierdź, że chcesz wygenerować URL `/logout` (`tak`).

Polecenie to zaktualizowało konfigurację bezpieczeństwa i powiązało wygenerowane klasy:

```diff
--- a/config/packages/security.yaml
+++ b/config/packages/security.yaml
@@ -15,7 +15,15 @@ security:
             security: false
         main:
             lazy: true
-            provider: users_in_memory
+            provider: app_user_provider
+            form_login:
+                login_path: app_login
+                check_path: app_login
+                enable_csrf: true
+            logout:
+                path: app_logout
+                # where to redirect after logout
+                # target: app_any_route

             # activate different ways to authenticate
             # https://symfony.com/doc/current/security.html#the-firewall
```

> [!TIP]
> Jak zapamiętać, że ścieżka EasyAdmin to `/admin` (zgodnie z konfiguracją w `App\Controller\Admin\DashboardController`)? Nie musisz! Możesz sprawdzić to w pliku, ale szybciej będzie uruchomić polecenie pokazujące powiązania między nazwami i ścieżkami routingu:
> ```bash
> symfony console debug:router
> ```

### Dodawanie reguł kontroli dostępu (Authorization)

System bezpieczeństwa składa się z dwóch części: *uwierzytelniania* (*authentication*) i *autoryzacji* (*authorization*)*. Tworząc użytkownika admina, nadaliśmy mu rolę `ROLE_ADMIN`. Ograniczmy dostęp do sekcji `/admin` tylko dla użytkowników z tą rolą, dodając regułę `access_control`:

```diff
--- a/config/packages/security.yaml
+++ b/config/packages/security.yaml
@@ -34,7 +34,7 @@ security:
     # Easy way to control access for large sections of your site
     # Note: Only the *first* access control that matches will be used
     access_control:
-        # - { path: ^/admin, roles: ROLE_ADMIN }
+        - { path: ^/admin, roles: ROLE_ADMIN }
         # - { path: ^/profile, roles: ROLE_USER }

 when@test:
```

Reguły `access_control` ograniczają dostęp na podstawie wyrażeń regularnych. Przy próbie dostępu do adresu zaczynającego się od `/admin`, Symfony sprawdzi, czy zalogowany użytkownik ma rolę `ROLE_ADMIN`.

### Logowanie przez formularz

Jeśli spróbujesz teraz wejść na panel administracyjny, zostaniesz przekierowany na stronę logowania i poproszony o podanie loginu oraz hasła:

![easy-admin-login](https://symfony.com/doc/6.4en//the-fast-track/_images/easy-admin-login.png)

Zaloguj się używając loginu `admin` oraz hasła w postaci zwykłego tekstu, które wybrałeś wcześniej. Jeśli skopiowałeś dokładnie moje polecenie SQL, hasło to `admin`.

Zwróć uwagę, że EasyAdmin automatycznie rozpoznaje system uwierzytelniania Symfony:

![easy-admin-secured](https://symfony.com/doc/6.4en//the-fast-track/_images/easy-admin-secured.png)

Kliknij link "*Wyloguj się*" (Sign out). Udało się! Masz w pełni zabezpieczony panel administratora.

> [!NOTE]
> Jeśli chcesz stworzyć pełnoprawny system rejestracji z formularzem, zapoznaj się z poleceniem: `symfony console make:registration-form`

### Sprawdź również:
- [Dokumentacja Symfony Security](https://symfony.com/doc/current/security.html);
- [Kurs SymfonyCasts: Security](https://symfonycasts.com/screencast/symfony-security);
- [Jak zbudować formularz logowania w Symfony](https://symfony.com/doc/current/security/form_login_setup.html);
- [Ściąga z bezpieczeństwa Symfony](https://cheatsheetseries.owasp.org/cheatsheets/Symfony_Security_Cheat_Sheet.html) (Symfony Security Cheat Sheet).

---

- **Poprzednia strona:** [Akceptowanie opinii za pomocą formularzy](14-form.md)
- **Następna strona:** [Zapobieganie spamowi za pomocą API](16-spam.md)
