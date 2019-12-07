---
title: Маппинг
---

# Маппинг

Маппинг - отображение таблиц БД на объекты и их свойства, любая ORM должна знать какой объект какой таблице соответствует и какие колонки данной таблицы каким соответствуют каким полям объекта. Так же к маппингу относится и преобразование типов данных, между БД и объектами.

Разберемся с **базовыми** настройками маппинга в `linq2db`. По умолчанию действуют простые правила:

* Имя таблицы === Название класса.
* Имя колонки таблицы === Имя свойства (поля) класса.
* Типы данных у колонки и члена класса должны быть взаимно конвертируемы средствами .Net.
* Все публичные поля и свойства класса имеющие простые типы (`int`, `string`, и прочие примитивные типы, а так же `DateTime`, `DateTimeOffset`, `TimeSpan`, `Guid` и их Nullable версии) - являются колонками, прочие - нет.

Используя эти правила `linq2db` строит из LINQ запросов SQL запросы, рассмотрим это на простейшем примере:

```cs
public class Customer
{
    public long     Id               { get; set; } // integer
    public string   FullName         { get; set; } // varchar(50)
    public string   Phone            { get; set; } // varchar(15)
    public DateTime RegistrationTime { get; set; } // datetime
}

using (var db = new DataConnection())
{
    db.GetTable<Customer>().ToArray()
}
```

Для таких модели и запроса будет построен и выполнен следующий SQL:

```sql
-- * SQLite.Classic SQLite
SELECT
    [t1].[Id],
    [t1].[FullName],
    [t1].[Phone],
    [t1].[RegistrationTime]
FROM
    [Customer] [t1]
```

**NB**: обратите внимание, в построенном запросе перечислены все колонки **явно**, такой способ выборки вместо `SELECT * FROM Customer` является более эффективным как с точки зрения БД, так и с точки зрения материализации. Так же все имена в итоговом запросе - экранируются, что делает запрос более безопасным и надежным.

При использовании Т4 шаблонов мы получаем **гарантированно** валидную модель данных, именно поэтому это является рекомендуемым подходом.

И всё же в жизни бывает так, что нотации, используемые в БД и в C# не соответствуют друг другу, допустим, в БД может быть принято использование `snake_case` в именах полей и таблиц (кстати, конкретно этот случай Т4 шаблоны обработают и сгенерируют код в нотации C#). Как следствие нам необходимо иметь возможность вносить изменения в стандартные правила. Сделать мы это можем двумя способами:

1. Применив на классы и их члены специальные атрибуты
2. А так же в рантайме, используя [FluentMappingBuilder](https://linq2db.github.io/api/LinqToDB.Mapping.FluentMappingBuilder.html). Рассмотрение данного способа мы отложим.

## Атрибуты мапинга

Давайте внимательно посмотрим на код, сгенерированный T4 шаблоном:

```cs
[Table("Customer")]
public partial class Customer : IId
{
    [PrimaryKey, Identity] public long     Id               { get; set; } // integer
    [Column,     NotNull ] public string   FullName         { get; set; } // varchar(50)
    [Column,     NotNull ] public string   Phone            { get; set; } // varchar(15)
    [Column,     NotNull ] public DateTime RegistrationTime { get; set; } // datetime
}
```

Как вы видите, на класс и его свойства применены дополнительные атрибуты. Думаю, понять их назначение для вас не составит труда, но всё же рассмотрим каждый атрибут более подробно.

### Table

Докуметнация: [здесь](https://linq2db.github.io/api/LinqToDB.Mapping.TableAttribute.html).

В первую очередь атрибут позволяет задать имя таблицы а БД, а так же у него есть дополнительные свойства, которые позволяют задать:

* Имя базы данных и схемы ([Database](https://linq2db.github.io/api/LinqToDB.Mapping.TableAttribute.html#LinqToDB_Mapping_TableAttribute_Database), [Schema](https://linq2db.github.io/api/LinqToDB.Mapping.TableAttribute.html#LinqToDB_Mapping_TableAttribute_Schema)). Это может быть полезно, если на одном сервере БД вы работаете с несколькими базами. Если данные параметры установлены, то в запросах будет использовано **полное** имя таблицы. Это позволяет выполнять запросы, даже если соединение установлено с выбором другой БД.
* [IsColumnAttributeRequired](https://linq2db.github.io/api/LinqToDB.Mapping.TableAttribute.html#LinqToDB_Mapping_TableAttribute_IsColumnAttributeRequired) - позволяет указать явное требование на применения атрибута `Column` для полей и свойств класса, в этом случае колонками будут считаться только члены, помеченные данным атрибутом.

### Column

Докуметнация: [здесь](https://linq2db.github.io/api/LinqToDB.Mapping.ColumnAttribute.html).

Управляет мапингом колонок таблиц. Так же позволяет задать название колонки, и ещё множество дополнительных свойств. Имеет "побратима" - [NotColumn](https://linq2db.github.io/api/LinqToDB.Mapping.NotColumnAttribute.html) - указывающего, что данный член класса не является колонкой и не должен быть использован в запросах.

Рассмотрим основные свойства, которые позволяет задать данный аотрибут кроме имени:

* [IsIdentity](https://linq2db.github.io/api/LinqToDB.Mapping.ColumnAttribute.html#LinqToDB_Mapping_ColumnAttribute_IsIdentity) - указывает что данная колонка - автогенерируемый идентификатор, что исключает ее из запросов на вставку и обновление. Так же этот параметр можно задать использовав [IdentityAttribute](https://linq2db.github.io/api/LinqToDB.Mapping.IdentityAttribute.html).
* [IsPrimaryKey](https://linq2db.github.io/api/LinqToDB.Mapping.ColumnAttribute.html#LinqToDB_Mapping_ColumnAttribute_IsPrimaryKey) - указывает что колонка первичный ключ. Аналогично применению [IsPrimaryKey](https://linq2db.github.io/api/LinqToDB.Mapping.ColumnAttribute.html#LinqToDB_Mapping_ColumnAttribute_IsPrimaryKey).
* [SkipOnInsert](https://linq2db.github.io/api/LinqToDB.Mapping.ColumnAttribute.html#LinqToDB_Mapping_ColumnAttribute_SkipOnInsert) - при вставке целого объекта эта колонка будет пропущена в SQL запросе `INSERT`. Что может быть полезно для полей, получающих значение по умолчанию.
* [SkipOnUpdate](https://linq2db.github.io/api/LinqToDB.Mapping.ColumnAttribute.html#LinqToDB_Mapping_ColumnAttribute_SkipOnUpdate) - при обновлении целого объекта эта колонка будет пропущена в SQL запросе `UPDATE`. Это может быть полезно для неизменных колонок.

Прочие параметры вы можете посмотреть в документации самостоятельно.

### NotNull и Nullable

Докуметнация: [здесь](https://linq2db.github.io/api/LinqToDB.Mapping.NullableAttribute.html) и [здесь](https://linq2db.github.io/api/LinqToDB.Mapping.NotNullAttribute.html).

Как и следует из названия данные атрибуты позволяют задать, что колонка может или не может принимать значения `NULL`.

## Далее

[Управление соединениями](dataconnection.md)