# notes
Тут буду писать всякие мысли насчет code-style, архитектуры и тому подобного

### Сервисы
Название сервиса ни в коем случае не должно заканчиваться на "*Service". Обычно это признак того,
что вы только что создали свалку для слабо связанных между собой методов.
Вместо этого название сервиса должно происходить от глагола или [паттерна (Factory, Strategy, Builder, etc)](https://en.wikipedia.org/wiki/Software_design_pattern)

Плохо:
```php
// Набор совершенно несвязанных между собой методов
class CommentsService
{
    ...
    public function getComment(int $id) { ... }
    ...
    public function markAsSpam(Comment $comment) { ... }
}
```

Хорошо:
```php
// Происходит от паттерна Repository - все методы связаны с хранением объектов одного типа
class CommentsRepository
{
    ...
    public function getComment(int $id) { ... }
}

// Происходит от глагола - все методы будут связаны с "помечанием" (to mark) комментариев как "спам"
class CommentsSpamMarker
{
    ...
    public function markAsSpam(Comment $comment) { ... }
}
```
