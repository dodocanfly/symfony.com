# 🧪 Symfony CLI — Najczęściej używane komendy

### 🔧 Lokalne środowisko i serwer

```bash
symfony serve             # Uruchomienie lokalnego serwera (na porcie 8000)
symfony serve -d          # Serwer w tle (daemon)
symfony server:stop       # Zatrzymanie działającego serwera
```

---

### 🚀 SymfonyCloud — deploy & środowiska

```bash
symfony cloud:login       # Logowanie do SymfonyCloud
symfony cloud:init        # Inicjalizacja projektu z SymfonyCloud
symfony cloud:deploy      # Deployment kodu na środowisko zdalne
symfony cloud:redeploy    # Redeployment ostatniej wersji (np. by naprawić błędy builda)
symfony cloud:env:list    # Lista środowisk (production, staging, preview itd.)
symfony cloud:checkout    # Przełączenie się lokalnie na wybrane środowisko
```

---

### 📦 Dane i bazy danych

```bash
symfony cloud:sql         # Łączenie z bazą danych (np. MySQL/PostgreSQL) przez CLI
symfony cloud:tunnel:open # Tunelowanie lokalnego połączenia do bazy z chmury
```

---

### 🐞 Debugowanie i logi

```bash
symfony cloud:logs        # Wyświetlenie logów środowiska (np. prod)
symfony cloud:activity    # Historia akcji i deploymentów
symfony debug:router      # Sprawdzenie zarejestrowanych tras w aplikacji
```

---

### 🌐 Domeny i certyfikaty SSL

```bash
symfony cloud:domain:add     # Dodanie domeny
symfony cloud:certificate:request # Wygenerowanie certyfikatu SSL
symfony cloud:domain:list    # Lista domen przypiętych do projektu
```

---

### ⏰ Crontab (zadania cykliczne)

```bash
symfony cloud:cron:add     # Dodanie zadania cron
symfony cloud:cron:list    # Lista cronów
```

---

### 🧰 Narzędzia developerskie

```bash
symfony console            # Uruchomienie Symfony Console (np. doctrine:migrations:migrate)
symfony var:export         # Eksport zmiennych środowiskowych do `.env.local`
symfony open:local         # Otwiera aplikację lokalnie w przeglądarce
symfony cloud:url          # Wyświetla publiczny URL środowiska
```

---

### 🔑 Klucze SSH

```bash
symfony cloud:ssh-key:add     # Dodanie klucza SSH
symfony cloud:ssh-key:list    # Lista Twoich kluczy
```

---

### ✅ Deployment — typowy scenariusz

```bash
git push origin main
symfony cloud:deploy
symfony console doctrine:migrations:migrate
symfony cloud:logs -e=prod
```
