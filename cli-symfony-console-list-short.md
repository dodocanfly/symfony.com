# 📝 Symfony – Najczęstsze komendy (ściągawka)

### 🔹 Ogólne
- `list` – Wszystkie komendy  
- `help <komenda>` – Pomoc dla komendy  
- `about` – Info o projekcie

### 🔹 Cache
- `cache:clear` – Czyść cache  
- `cache:warmup` – Załaduj cache  
- `cache:pool:clear` – Czyść wybraną pulę

### 🔹 Debugowanie
- `debug:router` – Lista tras  
- `debug:container` – Lista serwisów  
- `debug:event-dispatcher` – Eventy & listenery  
- `debug:twig` – Info o Twig  
- `debug:config <nazwa>` – Konfiguracja bundla  

### 🔹 Doctrine (baza danych)
- `doctrine:database:create|drop` – Tworzenie/usuwanie bazy  
- `doctrine:schema:update --force` – Aktualizacja schematu  
- `doctrine:migrations:migrate` – Migracja bazy  
- `doctrine:migrations:diff` – Wygeneruj migrację  

### 🔹 MakerBundle (generatory)
- `make:controller` – Nowy kontroler  
- `make:entity` – Nowa encja  
- `make:migration` – Nowa migracja  
- `make:crud` – CRUD dla encji  
- `make:form` – Formularz  
- `make:user` – Klasa użytkownika  

### 🔹 Inne
- `router:match /url` – Jak dopasuje się trasa  
- `security:hash-password` – Haszowanie hasła  
- `lint:yaml|twig|xliff` – Walidacja plików  
- `assets:install` – Instaluj assety  
- `messenger:consume` – Konsumuj wiadomości  
