+++
title = 'Операция Prepend в golang'
date = 2024-01-13T14:03:30+02:00
draft = false
+++

Столкнулся с задачей в которой необходимо была операция prepend (помещение элемента в начало списка). Так как код сервиса написан на golang, то для этого языка проблему и рассмотрим.  
Есть несколько способов положить значение в начало списка. Посмотрим на них по-отдельности. Попытаюсь, как умею, сравнить их производительность и выбрать лучший.

## TLDR

Используйте вот такую функцию для операции prepend, так как она самая эффективная.

```go
func Prepend[T any](value T, arr []T) []T {
    if len(arr) == 0 {
        return append(arr, value)
    }

    var zeroValue T
    arr = append(arr, zeroValue)

    copy(arr[1:], arr)
    arr[0] = value

    return arr
}
```

## Очень простой способ

```go
func PrependSimple[T any](value T, arr []T) []T {
    return append([]T{value}, arr...)
}
```

Даже без глубокого анализа можно понять, что такой способ приведет к излишнему количеству аллокаций. Здесь в первом аргументе к `append` аллоцируется память под новый список. Затем еще одна аллокация чтобы добавить к списку все значения слайса `arr`.

## Способ с копированием из go slice tricks

```go
func PrependCopy[T any](value T, arr []T) []T {
    if len(arr) == 0 {
        return append(arr, value)
    }

    var zeroValue T
    arr = append(arr, zeroValue)

    copy(arr[1:], arr)
    arr[0] = value

    return arr
}
```

Этот способ не делает много лишних движений. Аллокация произойдет только в случае если емкость нижележащего массива закончится.
Однако, здесь есть накладные расходы на копирование данных. Копированием мы добиваемся сдвига элементов на ячейку вправо.

## Способ reverse - append - reverse

```go
func PrependReverse[T any](value T, arr []T) []T {
    reverse(arr)
    arr = append(arr, value)
    reverse(arr)
    return arr
}


func reverse[T any](arr []T) {
    for i := len(arr)/2 - 1; i >= 0; i-- {
        opp := len(arr) - 1 - i
        arr[i], arr[opp] = arr[opp], arr[i]
    }
}
```

Это, конечно, изощренный способ, но его просто захотелось проверить вместе с остальными.

## Что показали бенчмарки?

Полный код тестов и варианты prepend можно будет посмотреть в конце.
Запустим бенчмарки и посмотрим на них.

```shell
go test -bench=. -benchmem -benchtime 10s
```

И вот какой результат:

```text
goos: darwin
goarch: arm64
pkg: prepend
BenchmarkPrependCopyEmptyArr-8              686147342        17.54 ns/op              8 B/op           1 allocs/op
BenchmarkPrependSimpleEmptyArr-8            1000000000      0.4096 ns/op              0 B/op           0 allocs/op
BenchmarkPrependReverseEmptyArr-8           596733974        20.09 ns/op              8 B/op           1 allocs/op

BenchmarkPrependCopyWithoutAlloc-8          1000000000      0.4123 ns/op              0 B/op           0 allocs/op
BenchmarkPrependSimpleWithoutAlloc-8        1000000000       2.419 ns/op              0 B/op           0 allocs/op
BenchmarkPrependReverseWithoutAlloc-8       1000000000       4.545 ns/op              0 B/op           0 allocs/op

BenchmarkPrependCopy10-8                    77814157         147.6 ns/op            248 B/op           5 allocs/op
BenchmarkPrependSimple10-8                  44470742         280.0 ns/op            536 B/op          19 allocs/op
BenchmarkPrependReverse10-8                 80974004         154.4 ns/op            248 B/op           5 allocs/op

BenchmarkPrependCopy100-8                    8796500          1376 ns/op           2040 B/op           8 allocs/op
BenchmarkPrependSimple100-8                  1502822          7888 ns/op          43016 B/op         199 allocs/op
BenchmarkPrependReverse100-8                 3057127          3905 ns/op           2040 B/op           8 allocs/op

BenchmarkPrependCopy1K-8                      159852         74808 ns/op          25208 B/op          12 allocs/op
BenchmarkPrependSimple1K-8                     21598        548318 ns/op        4281955 B/op        1999 allocs/op
BenchmarkPrependReverse1K-8                    37006        324320 ns/op          25208 B/op          12 allocs/op

BenchmarkPrependCopy100K-8                         12    982666875 ns/op        4101384 B/op          28 allocs/op
BenchmarkPrependSimple100K-8                       3    3618199472 ns/op    40398482480 B/op      202678 allocs/op
BenchmarkPrependReverse100K-8                      4    3175945719 ns/op        4101400 B/op          28 allocs/op
PASS
ok      prepend    134.436s
```

Посмотрим на каждую группу тестов отдельно.

### Тест на nil слайсе

```text
BenchmarkPrependSimpleEmptyArr-8            1000000000          0.4096 ns/op           0 B/op           0 allocs/op
BenchmarkPrependCopyEmptyArr-8              686147342            17.54 ns/op           8 B/op           1 allocs/op
BenchmarkPrependReverseEmptyArr-8           596733974            20.09 ns/op           8 B/op           1 allocs/op
```

На пустом слайсе победителем оказался самый простой вариант. На nil слайсе получается следующее выражение:

```go
    append([]T{value}, nil...)
```

Из аллокаций должно быть только создание списка с одним единственным элементом. Функция `append` в таких условиях ничего не должна сделать.
Очень странно, что в бенчмарке стоит ноль аллокаций. Возможно, компилятор что-то придумал чтобы не делать лишних аллокаций.

### Тест со слайсом достаточного размера

```text
BenchmarkPrependCopyWithoutAlloc-8          1000000000             0.4123 ns/op          0 B/op           0 allocs/op
BenchmarkPrependSimpleWithoutAlloc-8        1000000000             2.419 ns/op           0 B/op           0 allocs/op
BenchmarkPrependReverseWithoutAlloc-8       1000000000             4.545 ns/op           0 B/op           0 allocs/op
```

Здесь снова `PrependSimple` прошел тест без дополнительных аллокаций. Кажется, все же компилятор что-то оптимизировал.
В остальном, видно что вариант с копированием явно лидирует по скорости работы. Встроенная функция `copy` работает сильно лучше чем цикл for в варианте `PrependReverse`.

### Тесты на добавление 10, 100, 1К и 100К элементов

```text
BenchmarkPrependCopy10-8                    77814157           147.6 ns/op           248 B/op           5 allocs/op
BenchmarkPrependReverse10-8                 80974004           154.4 ns/op           248 B/op           5 allocs/op
BenchmarkPrependSimple10-8                  44470742           280.0 ns/op           536 B/op          19 allocs/op

BenchmarkPrependCopy100-8                    8796500          1376 ns/op            2040 B/op           8 allocs/op
BenchmarkPrependReverse100-8                 3057127          3905 ns/op            2040 B/op           8 allocs/op
BenchmarkPrependSimple100-8                  1502822          7888 ns/op           43016 B/op         199 allocs/op

BenchmarkPrependCopy1K-8                      159852         74808 ns/op           25208 B/op          12 allocs/op
BenchmarkPrependReverse1K-8                    37006        324320 ns/op           25208 B/op          12 allocs/op
BenchmarkPrependSimple1K-8                     21598        548318 ns/op         4281955 B/op        1999 allocs/op

BenchmarkPrependCopy100K-8                        12     982666875 ns/op         4101384 B/op          28 allocs/op
BenchmarkPrependReverse100K-8                      4    3175945719 ns/op         4101400 B/op          28 allocs/op
BenchmarkPrependSimple100K-8                       3    3618199472 ns/op     40398482480 B/op      202678 allocs/op
```

На более менее реальных размерах массивов видно, что `PrependCopy` ведет себя лучше других вариантов.  
Несмотря на неплохое количество аллокаций, вариант `PrependReverse` на больших слайсах сильно уступает `PrependCopy` по скорости исполнения.  
Из бенчмарков можно сделать вывод, что лучше всего использовать `PrependCopy` вариант для списков любого размера.

## Весь код

Файл: `prepend.go`

```go
package prepend

import (
    "fmt"
)

func main() {
    var arr []int

    for i:=0; i < 100; i++ {
        arr = PrependCopy(i, arr)
    }

    fmt.Println("That's it")
}

func PrependCopy[T any](value T, arr []T) []T {
    if len(arr) == 0 {
        return append(arr, value)
    }

    var zeroValue T
    arr = append(arr, zeroValue)

    copy(arr[1:], arr)
    arr[0] = value

    return arr
}

func PrependSimple[T any](value T, arr []T) []T {
    return append([]T{value}, arr...)
}

func PrependReverse[T any](value T, arr []T) []T {
    reverse(arr)
    arr = append(arr, value)
    reverse(arr)
    return arr
}


func reverse[T any](arr []T) {
    for i := len(arr)/2 - 1; i >= 0; i-- {
        opp := len(arr) - 1 - i
        arr[i], arr[opp] = arr[opp], arr[i]
    }
}

```

Файл: `prepend_test.go`

```go
package prepend

import (
    "testing"
)

func BenchmarkPrependCopyEmptyArr(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        PrependCopy(1, nil)
    }
}

func BenchmarkPrependSimpleEmptyArr(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        PrependSimple(1, nil)
    }
}

func BenchmarkPrependReverseEmptyArr(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        PrependReverse(1, nil)
    }
}

func BenchmarkPrependCopyWithoutAlloc(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        var arr []int = make([]int, 0, 10)
        arr = PrependCopy(1, arr)
    }
}

func BenchmarkPrependSimpleWithoutAlloc(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        var arr []int = make([]int, 0, 10)
        arr = PrependSimple(1, arr)
    }
}

func BenchmarkPrependReverseWithoutAlloc(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        var arr []int = make([]int, 0, 10)
        arr = PrependReverse(1, arr)
    }
}

func BenchmarkPrependCopy10(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        var arr []int
        for i := 0; i < 10; i++ {
            arr = PrependCopy(1, arr)
        }
    }
}

func BenchmarkPrependSimple10(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        var arr []int
        for i := 0; i < 10; i++ {
            arr = PrependSimple(1, arr)
        }
    }
}

func BenchmarkPrependReverse10(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        var arr []int
        for i := 0; i < 10; i++ {
            arr = PrependReverse(1, arr)
        }
    }
}

func BenchmarkPrependCopy100(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        var arr []int
        for i := 0; i < 100; i++ {
            arr = PrependCopy(1, arr)
        }
    }
}

func BenchmarkPrependSimple100(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        var arr []int
        for i := 0; i < 100; i++ {
            arr = PrependSimple(1, arr)
        }
    }
}

func BenchmarkPrependReverse100(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        var arr []int
        for i := 0; i < 100; i++ {
            arr = PrependReverse(1, arr)
        }
    }
}

func BenchmarkPrependCopy1K(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        var arr []int
        for i := 0; i < 1000; i++ {
            arr = PrependCopy(1, arr)
        }
    }
}

func BenchmarkPrependSimple1K(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        var arr []int
        for i := 0; i < 1000; i++ {
            arr = PrependSimple(1, arr)
        }
    }
}

func BenchmarkPrependReverse1K(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        var arr []int
        for i := 0; i < 1000; i++ {
            arr = PrependReverse(1, arr)
        }
    }
}

func BenchmarkPrependCopy100K(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        var arr []int
        for i := 0; i < 100000; i++ {
            arr = PrependCopy(1, arr)
        }
    }
}

func BenchmarkPrependSimple100K(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        var arr []int
        for i := 0; i < 100000; i++ {
            arr = PrependSimple(1, arr)
        }
    }
}

func BenchmarkPrependReverse100K(b *testing.B) {
    for bench := 0; bench < b.N; bench++ {
        var arr []int
        for i := 0; i < 100000; i++ {
            arr = PrependReverse(1, arr)
        }
    }
}
```
