# notes
Тут буду писать всякие мысли насчет code-style, архитектуры и тому подобного

### Сервисы
Название сервиса ни в коем случае не должно заканчиваться на "*Service". Обычно это признак того,
что вы только что создали свалку для слабо связанных между собой методов.
Вместо этого название сервиса должно происходить от глагола или [паттерна (Factory, Observer, etc)](https://en.wikipedia.org/wiki/Software_design_pattern)

Плохо:
```php
class UserNameFormattingService
{
  public function fullName($user) { ... }
  public function firstAndLastName($user) { ... }
  public function firstName($user) { ... }
}
```

Хорошо
```php
namespace UserNameFormatter; 

class FullNameFormatter {
  ...
  public function format($user) { ... }
}

class Factory
{
  public function makeFullName($user) {
    return new FullNameFormatter(...); 
  }
  ...
}
```
