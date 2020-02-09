---
titile: Функции
---

# Функции

Все SQL диалекты богаты на разного рода функции:

* арифметические;
* тригонометрические;
* для работы со строками;
* для работы со временем;
* etc...

И очень часто мы хотим их использовать, как, например, в предыдущей главе, где мы использовали преобразование даты со временем к дате в ключе для группировки. Надеюсь вы обратили внимание на то, что мы использовали свойство `DateTime.Date` для группировки в запросе, а `linq2db` для нас корректно перевел это на SQL диалект `SQLite`.

С полным перечнем поддерживаемых функций можно ознакомиться в документации к классу [LinqToDB.Sql](https://linq2db.github.io/api/LinqToDB.Sql.html). Здесь мы не будем останавливаться на полном перечне методов, рассмотрим только примеры использования.

Рассмотрим метод получения длинны строки:

```cs
db.GetTable<Customer>().Select(_ =>
    new
    {
        Implicit = _.FullName.Length,
        Explicit = Sql.Length(_.FullName)
    }
);
```

Как нетрудно догадаться результирующий SQL будет примерно следующего вида:

```sql
SELECT
    length(FullName),
    length(FullName)
FROM Customer
```

В обоих случаях вызов будет переведен в SQL функцию `length`, и, как следствие, выполнен на стороне базы данных. В первом случае будет выполнено неявное преобразование, во втором явное. Использование неявных преобразований - суть удобство, оно позволяет нам использовать свойства и методы C# типов так, как мы привыкли это делать, а всю рутинную работу берет на себя `linq2db`.

Неявные преобразования определены для большинства типов и методов, где это в принципе возможно:

* `string`: длинна, поиск подстроки, символ по индексу, и т.д.
* `DateTime`: преобразование к дате, получение числа, дня недели или любой другой части даты, и даже для операций сложения и вычитания дат есть неявные преобразования.
* `Math`: получение абсолютного значения, тригонометрические функции, округления и т.д.

В 90% случаев вам не придется задумываться о том, как в `linq2db` использовать ту или иную функцию, обычный ответ - так же как в C#. Однако, в ряде случаев, допустим для генерации нового `GUID` необходимо использовать явный вызов:

```cs
db.GetTable<Customer>()
    .Select(_ => Sql.NewGuid())
    .First();
```

## Объявление собственных функций

Как вы уже, наверное, догадались `linq2db` позволяет задавать соответствие C# функций SQL функциям. Рассмотрим это на примере метода `Sql.Between`:

```cs
[Sql.Expression("{0} BETWEEN {1} AND {2}", PreferServerSide = true, IsPredicate = true)]
public static bool Between<T>(this T value, T low, T high)
    where T : IComparable
{
    return value != null && value.CompareTo(low) >= 0 && value.CompareTo(high) <= 0;
}

[Sql.Expression("{0} BETWEEN {1} AND {2}", PreferServerSide = true, IsPredicate = true)]
public static bool Between<T>(this T? value, T? low, T? high)
    where T : struct, IComparable
{
    return value != null && value.Value.CompareTo(low) >= 0 && value.Value.CompareTo(high) <= 0;
}
```

Метод `Sql.Between` приводится к SQL утверждению `BETWEEN`, делается это посредствам атрибута [Sql.Expression](https://linq2db.github.io/api/LinqToDB.Sql.ExpressionAttribute.html). Используя этот атрибут вы можете расширять возможности `linq2db`. Рассмотрим некоторые свойства данного атрибута:

* `Configuration` - позволяет указать для какого диалекта применим данный синтаксис, по умолчанию синтаксис считается применимым во всех диалектах.
* `PreferServerSide`: установка этого значения говорит о том, что выражение желательно выполнить на стороне сервера, но если не выйдет, то можно и на клиенте.
* `ServerSideOnly`: требует выполнения метода на стороне сервера, в случае если это невозможно будет сгенерировано исключение.
* `InlineParameters`: параметры в выражении будут подставлены по значению.
* `IsPredicate`: говорит о том, что выражение само по себе является предикатом (возвращает булево значение). Это необходимо как в случае `Sql.Between` для того, что бы функция не приводилась к сравнению (грубо говоря выражение `Where(_ => true)`, для большинства SQL диалектов будет преобразовано в `WHERE 1=1`, что бы `linq2db` не добавляла к вызову функции данное сравнение следует выставить этот параметр в `true`);

`linq2db` не включает многих функций различных диалектов, это обусловлено необходимостью поддерживать "из коробки" выполнять одни и те же запросы на разных БД, однако, используя [Sql.Expression](https://linq2db.github.io/api/LinqToDB.Sql.ExpressionAttribute.html) и его наследников вы можете расширять функциональность своих проектов в соответствие с используемыми базами данных.

Рассмотрим пример реализации функции, проверяющей, что строка может быть приведена к числу:

```cs
[Sql.Expression("SqlServer","ISNUMERIC('-' + {0} + '.0e0')",             PreferServerSide = true)]
[Sql.Expression("SQLite",   "NOT {0} GLOB '*[^0-9]*' AND {0} LIKE '_%'", PreferServerSide = true)]
public static bool IsPositiveInteger<T>(T value)
{
    int checkDecimal = 0;
    if (int.TryParse(value.ToString(), out checkDecimal))
    {
        return checkDecimal.ToString() == value.ToString();
    }
    return false;
}
```

В данном примере мы предоставили две реализации метода - для MS SQL Server и для SQLite, как видите, для них реализация сильно отличается.

Для описания простых функций достаточно использовать [Sql.Function](https://linq2db.github.io/api/LinqToDB.Sql.FunctionAttribute.html), так для приведенного выше примера в случае MS SQL Server можно просто указать:

```cs
[Sql.Function("IsNumeric", PreferServerSide = true)]
public static bool IsNumeric(string s)
{
    double checkDecimal = 0;
    if (double.TryParse(value.ToString(), out checkDecimal))
    {
        return true;
    }
    return false;
}
```

С полным перечнем возможностей [Sql.Expression](https://linq2db.github.io/api/LinqToDB.Sql.ExpressionAttribute.html) можно ознакомиться в [исходном коде](https://github.com/linq2db/linq2db/tree/master/Source/LinqToDB/Sql/).

## Далее

Вас ждет [тестовое задание](test.md).
