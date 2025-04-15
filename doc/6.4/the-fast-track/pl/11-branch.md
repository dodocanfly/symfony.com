## Rozgałęzianie kodu

Istnieje wiele sposobów organizowania przepływu pracy przy wprowadzaniu zmian w kodzie projektu. Jednak praca bezpośrednio na głównym branchu Git i wdrażanie na produkcję bez testów prawdopodobnie nie jest najlepszym rozwiązaniem.

Testowanie to nie tylko testy jednostkowe czy funkcjonalne, ale także sprawdzanie zachowania aplikacji z danymi produkcyjnymi. Jeśli Ty lub Twoi [interesariusze](https://pl.wikipedia.org/wiki/Interesariusz) możecie przeglądać aplikację dokładnie tak, jak zostanie ona wdrożona dla użytkowników końcowych, staje się to ogromną zaletą i pozwala na wdrażanie z pewnością. Jest to szczególnie potężne, gdy osoby nietechniczne mogą weryfikować nowe funkcje.

W kolejnych krokach będziemy kontynuować wszystkie prace na głównym branchu Git dla uproszczenia i uniknięcia powtarzania, ale zobaczmy, jak mogłoby to działać lepiej.

### Przyjęcie Git Workflow

Jednym z możliwych sposobów pracy jest tworzenie jednej gałęzi (brancha) dla każdej nowej funkcji lub poprawki błędu. Jest to proste i skuteczne rozwiązanie.

### Tworzenie gałęzi

Workflow rozpoczyna się od utworzenia gałęzi w Git:

```bash
git checkout -b sessions-in-db
```

To polecenie tworzy gałąź `sessions-in-db` na podstawie gałęzi `master` (`main`). „Rozgałęzia” ono kod oraz konfigurację infrastruktury.

### Przechowywanie sesji w bazie danych

Jak można się domyślić z nazwy gałęzi, chcemy zmienić sposób przechowywania sesji z systemu plików na bazę danych (w naszym przypadku PostgreSQL).

Kroki potrzebne do realizacji tego celu są typowe:

1. Utwórz gałąź Git;
2. Zaktualizuj konfigurację Symfony, jeśli to konieczne;
3. Napisz lub zaktualizuj kod, jeśli to konieczne;
4. Zaktualizuj konfigurację PHP, jeśli to konieczne (np. dodaj rozszerzenie PostgreSQL dla PHP);
5. Zaktualizuj infrastrukturę w Dockerze i Platform.sh, jeśli to konieczne (dodaj usługę PostgreSQL);
6. Przetestuj lokalnie;
7. Przetestuj zdalnie;
8. Scal gałąź z masterem;
9. Wdróż na produkcję;
10. Usuń gałąź.

Aby przechowywać sesje w bazie danych, należy zmienić ustawienie `session.handler_id`, aby wskazywało na DSN bazy danych:

```diff
--- a/config/packages/framework.yaml
+++ b/config/packages/framework.yaml
@@ -9,7 +9,7 @@ framework:
     # Enables session support. Note that the session will ONLY be started if you read or write from it.
     # Remove or comment this section to explicitly disable session support.
     session:
-        handler_id: null
+        handler_id: '%env(resolve:DATABASE_URL)%'
         cookie_secure: auto
         cookie_samesite: lax
```

Aby sesje były przechowywane w bazie danych, musimy utworzyć tabelę `sessions`. Zrobimy to za pomocą migracji Doctrine:

```bash
symfony console make:migration
```

Następnie wykonaj migrację bazy danych:

```bash
symfony console doctrine:migrations:migrate
```

Przetestuj lokalnie, przeglądając stronę. Ponieważ nie ma żadnych zmian wizualnych i nie korzystamy jeszcze z sesji, wszystko powinno działać tak jak wcześniej.

> [!NOTE]
> Kroki 3 do 5 nie są tu potrzebne, ponieważ używamy już bazy danych jako magazynu sesji, ale rozdział dotyczący używania Redis pokazuje, jak łatwe jest dodanie, przetestowanie i wdrożenie nowej usługi zarówno w Dockerze, jak i Platform.sh.

Zacommituj swoje zmiany w nowej gałęzi:

```bash
git add .
git commit -m 'Configure database sessions'
```

### Wdrażanie gałęzi

Przed wdrożeniem na produkcję powinniśmy przetestować daną gałąź na takiej samej infrastrukturze jak ta produkcyjna. Powinniśmy również upewnić się, że wszystko działa poprawnie w środowisku `prod` Symfony (lokalna strona działała w środowisku `dev` Symfony).

Teraz utwórzmy środowisko *Platform.sh* na podstawie *gałęzi Git*:

```bash
symfony cloud:deploy
```

To polecenie tworzy nowe środowisko w następujący sposób:  
- Gałąź dziedziczy kod i konfigurację infrastruktury z aktualnej gałęzi Git (`sessions-in-db`);  
- Dane pochodzą z gałęzi `master` (czyli produkcyjnej), dzięki czemu tworzona jest spójna migawka wszystkich danych usług, w tym plików (np. przesłanych przez użytkowników) i baz danych;  
- Tworzony jest nowy, dedykowany klaster do wdrożenia kodu, danych i infrastruktury.

Ponieważ proces wdrażania jest taki sam jak na produkcji, migracje bazy danych również zostaną wykonane. To świetny sposób, aby upewnić się, że migracje działają poprawnie z danymi produkcyjnymi.

Środowiska inne niż `master` są bardzo podobne do środowiska `master`, z kilkoma drobnymi różnicami – na przykład domyślnie nie są wysyłane e-maile.

Po zakończeniu wdrożenia otwórz nową gałąź w przeglądarce:

```bash
symfony cloud:url -1
```

Warto zauważyć, że wszystkie polecenia Platform.sh działają na aktualnej gałęzi Git. Powyższe polecenie otwiera wdrożony adres URL dla gałęzi `sessions-in-db`; adres ten będzie wyglądał mniej więcej tak: `https://sessions-in-db-xxx.eu-5.platformsh.site/`.

Przetestuj stronę w tym nowym środowisku – powinieneś zobaczyć wszystkie dane, które zostały utworzone w środowisku `master`.

Jeśli dodasz nowe konferencje w środowisku `master`, nie pojawią się one w środowisku `sessions-in-db` i odwrotnie. Środowiska są niezależne i odizolowane.

Jeśli kod w gałęzi `master` się zmieni, możesz zawsze zrebasować gałąź Git i wdrożyć zaktualizowaną wersję, rozwiązując konflikty zarówno w kodzie, jak i w konfiguracji infrastruktury.

Możesz nawet zsynchronizować dane ze środowiska `master` z powrotem do środowiska `sessions-in-db`:

```bash
symfony cloud:env:sync
```

### Debugowanie wdrożeń produkcyjnych przed właściwym wdrożeniem

Domyślnie wszystkie środowiska Platform.sh korzystają z tych samych ustawień co środowisko `master`/`prod` (czyli produkcyjne środowisko Symfony). Dzięki temu możesz testować aplikację w warunkach zbliżonych do rzeczywistych. To daje wrażenie, że tworzysz i testujesz bezpośrednio na serwerach produkcyjnych, ale bez ryzyka, jakie się z tym zwykle wiąże. Przypomina to stare dobre czasy, kiedy wdrażało się aplikacje przez FTP.

W razie problemów możesz przełączyć się na środowisko deweloperskie (`dev`) Symfony:

```bash
symfony cloud:env:debug
```

Gdy skończysz, wróć do ustawień produkcyjnych:

```bash
symfony cloud:env:debug --off
```

> [!WARNING]
> **Nigdy nie włączaj środowiska deweloperskiego ani Symfony Profiler na gałęzi `master`** – spowoduje to znaczne spowolnienie aplikacji i otworzy wiele poważnych luk bezpieczeństwa.

### Testowanie wdrożeń produkcyjnych przed ich wykonaniem

Dostęp do nadchodzącej wersji strony internetowej z danymi produkcyjnymi otwiera wiele możliwości — od testów regresji wizualnej po testy wydajnościowe. [Blackfire](https://blackfire.io) to idealne narzędzie do tego celu.

Zajrzyj do kroku dotyczącego [wydajności](29-performance.md), aby dowiedzieć się więcej o tym, jak możesz użyć Blackfire do testowania swojego kodu przed wdrożeniem.

### Scalanie do produkcji

Gdy jesteś zadowolony ze zmian w gałęzi, scal kod oraz konfigurację infrastruktury z powrotem do gałęzi `master`:

```bash
git checkout master
git merge sessions-in-db
```

I wdrożenie:

```bash
symfony cloud:deploy
```

Podczas wdrażania na Platform.sh przesyłane są tylko zmiany w kodzie i infrastrukturze — dane nie są w żaden sposób modyfikowane.

### Czyszczenie

Na koniec uporządkuj wszystko, usuwając gałąź Git oraz środowisko Platform.sh:

```bash
git branch -d sessions-in-db
symfony cloud:env:delete -e sessions-in-db
```

### Sprawdź również:
- [Gałęzie Gita](https://git-scm.com/book/pl/v2/Ga%c5%82%c4%99zie-Gita-Czym-jest-ga%c5%82%c4%85%c5%ba).

---

- **Poprzednia strona:** [Budowanie interfejsu użytkownika](10-twig.md)
- **Następna strona:** [Nasłuchiwanie zdarzeń](12-event.md)
