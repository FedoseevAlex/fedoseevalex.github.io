+++
title = 'Phantom Read Postgres'
date = 2023-10-08T12:32:05+03:00
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

### Воспроизводим phantom read

В первом терминале начинаем транзакцию и получаем все строки в у которых поле `field` нечетное.

```sql
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT * FROM test WHERE mod(field, 2) <> 0;
```

```console
 field | description
-------+-------------
     1 | row 1
     3 | row 3
(2 rows)
```

Видно, что у нас сейчас всего 2 строки подходят под наше условие.
Теперь, во втором терминале добавим пятую строку.

```sql
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
INSERT INTO test(field, description) VALUES (5, 'row 5');
COMMIT;
```

Теперь у нас такие строки в таблице.

```console
 field | description
-------+-------------
     1 | row 1
     2 | row 2
     3 | row 3
     4 | row 4
     5 | row 5
(5 rows)
```

В первом терминале снова попробуем получить строки с нечетным значением в `field`.

```sql
SELECT * FROM test WHERE mod(field, 2) <> 0;
```

Наш результат:

```console
 field | description
-------+-------------
     1 | row 1
     3 | row 3
     5 | row 5
(3 rows)
```

Как мы видим, внутри нашей транзакции мы получили новую строку, которая подходит под наше условие. Таким образом, мы можем обработать новую добавленную строку если она подойдет нам по условиям.

Попробуем поднять уровень изоляции и проделать эти же действия еще раз.
В первом терминале получаем строки с нечетным значением `field`.

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM test WHERE mod(field, 2) <> 0;
```

```console
 field | description
-------+-------------
     1 | row 1
     3 | row 3
(2 rows)
```

Далее, во втором терминале добавляем новую строку.

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
INSERT INTO test(field, description) VALUES (5, 'row 5');
COMMIT;
```

Возвращаемся в первый терминал и снова получаем строки с нечетным `field`.

```sql
SELECT * FROM test WHERE mod(field, 2) <> 0;
```

```console
 field | description
-------+-------------
     1 | row 1
     3 | row 3
(2 rows)
```

Как мы видим, при поднятии уровня изоляции до repeatable read postgres нам уже не дает увидеть новую строку. Никакого больше phantom read!
