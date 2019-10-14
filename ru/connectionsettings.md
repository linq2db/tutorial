---
title: Настройка соединения
---

# Настройка соединения

В данном разделе мы рассмотрим основные настройки соединения с БД. В первую очередь для соединения с БД необходимо задать [строку соединения](https://docs.microsoft.com/ru-ru/dotnet/api/system.data.idbconnection.connectionstring). Строки соединения используются в ADO .Net для определения параметров соединения с БД, в общем случае можно выделить следующие параметры, настраиваемые через данный механизм:

* Адрес сервера БД.
* Название базы данных, с которой осуществляется работа.
* Информация для аутентификации на сервере (логин и пароль, или другие параметр, например, возможность использовать учетную запись Windows для соединения с MS Sql Server).

Примеры различных строк соединения вы можете подглядеть [здесь](https://www.connectionstrings.com/).

По сути `linq2db` это надстройка над ADO .Net, и для настройки соединения необходимо задать правильную строку (строки для каждой конкретной БД следует использовать свои, в соответствие с документацией данной БД).

Важный момент заключается в том, что `linq2db` не использует строки напрямую, вместо этого используются "конфигурации" - именованный перечень параметров необходимых для соединения с БД:

* Поставщик данных (по сути указывает какая БД используется) - SQLite, MS Sql Server, Oracle и т.д.
* Строка соединения
* Дополнительные параметры, зависящие от целевого поставщика данных.

## Поставщики данных

Несколько слов уделим данной теме - `linq2db` поддерживает множество БД, конвертируя LINQ запрос в целевой SQL диалект. Кроме различных диалектов для разных БД так же используются разные ADO .Net клиенты, и `linq2db` должна создавать разные объекты реализующие [IDbConnection](https://docs.microsoft.com/ru-ru/dotnet/api/system.data.idbconnection), [IDbCommand](https://docs.microsoft.com/ru-ru/dotnet/api/system.data.idbcommand) и прочие ADO .Net классы. Так же, не смотря на "единый стандарт" ADO .Net разные клиенты обладают своими нюансами. Всё это многообразие требует изолировать знания о конечной БД в отдельном месте - поставщике данных.

Поставщик данных это объект, реализующий интерфейс [IDataProvider](https://linq2db.github.io/api/LinqToDB.DataProvider.IDataProvider.html) и содержащий всю необходимую информацию для взаимодействия с целевой БД через соответствующий ADO .Net клиент.

## Поставщик настроек

Настройка осуществляется через специального поставщика настроек [ILinqToDBSettings](https://linq2db.github.io/api/LinqToDB.Configuration.ILinqToDBSettings.html). Это обусловлено необходимостью реализовывать разные способы хранения настроек в зависимости от нужд приложения. Стандартные способы мы рассмотрим с вами далее, пока же уделим внимание данному интерфейсу для получения общих представлений о настройках.

Рассмотрим более подробно члены данного интерфейса:

* [DefaultDataProvider](https://linq2db.github.io/api/LinqToDB.Configuration.ILinqToDBSettings.html#LinqToDB_Configuration_ILinqToDBSettings_DefaultDataProvider) - имя поставщика данных по умолчанию.
* [DefaultConfiguration](https://linq2db.github.io/api/LinqToDB.Configuration.ILinqToDBSettings.html#LinqToDB_Configuration_ILinqToDBSettings_DefaultConfiguration) - имя конфигурации по умолчанию. Соответствует [IConnectionStringSettings.Name](https://linq2db.github.io/api/LinqToDB.Configuration.IConnectionStringSettings.html#LinqToDB_Configuration_IConnectionStringSettings_Name).
* [ConnectionStrings](https://linq2db.github.io/api/LinqToDB.Configuration.ILinqToDBSettings.html#LinqToDB_Configuration_ILinqToDBSettings_ConnectionStrings) - список конфигураций.
* [DataProviders](https://linq2db.github.io/api/LinqToDB.Configuration.ILinqToDBSettings.html#LinqToDB_Configuration_ILinqToDBSettings_DataProviders) - настройки различных поставщиков данных, эта секция заполняется при необходимости более тонкой настройки поставщика данных, либо в случае необходимости добавить внешний поставщик данных во время выполнения программы. Данный раздел используется редко

Наиболее интересным для нас является список конфигураций, давайте рассмотрим какие возможности предоставляет [IConnectionStringSettings](https://linq2db.github.io/api/LinqToDB.Configuration.IConnectionStringSettings.html):

* [Name](https://linq2db.github.io/api/LinqToDB.Configuration.IConnectionStringSettings.html#LinqToDB_Configuration_IConnectionStringSettings_Name) - название конфигурации, произвольная строка, должно быть уникальным. По этому имени осуществляется поиск конфигурации.
* [ProviderName](https://linq2db.github.io/api/LinqToDB.Configuration.IConnectionStringSettings.html#LinqToDB_Configuration_IConnectionStringSettings_ProviderName) - название поставщика данных, строка. Использование строк, вместо перечисления вызвано возможностью использовать не только "встроенные" поставщики данных, но и внешние (допустим, iSeries DB2 доступна только при установке специального пакета). Список встроенных поставщиков и их имена вы можете найти [здесь](https://linq2db.github.io/articles/general/databases.html)
* [ConnectionString](https://linq2db.github.io/api/LinqToDB.Configuration.IConnectionStringSettings.html#LinqToDB_Configuration_IConnectionStringSettings_ConnectionString) - собственно строка соединения.
* [IsGlobal](https://linq2db.github.io/api/LinqToDB.Configuration.IConnectionStringSettings.html#LinqToDB_Configuration_IConnectionStringSettings_IsGlobal) - признак того, что конфигурация является глобальной (определена на уровне `machine.config`)

Такой подход позволяет хранить несколько строк соединения в т.ч. к разным БД и переключаться между ними, используя название конфигурации.

Поставщик настроек задается как статическое свойство [DefaultSettings](https://linq2db.github.io/api/LinqToDB.Data.DataConnection.html#LinqToDB_Data_DataConnection_DefaultSettings).

При необходимости вы можете реализовать свой собственный поставщик настроек, что обеспечивает возможность хранить и управлять настройками в любом удобном вам виде.

### Использование конфигураций

После того как настройки заданы, нужно их использовать. Конфигурация задается в конструкторе объекта соединения и более не меняется (т.е. один экземпляр соединения работает с одной конфигурацией).

Рассмотрим соответствующие конструкторы:

* [new DataConnection()](https://linq2db.github.io/api/LinqToDB.Data.DataConnection.html#LinqToDB_Data_DataConnection__ctor) и [new DataContext()](https://linq2db.github.io/api/LinqToDB.DataContext.html#LinqToDB_DataContext__ctor) - создают соединение с конфигурацией по умолчанию.
* [new DataContext(string configurationString)](https://linq2db.github.io/api/LinqToDB.Data.DataConnection.html#LinqToDB_Data_DataConnection__ctor_System_String_) и [new DataContext(string configurationString)](https://linq2db.github.io/api/LinqToDB.DataContext.html#LinqToDB_DataContext__ctor_System_String_) - создают соединения с заданными конфигурациями.

Обратите внимание, `DataConnection` имеет множество конструкторов, позволяющих создать соединение с любыми параметрами, некоторые перегрузки так же принимают строки, и для того что бы избежать неоднозначности следует обращать внимание на название этих параметров:

* `configurationString` - название **конфигурации**
* `providerName` - название поставщика данных
* `connectionString` - строка соединения

В такое "изобилие" строк может внести путаницу, но есть достаточно простое правило - строка соединения не может быть передана объекту соединения без информации о поставщике данных, т.е. вместе со строкой соединения **нужно** передать либо название либо экземпляр класса поставщика. Это вполне логично, т.к. `linq2db` поддерживает множество баз данных, сама по себе строка не может сообщить как правильно работать с целевой БД, там просто нет этой информации, но, данным "знанием" обладает поставщик данных.

Полный перечень конструкторов соединения позволяет определить любые необходимые параметры, таким образом, использование поставщика параметров является ни необходимостью, а удобством, согласитесь, не очень удобно указывать все необходимые параметры при каждом обращении к БД, в большинстве же случаев, приложение использует только одну базу данных, что позволяет единожды в начале программы задать настройки по умолчанию и в дальнейшем о них не беспокоиться.

## Использование app.config или web.config

У .Net есть "врожденный" механизм [управления строками соединений](https://docs.microsoft.com/ru-ru/dotnet/framework/data/adonet/connection-strings-and-configuration-files) - хранение их в `app.config` для настольных приложений, и в `web.config` для веб приложений. По умолчанию `linq2db` поддерживает данную возможность тоже.

Рассмотрим пример, со строкой соединения для MS Sql Server:

```xml
<?xml version='1.0' encoding='utf-8'?>  
    <configuration>  
        <connectionStrings>  
            <add
                name             = "Northwind"
                providerName     = "SqlServer"
                connectionString = "Server=.\;Database=Northwind;Trusted_Connection=True;Enlist=False;" />  

            <add
                name             = "Logs"
                providerName     = "SQLite"
                connectionString = "Data Source=.\logs\logs.sqlite" />

        </connectionStrings>  
  </configuration>  
  ```

В данном случае мы создали конфигурацию две конфигурации:

* С именем "Northwind", поставщиком данных - "SqlServer" и, соответствующей строкой соединения.
* И с именем "Logs", аоставщиком данных "SQLite" и, соответствующей строкой соединения.

Создадим соединение с заданной конфигурацией:

```cs
using (var db = new DataConnection("Northwind"))
{
    //
}
```

Так же мы можем задать [DataConnection.DefaultConfiguration](https://linq2db.github.io/api/LinqToDB.Data.DataConnection.html#LinqToDB_Data_DataConnection_DefaultConfiguration) и использовать конструктор без параметров:

```cs
DataConnection.DefaultConfiguration = "Northwind";

//...

using (var db = new DataConnection())
{
    //...
}
```

В случае, если нам необходимо что то записать в базу данных с логами, мы можем сделать так:

```cs
using (var db = new DataConnection("Logs"))
{
    //
}
```

## Определение конфигурации в коде

### Добавление конфигурации

`app.config` - хороший инструмент для разработчика, но в продакшене он имеет ряд существенных недостатков, которые делают его использование не очень удобным. Да и в ряде других случаев может возникнуть необходимость настраивать конфигурации в момент выполнения программы. Есть два способа сделать это:

* Реализация [ILinqToDBSettings](https://linq2db.github.io/api/LinqToDB.Configuration.ILinqToDBSettings.html) - это хороший, инструмент, но у него есть одна особенность - настройки из него вычитываются только **единожды**, это связано с необходимостью минимизировать все возможные издержки при создании экземпляра соединения.
* Использование специальных методов `DataConnection` для записи конфигураций - этот способ позволяет в том числе обновлять конфигурации в ходе выполнения программы.

Рассмотрим более подробно второй способ:

* Методы [AddOrSetConfiguration](https://linq2db.github.io/api/LinqToDB.Data.DataConnection.html#LinqToDB_Data_DataConnection_AddOrSetConfiguration_System_String_System_String_System_String_) и [AddConfiguration](https://linq2db.github.io/api/LinqToDB.Data.DataConnection.html#LinqToDB_Data_DataConnection_AddConfiguration_System_String_System_String_LinqToDB_DataProvider_IDataProvider_) позволяют добавить или обновить конфигурацию.
* [DefaultConfiguration](https://linq2db.github.io/api/LinqToDB.Data.DataConnection.html#LinqToDB_Data_DataConnection_DefaultConfiguration) позволяет задать конфигурацию по умолчанию.

Рассмотрим это [на примере](https://github.com/linq2db/tutorial.sources/tree/configuration):

```cs
static void Main(string[] args)
{
    var path = System.IO.Path.GetFullPath(@"..\..\..\..\DB\database.sqlite");

    // Зададим конфигурацию
    DataConnection.AddOrSetConfiguration("*", $"Data Source={path};", ProviderName.SQLiteClassic);

    // Зададим конфигурацию по умолчанию
    DataConnection.DefaultConfiguration = "*";

    // Создадим соединение
    using (var db = new DataConnection())
    {
        // Создадим объект для выполнения запроса
        IQueryable<Customer> customersTable = db.GetTable<Customer>();

        // Выполним запрос
        Customer[] customers = customersTable.ToArray();

        // Выведем результаты запроса
        foreach (var c in customers)
            Console.WriteLine($"{c.FullName}: {c.Phone}");
    }

    Console.ReadKey();
}
```

### Настройка через конструктор

`DataConnection` предоставляет множество [конструкторов](https://linq2db.github.io/api/LinqToDB.Data.DataConnection.html#constructors), позволяющих настроить соединение не используя поставщик конфигурации, рассмотрим основные:

* [DataConnection(string providerName, string connectionString)](https://linq2db.github.io/api/LinqToDB.Data.DataConnection.html#LinqToDB_Data_DataConnection__ctor_System_String_System_String_) - позволяет создать соединение, используя имя поставщика данных и строку соединения.
* [DataConnection(IDataProvider dataProvider, string connectionString)](https://linq2db.github.io/api/LinqToDB.Data.DataConnection.html#LinqToDB_Data_DataConnection__ctor_LinqToDB_DataProvider_IDataProvider_System_String_) - позволяет создать соединение, используя **экземпляр** поставщика данных и строку соединения.

Рассмотрим это [на примере](https://github.com/linq2db/tutorial.sources/tree/configuration_contructor):

```cs
static void Main(string[] args)
{
    var path = System.IO.Path.GetFullPath(@"..\..\..\..\DB\database.sqlite");

    // Создадим соединение
    using (var db = new DataConnection(ProviderName.SQLiteClassic, $"Data Source={path};"))
    {
        // Создадим объект для выполнения запроса
        IQueryable<Customer> customersTable = db.GetTable<Customer>();

        // Выполним запрос
        Customer[] customers = customersTable.ToArray();

        // Выведем результаты запроса
        foreach (var c in customers)
            Console.WriteLine($"{c.FullName}: {c.Phone}");
    }

    Console.ReadKey();
}
```

## Особенности .Net Core

Для .Net Core проектов не поддерживается использование `app.config`, но вы можете использовать все остальные способы определения конфигурации.

Для примера рассмотрим создание собственного поставщика конфигурации, исходный код доступен [здесь](https://github.com/linq2db/tutorial.sources/tree/configuration_settingsprovider).

## Таймаут запроса

Одним из немаловажных параметров при выполнении запросов к БД является [таймаут запроса](https://docs.microsoft.com/ru-ru/dotnet/api/system.data.idbcommand.commandtimeout), этот параметр определяет через какой промежуток времени безрезультатное ожидание результатов запроса будет прервано и будет выброшено исключение.

Данный параметр настраивается через свойство [DataConnection.CommandTimeout](https://linq2db.github.io/api/LinqToDB.Data.DataConnection.html#LinqToDB_Data_DataConnection_CommandTimeout) и может принимать одно из следующих значений:

* Отрицательное число - данная настройка не будет применена, будет использован таймаут по умолчанию ADO .Net клиента (соответствует поведению по умолчанию).
* Ноль - бесконечный таймаут.
* Положительное число - время ожидания запроса в секундах.

Данный параметр настраивается для каждого соединения и действует до его смены или высвобождения. Изменение данного параметра не повлияет на уже выполняемый запрос, и будет действовать только на последующие запросы. Изменять данный параметр рекомендуется **только** в том случае, если вы уверены что здесь, сейчас и для данного запроса это необходимый шаг.

## Исходный код к статье

* [Добавление и изменение конфигураций в ходе выполнения программы](https://github.com/linq2db/tutorial.sources/tree/configuration)
* [Настройка соединения через конструктор](https://github.com/linq2db/tutorial.sources/tree/configuration_contructor)
* [Использование своего поставщика конфигураций](https://github.com/linq2db/tutorial.sources/tree/configuration_settingsprovider)

## Далее

[Выполнение запросов](queries.md)