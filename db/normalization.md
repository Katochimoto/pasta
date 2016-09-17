# Нормализация баз данных

Принципы нормализации описывают, как надо правильно проектировать таблицы для хранения данных. Если им не следовать, то потом с БД будет неудобно работать (а разработчики будут вспоминать проектировщика нехорошими словами). К сожалению, не везде эти принципы описаны понятным языком, потому я попытался найти доступные статьи, а также добавил пояснения своими словами.

Перед чтением этого урока полезно вспомнить, какие виды отношений между таблицами бывают в базе данных (один-к-одному, один-ко-многим, многие-ко-многим).

Ссылки по теме: 

- https://habrahabr.ru/post/129195/
- https://habrahabr.ru/post/254773/
- http://club.shelek.ru/viewart.php?id=177
- http://alexvolkov.ru/database-normalizatio.html
-  (сложновато, но зато официально) https://ru.wikipedia.org/wiki/%D0%9D%D0%BE%D1%80%D0%BC%D0%B0%D0%BB%D1%8C%D0%BD%D0%B0%D1%8F_%D1%84%D0%BE%D1%80%D0%BC%D0%B0

Теория описывает так называемые *нормальные формы*, пронумерованные от 1NF до 6NF (на практике хватает первых трех). Каждая форма содержит определенный список требований, которым должна соответствовать база данных (говорят "таблица находится во второй нормальной форме"). При этом требования для N-й нормальной формы включают в себя и требования к формам с меньшими номерами. То есть, чем выше номер, тем больше этот список. Обычно достаточно, чтобы БД находилась в третьей нормальной форме (3NF). 

Большинство требований направлено на борьбу с дублированием данных, чтобы информация (например, email или имя пользователя) хранилась только в одном экземпляре.

Я попробую описать эти формы своими словами и привести примеры ошибок проектирования БД:

## 1NF

Требования:

- В первой нормальной форме все значения в ячейках (их в теории называют *атрибуты*) должны быть атомарны, то есть содержать ровно одно неделимое значение, а не список из нескольких значений. 

Требуется хранить в одной ячейке таблицы одно неделимое значение. Вот пример таблицы сотрудников компании `employees`, где это правило нарушается. В одной колонке хранится и имя сотрудника, и его должность:

| id  | employee              |
|-----|-----------------------| 
| 1   | Иванов И.И., директор |
| 2   | Петров П.П., менеджер |
| 3   | Сидоров С.С.,менеджер |

Из-за этого нам нелегко, например, найти всех менеджеров, или выбрать только фамилии без должностей. Также, непросто написать запрос для изменения должности сотрудника, например, с "менеджер" на "старший менеджер". Для решения проблемы необходимо вынести должность в отдельную колонку. Также, возможно, имеет смысл разбить ФИО на 3 отдельных колонки, но это зависит от того, как они будут использоваться - всегда вместе или по отдельности. 

Вот другой пример нарушения 1NF. Допустим, у нас есть блог и к каждому посту можно добавить теги (темы, к которым относится пост). Разработчик не соблюдает принцип атомарности в таблице `posts` и хранит в одной ячейке все теги сразу: 

| id   | title         | tags                   | 
|------|---------------|------------------------|
| 1    | Основы PHP    | php, программирование  |
| 2    | Мой кот       | кот, личное            |
| 3    | Циклы в PHP   | php, циклы             |

Имеем такие недостатки: название тега не может содержать в себе запятую. Неудобно искать посты по тегу. Чтобы добавить или убрать тег, надо сначала выбрать полный список тегов, отредактировать его и сохранить обратно и это нелегко сделать одним запросом. Если мы захотим переименовать тег, то придется делать поиск и замену по всей таблице (и есть риск, что вместе с заменой тега "php" мы заменим и тег "уроки php"). Трудно вывести список тегов и число постов по каждому. Нельзя добавить тегу какие-то свойства (например, цвет или ссылку).

Для исправления проблемы теги необходимо сделать отдельной сущностью. Для этого нужно сделать отдельную таблицу `tags` для них: 

| id   | name   |
|------|--------|
| 1    | php    |
| 2    | программирование |
| 3    | кот    |
| 4    | личное |
| 5    | циклы  |

И таблицу связи постов с тегами (многие-ко-многим) `posts_to_tags`, состоящую из 2 колонок tag_id и post_id. Первичным ключом в ней будет эта пара колонок (это заодно не позволит поставить посту 2 одинаковых тега, так как значения перв. ключа должны быть уникальны). 

Мы заменили одну таблицу на три, но зато теперь с ними стало проще работать. Попробуй написать SQL-запросы для добавления тега к посту, удаления, переиенования и увидишь разницу.

# 2NF

На всякий случай напомню, что *первичным ключом* называют колонку или несколько колонок, значения которых не пусты, не повторяются и таким образом являются уникальным идентификатором. Первичный ключ бывает *естветсвенный* - когда он уже содержится в данных (например, номер телефона может быть первичным ключом в телефонном справочнике) или *суррогатным* - когда он добавлен искуственно (например, числовой id).

Практически для любой таблицы стоит определять первичный ключ, иначе с ней будет неудобно работать. 

Требования 2NF:

- БД должна соответствовать 1NF
- все поля в таблице должны *полностью зависеть* от первичного ключа целиком, а не от его части

Обычно это требование применяется только тогда, когда первичный ключ составной и содержит 2 или больше колонок. В этом случае, если какие-то значения в таблице зависят только от части ключа, то они могут повторяться, и их надо вынести в отдельную таблицу. Если первичный ключ - это одно поле, то таблица уже соответствует требованию.

"Поле A полностью зависит от B" здесь значит, что, зная B, можно найти значение A в таблице. Ну например, в таблице ниже поле `built_year` зависит от пары (`street_id`, `house_number`), но не зависит от них по отдельности (только по номеру дому нельзя понять о каком доме идет речь и найти его год постройки).

Попробую привести пример нарушения этого требования. Допустим, у нас есть таблица `buildings`, в которой хранится информация о зданиях в городе: число этажей, год постройки дома, название улицы: 

| street_id | street_name | street_type | house_number | storeys  | built_year | 
|----------:|-------------|-------------|-------------:|---------:|-----------:|
| 1         | Центральная | ул.         | 1            | 5        | 1960       |
| 1         | Центральная | ул.         | 2            | 8        | 1962       |
| 1         | Центральная | ул.         | 3            | 1        | 1932       |
| 2         | Спортивный  | просп.      | 1            | 12       | 1975       |
| 2         | Спортивный  | просп.      | 2            | 18       | 1982       |

Видно, что она соответствует 1NF, так как даже название улицы разбито на 2 части (`ул.` и `Центральная`). Попробуем понять, что в этой таблице может быть первичным ключом. Как указать на отдельный дом? Идентификаторы улиц встречаются по несколько раз, номера домов тоже. Однако их сочетание (`street_id`, `house_number`) - уникально. Это естественный первичный ключ.

Теперь попробуй, глядя на таблицу, понять, как здесь нарушено требование к 2NF.

Чтобы помешать подглядывать, я вставлю тут умное определение второй нормальной формы, которое не требуется учить наизусть (и даже читать), но которое помешает увидеть правильный ответ ниже.

    Переменная отношения находится *во второй нормальной форме* тогда и только тогда, когда она находится в первой нормальной форме и каждый неключевой атрибут *неприводимо* зависит от её потенциального ключа.

    *Неприводимость* означает, что в составе потенциального ключа отсутствует меньшее подмножество атрибутов, от которого можно также вывести данную функциональную зависимость. Для неприводимой функциональной зависимости часто используется эквивалентное понятие «полная функциональная зависимость»

Даже если ты не понял ни слова из определения, догадаться, в чем проблема, нетрудно. Видно, что значения в колонках `street_name` и `street_type` повторяются, так как они "зависят" только от одной колонки `street_id` (и нам достаточно знать только id улицы, без номера дома, чтобы найти ее название). Это дублирование увеличивает объем таблицы, а при попытке обновить название улицы, нам придется искать все ячейки, где оно есть, и заменять, что создает риск, что где-то мы его забудем поменять.

Эти 2 колонки надо вынести в отдельную таблицу-справочник улиц `streets`. Я заодно убрал префикс `street` из названий полей, так как он уже есть в имени таблицы:

| id    |  name         | type   |
|-------|---------------|--------|
| 1     | Центральная   | ул.    |
| 2     | Спортивный    | просп. |

## 3NF

Требования:

- должны выполняться требования 1NF и 2NF
- значения полей должны зависеть только от первичного ключа, а не от других полей

"Поле A зависит от B" значит, что одно поле (A) может служить идентификатором для другого (B). Ну к примеру, между фамилией человека (B) и номером паспорта (A) есть такая связь, так как (имея базу данных), по номеру паспорта можно определить фамилию (а обратное - неверно). Значит, фамилия человека "зависит" от его номера паспорта. В хорошо спроектированной таблице значения полей должны зависеть только от первичного ключа, но не от других полей.

Вот пример таблицы `stations` со списком станций метро, нарушающей 3NF. Она содержит идентификатор станции (`id`), название станции (`name`), идентификатор ветки метро (`line_id`), название ветки (`line_name`). Попробуй догадаться, что здесь не так: 

| id    | name              | line_id   | line_name          |
|-------|-------------------|-----------|--------------------|
| 1     | Чистые пруды      | 1         | Сокольническая     |
| 2     | Лубянка           | 1         | Сокольническая     |
| 3     | Спортивная        | 1         | Сокольническая     |
| 4     | Таганская         | 2         | Кольцевая          |
| 5     | Курская           | 2         | Кольцевая          |

Чтобы не подглядывать, вот текст, который не обязательно читать:

    Запоминающееся и, по традиции, наглядное резюме определения 3NF Кодда было дано Биллом Кентом: каждый неключевой атрибут «должен предоставлять информацию о ключе, полном ключе и ни о чём, кроме ключа».

    Условие зависимости от «полного ключа» неключевых атрибутов обеспечивает то, что отношение находится во второй нормальной форме; а условие зависимости их от «ничего, кроме ключа» — то, что они находятся в третьей нормальной форме.

Как и в прошлый раз, догадаться нетрудно, просто взглянув на таблицу: видно, что названия веток повторяются, так как "зависят" от `line_id`. Но 3NF требует чтобы поля зависели только от первичного ключа. Это значит, что поле `line_name` нужно вынести из этой таблицы в отдельную таблицу веток `lines`:

| id   | name            |
|------|-----------------|
| 1    | Сокольническая  |
| 2    | Кольцевая       |

## Другие нормальные формы

Еще есть более строгие BCNF, 4NF, 5NF, 6NF. Про них можно почитать в статьях по ссылкам выше, но в общем их суть сводится к тому, чтобы при наличии в таблице отношений и зависимостей между  колонками выносить их отдельно.

## Что будет, если не соблюдать требования нормализации

- будет неудобно писать некоторые запросы к таблицам
- данные будут дублироваться, что может привести к тому, что в одном месте будет одно значение, а в другом - другое и непонятно, какое из них правильное

Конечно, пока в базе несколько таблиц, это не так заметно, но в больших системах с десятками и сотнями таблиц последствия нарушения требований могут быть тяжелыми.
