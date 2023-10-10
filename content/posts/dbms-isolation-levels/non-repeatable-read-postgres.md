+++
title = 'Non Repeatable Read Postgres'
date = 2023-10-08T12:31:54+03:00
draft = false
tags = ['dbms', 'postgres']
+++

### Подготовка

Открываем терминал и стартуем контейнер с базой. Если уже есть готовая база, то этот шаг можно пропустить.

```console
docker run --name some-postgres -e POSTGRES_PASSWORD=mysecretpassword -p 5432:5432 -d postgres
```

Создаем простую схему базы и заполняем ее случайными данными:

```sql
CREATE TABLE test (
    field integer PRIMARY KEY,
    description text
);

INSERT INTO test (field, description)
VALUES
    (1, 'row 1'),
    (2, 'row 2'),
    (3, 'row 3'),
    (4, 'row 4');
```

| Isolation Level  | Dirty Read             | Nonrepeatable Read | Phantom Read           | Serialization Anomaly |
| ---------------- | ---------------------- | ------------------ | ---------------------- | --------------------- |
| Read uncommitted | Allowed, but not in PG | Possible           | Possible               | Possible              |
| Read committed   | Not possible           | Possible           | Possible               | Possible              |
| Repeatable read  | Not possible           | Not possible       | Allowed, but not in PG | Possible              |
| Serializable     | Not possible           | Not possible       | Not possible           | Not possible          |

### Воспроизводим non repeatable read

Пытаемся увидеть, что внутри одной транзакции мы можем получить разные данные на один и тот же запрос. 
То есть два последовательных одинаковых запроса могут вернуть нам отличающиеся наборы данных в зависимости от результата работы других транзакций.
Берем первый терминал, открываем транзакцию с уровнем read committed и делаем запрос на получение данных из таблицы.

```sql
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT * FROM test;
```

Получаем наши четыре строки с данными.

```console
 field | description
-------+-------------
     1 | row 1
     2 | row 2
     3 | row 3
     4 | row 4
(4 rows)
```

Затем берем второй терминал и открываем такую же транзакцию. В этом терминале мы изменим описания у всех строк.

```sql
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
UPDATE test SET description = 'wow, new description';
COMMIT;
```

После этого получим такие данные:

```console
 field |     description
-------+----------------------
     1 | wow, new description
     2 | wow, new description
     3 | wow, new description
     4 | wow, new description
(4 rows)
```

Возвращаемся в первый терминал и повторяем запрос.

```sql
SELECT * FROM test;
```

Мы увидим, что набор данных у нас изменился.

```console
 field |     description
-------+----------------------
     1 | wow, new description
     2 | wow, new description
     3 | wow, new description
     4 | wow, new description
(4 rows)
```

Как мы видим, транзакция из второго терминала завершилась и ее результат видно из первого терминала. В случае более сложных запросов могут быть не те данные, которые мы видели в начале транзакции. Допустим, в первый раз мы выбрали данные с каким-то условием и получили одну строку. Внешняя транзакция поменяла данные и теперь под наше условие уже подходит две строки. А в нашем сложном запросе мы второй раз делаем выборку и уже можем получить расхождения в данных при дальнейшей работе запроса.

Попробуем решить эту проблему через повышение уровня изоляции.
Снова берем первый терминал, открываем транзакцию и получаем все данные.

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM test;
```

```console
 field | description
-------+-------------
     1 | row 1
     2 | row 2
     3 | row 3
     4 | row 4
(4 rows)
```

И берем второй терминал в котором удалим четвертую строку и добавим пятую.

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
UPDATE test SET description = 'wow, new description';
COMMIT;
```

После этого данные у нас должны стать такими:

```console
 field |     description
-------+----------------------
     1 | wow, new description
     2 | wow, new description
     3 | wow, new description
     4 | wow, new description
(4 rows)
```

Теперь снова смотрим данные в первом терминале.

```sql
SELECT * FROM test;
```

Получаем снова те же 4 строки.

```console
 field | description
-------+-------------
     1 | row 1
     2 | row 2
     3 | row 3
     4 | row 4
(4 rows)
```

Таким образом, получается, что мы всегда будем работать с данными которые были на момент начала транзакции.
А что будет если в первом терминале попробовать обновить данные в строках и завершить транзакцию?

```sql
UPDATE test SET description = 'super new description';
COMMIT;
```

И получим мы в итоге вот такую ошибку.

```console
ERROR:  could not serialize access due to concurrent delete
ROLLBACK
```

Эта ошибка говорит как раз о том, что СУБД заметила, что данные изменились и не дала применить наши изменения на удаленных данных.
В то же время мы можем выполнить insert в первом терминале:

```sql
INSERT INTO test(field, description) VALUES (6, 'row 6');
COMMIT;
```

Такое изменение не вызывает конфликтов и поэтому в результате мы получим такое.

```console
 field | description
-------+-------------
     1 | row 1
     2 | row 2
     3 | row 3
     5 | row 5
     6 | row 6
(5 rows)
```
В итоге получается, что уровень изоляции repeatable read дает нам возможность работать всегда с данными на момент старта транзакции. При этом результат транзакции будет зависеть от того вызывают ли наши изменения конфликт. Новые данные которые появились в процессе не будут затронуты изменениями. Non repeatable read побежден.
