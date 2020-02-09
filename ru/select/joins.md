---
titile: Джоины
---

# Джоины

[Джоины](https://ru.wikipedia.org/wiki/Join_(SQL)) один из самых востребованных механизмов SQL - позволяет стоить объединения и пересечения таблиц, что необходимо после [нормализации данных](https://ru.wikipedia.org/wiki/%D0%9D%D0%BE%D1%80%D0%BC%D0%B0%D0%BB%D1%8C%D0%BD%D0%B0%D1%8F_%D1%84%D0%BE%D1%80%D0%BC%D0%B0).

Итак, SQL предоставляет нам несколько видов джонинов: `LEFT`, `RIGHT`, `INNER`, `FULL`, `CROSS`. К сожалению при разработке LINQ не были учтены все особенности этих операций, ввиду чего некоторые из них достаточно сложно, а иногда невозможно, выразить в LINQ синтаксисе. В частности, штатный синтаксис позволяет сильно ограничен в части построения предиката для утверждения  `ON`.

Давайте рассмотрим способы построения запросов с джоинами, которые поддерживает `linq2db`.

## INNER JOIN

Шатный синтаксис с равенством двух полей:

```c#
var query =
    from c in db.Category
    join p in db.Product on c.CategoryID equals p.CategoryID
    where !p.Discontinued
    select c;
```

Так же `linq2db` позволяет строить джоин, с использованием функции `Where` (обратите внимание, что в данном случае **не используется** ключевое слово `join`, вместо него используем `from`):

```c#
var query =
    from c in db.Category
    from p in db.Product.Where(pr => pr.CategoryID == c.CategoryID)
    where !p.Discontinued
    select c;
```

И наконец, использование специальной функции `InnerJoin`:

```c#
var query =
    from c in db.Category
    from p in db.Product.InnerJoin(pr => pr.CategoryID == c.CategoryID)
    where !p.Discontinued
    select c;
```

Для всех этих вариантов мы получим одинаковый SQL:

```sql
SELECT
    [c].[CategoryID],
    [c].[CategoryName],
    [c].[Description],
    [c].[Picture]
FROM
    [Categories] [c]
        INNER JOIN [Products] [p] ON [c].[CategoryID] = [p].[CategoryID]
WHERE
    [p].[Discontinued] <> 1
```

### Join по нескольким полям

Это пример того, как порой сложно выразить простыв вещи:

```cs
var query =
    from p in db.Product
    from o in db.Order
    join d in db.OrderDetail
        on     new { p.ProductID, o.OrderID }
        equals new { d.ProductID, d.OrderID }
    where !p.Discontinued
    select new
    {
        p.ProductID,
        o.OrderID,
    };
```

И в то же время не так уж и сложно с использованием специального метода `InnerJoin`:

```cs
var query =
    from p in db.Product
    from o in db.Order
    join d in db.OrderDetail.InnerJoin(_ => p.ProductID = _.ProductID && o.OrderID == _.OrderID)
    where !p.Discontinued
    select new
    {
        p.ProductID,
        o.OrderID,
    };
```

Ну и выходной SQL:

```sql
SELECT
    [t3].[ProductID] as [ProductID1],
    [t3].[OrderID] as [OrderID1]
FROM
    (
        SELECT
            [t1].[ProductID],
            [t2].[OrderID],
            [t1].[Discontinued]
        FROM
            [Products] [t1],
            [Orders] [t2]
    ) [t3]
        INNER JOIN [Order Details] [d] ON [t3].[ProductID] = [d].[ProductID] AND [t3].[OrderID] = [d].[OrderID]
WHERE
    [t3].[Discontinued] <> 1
```

## LEFT JOIN

Так же рассмотрим несколько примеров запросов. Штатный:

```cs
var query =
    from c in db.Category
    join p in db.Product on c.CategoryID equals p.CategoryID into lj
    from lp in lj.DefaultIfEmpty()
    where !lp.Discontinued
    select c;
```

С использованием `Where().DefaultIfEmpty()`:

```cs
var query =
    from c in db.Category
    from lp in db.Product.Where(p => p.CategoryID == c.CategoryID).DefaultIfEmpty()
    where !lp.Discontinued
    select c;
```

И с использованием специального метода `LeftJoin`:

```cs
var query =
    from c in db.Category
    from p in db.Product.LeftJoin(pr => pr.CategoryID == c.CategoryID)
    where !p.Discontinued
    select c;
``````

SQL:

```sql
SELECT
    [c1].[CategoryID],
    [c1].[CategoryName],
    [c1].[Description],
    [c1].[Picture]
FROM
    [Categories] [c1]
        LEFT JOIN [Products] [lj] ON [c1].[CategoryID] = [lj].[CategoryID]
WHERE
    1 <> [lj].[Discontinued]
```

## RIGHT JOIN

Штатных способов выразить этот вид джоина нет (хотя, справедливости для отмечу, что `RIGHT JOIN` в природе встречается крайне редко, т.к. затрудняет прочтение SQL запроса - по сути он просто меняет порядок таблиц в утверждении `LEFT JOIN`, и гораздо проще переписать запрос с использованием оного, чем читать "снизу вверх"). `linq2db` представляет специальный метод `RightJoin` для построения данных запросов:

```cs
var query =
    from c in db.Category
    from p in db.Product.RightJoin(pr => pr.CategoryID == c.CategoryID)
    where !p.Discontinued
    select c;
```

SQL:

```sql
SELECT
    [t2].[CategoryID],
    [t2].[CategoryName],
    [t2].[Description],
    [t2].[Picture]
FROM
    [Categories] [t2]
        RIGHT JOIN [Products] [t1] ON [t1].[CategoryID] = [t2].[CategoryID]
WHERE
    1 <> [t1].[Discontinued]
```

## FULL JOIN

Данный джоин так же доступен только через специальный метод `FullJoin`:

```cs
var query =
    from c in db.Category
    from p in db.Product.FullJoin(pr => pr.CategoryID == c.CategoryID)
    where !p.Discontinued
    select c;
```

SQL:

```sql
SELECT
    [t2].[CategoryID],
    [t2].[CategoryName],
    [t2].[Description],
    [t2].[Picture]
FROM
    [Categories] [t2]
        FULL JOIN [Products] [t1] ON [t1].[CategoryID] = [t2].[CategoryID]
WHERE
    1 <> [t1].[Discontinued]
```

## CROSS JOIN

Данный вид джоина это декартово произведение нескольких таблиц, выражается достаточно просто через штатный синтаксис:

```cs
var query =
    from c in db.Category
    from p in db.Product
    where !p.Discontinued
    select new {c, p};
```

SQL:

```sql
SELECT
    [t1].[CategoryID],
    [t1].[CategoryName],
    [t1].[Description],
    [t1].[Picture],
    [t2].[ProductID],
    [t2].[ProductName],
    [t2].[SupplierID],
    [t2].[CategoryID] as [CategoryID1],
    [t2].[QuantityPerUnit],
    [t2].[UnitPrice],
    [t2].[UnitsInStock],
    [t2].[UnitsOnOrder],
    [t2].[ReorderLevel],
    [t2].[Discontinued]
FROM
    [Categories] [t1],
    [Products] [t2]
WHERE
    1 <> [t2].[Discontinued]
```

## Вывод

Какой синтаксис использовать остается полностью на ваше усмотрение, лично я предпочитаю использование специальных методов `InnerJoin`, `LeftJoin`, `RightJoin` и `FullJoin`:

* они выразительны (ни мне, ни моим коллегам читая запрос не придется ни о чем догадываться);
* они лаконичны (что ярко видлно на `LEFT JOIN`);
* и самое важное - они позволяют строить **полноценные** предикаты для утверждения `ON`.

## Далее

Мы рассмотрим использование разных [функций](functions.md).
