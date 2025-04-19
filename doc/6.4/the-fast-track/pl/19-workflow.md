## Podejmowanie decyzji z pomocą Workflow

Posiadanie stanu dla modelu jest dość powszechne. Stan komentarza jest obecnie określany wyłącznie przez system sprawdzania spamu. A co, jeśli dodamy więcej czynników decyzyjnych?

Być może chcielibyśmy, aby administrator strony moderował wszystkie komentarze po przejściu przez filtr spamu. Proces mógłby wyglądać następująco:
- Rozpocznij od stanu `submitted`, gdy komentarz zostanie przesłany przez użytkownika;
- Pozwól, aby filtr spamu przeanalizował komentarz i zmienił jego stan na jeden z: `potential_spam`, `ham` lub `rejected`;
- Jeśli komentarz nie zostanie odrzucony, poczekaj, aż administrator strony zdecyduje, czy komentarz jest wystarczająco dobry, zmieniając jego stan na `published` lub `rejected`.

Implementacja takiej logiki nie jest zbyt skomplikowana, ale możesz sobie wyobrazić, że dodanie większej liczby reguł znacznie zwiększyłoby złożoność. Zamiast samodzielnie kodować tę logikę, możemy użyć komponentu Workflow w Symfony:

```bash
symfony composer req workflow
```
