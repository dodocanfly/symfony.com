# Kontener usług (Service Container)

> [!TIP]
> Czy wolisz samouczki wideo? Sprawdź serię screencastów [Symfony Fundamentals](https://symfonycasts.com/screencast/symfony-fundamentals).

Twoja aplikacja jest *pełna* przydatnych obiektów: obiekt „Mailer” może pomóc Ci w wysyłaniu e-maili, podczas gdy inny obiekt może służyć do zapisywania danych w bazie. Prawie wszystko, co „robi” Twoja aplikacja, jest w rzeczywistości wykonywane przez jeden z takich obiektów. Za każdym razem, gdy instalujesz nowy pakiet (bundle), zyskujesz dostęp do jeszcze większej liczby takich narzędzi!

W Symfony te przydatne obiekty nazywają się **usługami** i każda z nich znajduje się wewnątrz bardzo specjalnego obiektu — **kontenera usług**. Kontener pozwala scentralizować sposób tworzenia obiektów. Ułatwia pracę, wspiera solidną architekturę i działa bardzo szybko!
