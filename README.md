# notes
Тут буду писать всякие мысли насчет code-style, архитектуры и тому подобного

## Название Сервиса
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

## AggregateRoot в Eloquent
Не секрет, что в отличие от Doctrine, в Eloquent нет никаких UnitOfWork и IdentityMap,
и это приводит к тому, что в разных местах одна и та же модель
будет представлена совершенно никак не связанными между собой объектами:
```php
$user = User::find(1);
$userProfilePicture = $user->profilePicture;
$theSameUser = $userProfilePicture->user; 
// $user и $theSameUser - два разных объекта.
// Изменения в одном никак не отразятся на другом,
// если только вы не будете постоянно вызывать refresh().
```
Соответственно, если вы будете передавать куда-то "ребёнка", и функция через "ребёнка" получит "родителя", и внесет в него изменения, ваш оригинальный "родитель" останется неизменным, что может привести к багам. У этой проблемы существуют следующие решения:

##### 1. Передаем некорневые модели, полагаясь на вызов refresh() на корне. Изменения будут идти от некорневой модели во все стороны.

Этот подход является общепринятым из-за своей простоты, но он может быть довольно тяжелым для базы данных из-за постоянных запросов, и учитывая, что в некоторых случаях это приведет к промежуточному состоянию данных в БД, вам скорее всего придется прибегнуть к явным транзакциям, чтобы ваши изменения БД не были видны в других Requests/Jobs. Более того, вы не всегда можете знать, когда стоит вызывать refresh() - к примеру, вы можете не знать, о том, что существует какой-то Event, получающий "ребенка", но вносящий изменения в "родителя".
```php
$user = User::find(1);
// Передаём в функцию некорневую модель.
updateProfilePicture($user->profilePicture, 'cat.jpg');
// Обновляем корень агрегации, чтобы получить все изменения.
$user->refresh();

function updateProfilePicture(ProfilePicture $profilePicture, string $file)
{
    $profilePicture->img = $file;
    ...
    $user = $profilePicture->user;
    $user->dtUpdatedProfilePicture = now();
    ...
    // Не надеемся, что управляющий код вызовет ->push() на корне, а сохраняем всё прямо на месте. 
    $profilePicture->save();
    // В том числе и сам корень.
    $user->save();
}
```

##### 2. Передаём исключительно корень. В таком случае все изменения должны идти от корня к листьям.

Этот подход намного легче на БД, но требует длиннющую транзакцию в тех случаях, когда вы создаете новые связи между моделями, ведь создание связей обновляет БД.
```php
$user = User::find(1);
// Передаём в функцию корень
updateProfilePicture($user, 'cat.jpg');
// Нам не нужно ничего доставать из БД.
// Все изменения в агрегации можно "увидеть" через корень
// Соответственно, сохранение корня приведет к сохранению всех изменений.
$user->push();


function updateProfilePicture(User $user, string $file)
{
    $profilePicture = $user->profilePicture;
    $profilePicture->img = $file;
    ...
    $user->dtUpdatedProfilePicture = now();
    // Ничего не сохраняем
}
```

##### 3. Используем жалкую имитацию UnitOfWork в виде DTO
В этом варианте мы откладываем сохранение __и создание связей__ до самого последнего момента. Это позволяет внести дополнительные изменения и провести необходимые проверки после того как функция "создала" связи между моделями. И вам не нужно растягивать длиннющую транзакцию на всё время исполнения, чтобы иметь возможность всё откатить. Этот подход имеет смысл только в пределах одного класса (Job или Service), где обновление/валидация/создание_связей размазаны по методам.
```php
class UserCommentBanhamer
{
    public function banComments(array $comments)
    {
        $unitOfWork = [];
    
        foreach ($comments as $comment) {
            $this->banUserComment(
                // Все модели, которые будут использоваться в сервисе лежат в DTO.
                $dto = new DTO($comment->user, $comment),
            );
            
            $unitOfWork[] = $dto;
        }
        
        // Мы не используем отношения, а получаем все модели из DTO.
        $this->doSomething($dto);
        
        // Наличие UnitOfWork позволяет отложить и не растягивать транзакцию на всё время исполнения
        $this->flush($unitOfWork);
    }

    protected function banUserComment(DTO $dto)
    {
        // ->message это связь, которую мы можем отложить на попозже благодаря UnitOfWork
        $dto->message = new Message(['You are banned because of your', new BBLink('comment', route('comments', [$dto->comment->id]))]);
        $dto->user->dtBanEnds = now()->addDays(7);
        $dto->comment->isBanned = true;
        // Пока ничего не сохраняем
    }
    
    protected function flush(array $unitOfWork)
    {
        DB::transaction(function () use ($unitOfWork) {
            // Используем данные из UnitOfWork для формирования связей между моделями
            foreach ($unitOfWork as $dto) {
                $dto->user->messages->save($dto->message);
                $dto->user->save();
                $dto->comment->save();
            }
        });
    }
}





