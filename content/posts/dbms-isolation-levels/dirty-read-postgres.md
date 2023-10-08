+++
title = 'Dirty Read in Postgres'
date = 2023-10-08T12:22:45+03:00
draft = true
tags = ['dbms', 'postgres']
+++

Попробуем воспроизвести dirty read в postgres.

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

У postgresql есть четыре уровня изоляции, однако реально их там три. Уровень read uncommited создан только чтобы соответствовать стандарту SQL, а на деле же он воспринимается СУБД и работает как read commited.
Еще у этой СУБД есть вот такая [таблица](https://www.postgresql.org/docs/current/transaction-iso.html#MVCC-ISOLEVEL-TABLE) с уровнями изоляции и проблемами которые мы можем встретить. Будем иметь ввиду, что 

| Isolation Level  | Dirty Read             | Nonrepeatable Read | Phantom Read           | Serialization Anomaly |
| ---------------- | ---------------------- | ------------------ | ---------------------- | --------------------- |
| Read uncommitted | Allowed, but not in PG | Possible           | Possible               | Possible              |
| Read committed   | Not possible           | Possible           | Possible               | Possible              |
| Repeatable read  | Not possible           | Not possible       | Allowed, but not in PG | Possible              |
| Serializable     | Not possible           | Not possible       | Not possible           | Not possible          |

### Воспроизводим dirty read

Здесь мы хотим увидеть, что возможно прочитать данные из еще незавершенной транзакции.
Скорее всего у нас это не получится из-за особенностей postgresql, но мы все равно проверим это.

Берем два разных терминала и начинаем вбивать там поочередно команды.
В первом терминале начинаем транзакцию:

```sql
BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
INSERT INTO test(field, description) VALUES (5, 'row 5');
```

Потом берем второй терминал, там тоже начинаем транзакцию. Пытаемся увидеть нашу новую строку из первого терминала.

```sql
BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT * FROM test;
```

И получаем такой вывод:

```console
 field | description
-------+-------------
     1 | row 1
     2 | row 2
     3 | row 3
     4 | row 4
(4 rows)
```

Если бы не особенность поведения postgres относительно уровня read uncommitted, то мы должны были бы увидеть такой вывод.

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

Можно только сказать спасибо postgres, что минимальный уровень изоляции (и к тому же дефолтный) это read committed.

Не забываем выйти из обеих транзакций! Вводим это в каждый терминал.

```sql
ROLLBACK;
```

По итогу видим, что dirty read в postgres запрещен даже на самом минимальном уровне изоляции.
