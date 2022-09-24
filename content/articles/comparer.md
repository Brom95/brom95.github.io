---
title: "Сравнение в C#"
date: 2021-11-20T17:23:23+03:00
draft: false
---

## Предисловие

Автор знает, как работает сравнение в C#, достаточно четко представляет разницу между семантикой значимого и ссылочного типов, однако все еще находит эту статью хорошей и позволяющей чуть глубже заглянуть под капот.

---

Думаю, что каждый программист рано или поздно сталкивается с кодом, который работает «не так, как ты от него ожидаешь». Именно это и подтолкнуло меня к написанию следующей статьи, в которой я пытаюсь понять, почему Except в Linq работает так, как написан, а не так, как я хочу.

---

Что, по вашему мнению, должен вывести следующий код:

```csharp
var documentsDir = new DirectoryInfo(Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments));

FileInfo[] FirstDirrectoryFiles = documentsDir.GetFiles();
FileInfo[] SecondDirrectoryFiles = documentsDir.GetFiles();
foreach (FileInfo item in FirstDirrectoryFiles.Except(SecondDirrectoryFiles))
{
    Console.WriteLine(item.Name);
}
```

Я вот предположил, что ничего, потому что Except должен вычитать множество (IEnumerable) правого аргумента из множества (IEnumerable) левого аргумента. Однако, вопреки моим ожиданиям я получил:

![](/asserts/images/comparer_1.png)

Вне всякого сомнения — это не похоже на пустое множество. Давайте попробуем разобраться в том, почему так получается (результат в .NET 5 и в .NET 6 — эквивалентен). Чтобы понять, почему так происходит, и что можно с этим сделать обратимся к документации метода [Except](https://docs.microsoft.com/ru-ru/dotnet/api/system.linq.enumerable.except?view=net-6.0). Там действительно написано, что этот метод «Находит разность множеств, представленных двумя последовательностями» и имеет две перегрузки:
| | |
|----- | ----- |
| `Except<TSource>(IEnumerable<TSource>, IEnumerable<TSource>)`| Находит разность множеств, представленных двумя последовательностями, используя для сравнения значений компаратор проверки на равенство по умолчанию. |
| `Except<TSource>(IEnumerable<TSource>, IEnumerable<TSource>, IEqualityComparer<TSource>)` | Находит разность множеств, представленных двумя последовательностями, используя для сравнения значений указанный компаратор IEqualityComparer<T>. |

Обратите внимание на заявление о том, что для сравнения используется компаратор по умолчанию. Чтобы понять, почему наш код сработал именно так, как сработал, нам предстоит разобраться с поведением компаратора по умолчанию. Для этого я предлагаю проследовать на [https://github.com/dotnet/runtime/](https://github.com/dotnet/runtime/) и проанализировать работу метода Except.

Наша [точка входа](https://github.com/dotnet/runtime/blob/a7c69745a4e22757c69f65d2a750e2d6923e1186/src/libraries/System.Linq/src/System/Linq/Except.cs#L10):

```csharp
public static IEnumerable<TSource> Except<TSource>(this IEnumerable<TSource> first, IEnumerable<TSource> second)
{
    if (first == null)
    {
        ThrowHelper.ThrowArgumentNullException(ExceptionArgument.first);
    }

    if (second == null)
    {
        ThrowHelper.ThrowArgumentNullException(ExceptionArgument.second);
    }

    return ExceptIterator(first, second, null);
}
```

Метод проверяет, что получил два объекта с ненулевым указателем и передает аргументы в [ExceptIterator](https://github.com/dotnet/runtime/blob/a7c69745a4e22757c69f65d2a750e2d6923e1186/src/libraries/System.Linq/src/System/Linq/Except.cs#L79).

Строго говоря, можно было бы использовать и метод перегрузку, так как он работает буквально также:

```csharp
public static IEnumerable<TSource> Except<TSource>(this IEnumerable<TSource> first, IEnumerable<TSource> second, IEqualityComparer<TSource>? comparer)
{
    if (first == null)
    {
        ThrowHelper.ThrowArgumentNullException(ExceptionArgument.first);
    }

    if (second == null)
    {
        ThrowHelper.ThrowArgumentNullException(ExceptionArgument.second);
    }

    return ExceptIterator(first, second, comparer);
}
```

Почему перегрузка, а не параметр по умолчанию? Силами [сообщества](https://t.me/dotnettalks) было вынесено предположение, что дело в [этом](https://coding.abel.nu/2014/07/adding-an-overload-is-a-breaking-change/) и [этом](https://rules.sonarsource.com/csharp/RSPEC-2360).

Собственно код [ExceptIterator](https://github.com/dotnet/runtime/blob/a7c69745a4e22757c69f65d2a750e2d6923e1186/src/libraries/System.Linq/src/System/Linq/Except.cs#L79):

```csharp
private static IEnumerable<TSource> ExceptIterator<TSource>(IEnumerable<TSource> first, IEnumerable<TSource> second, IEqualityComparer<TSource>? comparer)
{
    var set = new HashSet<TSource>(second, comparer);

    foreach (TSource element in first)
    {
        if (set.Add(element))
        {
            yield return element;
        }
    }
}

```

Из элементов второго аргумента создается [множество](https://github.com/dotnet/runtime/blob/57bfe474518ab5b7cfe6bf7424a79ce3af9d6657/src/libraries/System.Private.CoreLib/src/System/Collections/Generic/HashSet.cs#L55). Элементы первого аргумента, которые удалось добавить в множество, возвращаются в качестве итератора.

Конструктор множества принимает интерфейс компаратора, который используется для сравнения элементов множества:  

![](/asserts/images/comparer_2.png)

Конкретно в нашем случае компаратор равен null, поэтому проваливаемся в свойство Default обобщенного класса EqualityComparer.

```csharp
public HashSet(IEqualityComparer<T>? comparer)
{
    if (comparer is not null && comparer != EqualityComparer<T>.Default) // first check for null to avoid forcing default comparer instantiation unnecessarily
    {
        _comparer = comparer;
    }

    // Special-case EqualityComparer<string>.Default, StringComparer.Ordinal, and StringComparer.OrdinalIgnoreCase.
    // We use a non-randomized comparer for improved perf, falling back to a randomized comparer if the
    // hash buckets become unbalanced.
    if (typeof(T) == typeof(string))
    {
        IEqualityComparer<string>? stringComparer = NonRandomizedStringEqualityComparer.GetStringComparer(_comparer);
        if (stringComparer is not null)
        {
            _comparer = (IEqualityComparer<T>?)stringComparer;
        }
    }
}
```
![](/asserts/images/comparer_3.png)

[Тут](https://github.com/dotnet/runtime/blob/57bfe474518ab5b7cfe6bf7424a79ce3af9d6657/src/coreclr/System.Private.CoreLib/src/System/Collections/Generic/EqualityComparer.CoreCLR.cs#L10), на мой скромный взгляд, все очевидно:

```csharp
public abstract partial class EqualityComparer<T> : IEqualityComparer, IEqualityComparer<T>
{
    // To minimize generic instantiation overhead of creating the comparer per type, we keep the generic portion of the code as small
    // as possible and define most of the creation logic in a non-generic class.
    public static EqualityComparer<T> Default { [Intrinsic] get; } = (EqualityComparer<T>)ComparerHelpers.CreateDefaultEqualityComparer(typeof(T));
}

```

Комментарий гласит, что с целью минимизации накладных расходов на создание универсального экземпляра для каждого типа код обобщенных классов уменьшают насколько это возможно за счет переноса его логики в необобщенный класс (в нашем случае это ComparerHelpers).

Давайте же посмотрим на то, как [происходит процесс создания компаратора по умолчанию](https://github.com/dotnet/runtime/blob/57bfe474518ab5b7cfe6bf7424a79ce3af9d6657/src/coreclr/System.Private.CoreLib/src/System/Collections/Generic/ComparerHelpers.cs#L116):

```csharp
internal static object CreateDefaultEqualityComparer(Type type)
{
    Debug.Assert(type != null && type is RuntimeType);

    object? result = null;
    var runtimeType = (RuntimeType)type;

    if (type == typeof(byte))
    {
        // Specialize for byte so Array.IndexOf is faster.
        result = new ByteEqualityComparer();
    }
    else if (type == typeof(string))
    {
        // Specialize for string, as EqualityComparer<string>.Default is on the startup path
        result = new GenericEqualityComparer<string>();
    }
    else if (type.IsAssignableTo(typeof(IEquatable<>).MakeGenericType(type)))
    {
        // If T implements IEquatable<T> return a GenericEqualityComparer<T>
        result = CreateInstanceForAnotherGenericParameter((RuntimeType)typeof(GenericEqualityComparer<string>), runtimeType);
    }
    else if (type.IsGenericType)
    {
        // Nullable does not implement IEquatable<T?> directly because that would add an extra interface call per comparison.
        // Instead, it relies on EqualityComparer<T?>.Default to specialize for nullables and do the lifted comparisons if T implements IEquatable.
        if (type.GetGenericTypeDefinition() == typeof(Nullable<>))
        {
            result = TryCreateNullableEqualityComparer(runtimeType);
        }
    }
    else if (type.IsEnum)
    {
        // The equality comparer for enums is specialized to avoid boxing.
        result = TryCreateEnumEqualityComparer(runtimeType);
    }

    return result ?? CreateInstanceForAnotherGenericParameter((RuntimeType)typeof(ObjectEqualityComparer<object>), runtimeType);
}
```

Пойдем по порядку:

1. Если тип аргумента byte, то возвращается компаратор специально для этого типа (ByteEqualityComparer)

2. Если это строка, то возвращается GenericEqualityComparer<string>();

3. Если тип реализует IEquatable, то на основе GenericEqualityComparer<string> возвращается GenericEqualityComparer для типа аргумента (даже не спрашивайте);

4. Если аргумент является универсальным типом (обобщением) и если этот универсальный тип Nullable<>,  [то на основе](https://github.com/dotnet/runtime/blob/57bfe474518ab5b7cfe6bf7424a79ce3af9d6657/src/coreclr/System.Private.CoreLib/src/System/Collections/Generic/ComparerHelpers.cs#L160) NullableEqualityComparer<int>  создается NullableEqualityComparer для типа аргумента;

5. Если аргумент – перечисление, [то на основе](https://github.com/dotnet/runtime/blob/57bfe474518ab5b7cfe6bf7424a79ce3af9d6657/src/coreclr/System.Private.CoreLib/src/System/Collections/Generic/ComparerHelpers.cs#L179) EnumEqualityComparer<> создается EnumEqualityComparer;

6. Во всех остальных случаях на основе ObjectEqualityComparer<object> создается ObjectEqualityComparer.

С помощью такого нехитрого кода (хотел было переписать через string builder, но, думаю, тут можно забить :D) попробуем понять, какой же из перечисленных случаев – наш:

```csharp
var documentsDir = new DirectoryInfo(Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments));
FileInfo[] result1 = documentsDir.GetFiles();
FileInfo[] result2 = documentsDir.GetFiles();

Type type = result1[0].GetType();
Console.Write(
    $"type == typeof(byte): {type == typeof(byte)}\n" +
    $"type == typeof(string): {type == typeof(string)}\n" +
    $"type.IsAssignableTo(typeof(IEquatable<>).MakeGenericType(type)): {type.IsAssignableTo(typeof(IEquatable<>).MakeGenericType(type))}\n" +
    $"type.IsGenericType: {type.IsGenericType}\n" +
    $"type.IsEnum: {type.IsEnum}\n\n");
```

Что и следовало ожидать:

![](/asserts/images/comparer_4.png)

Это значит, что теперь наш путь лежит в [ObjectEqualityComparer](https://github.com/dotnet/runtime/blob/57bfe474518ab5b7cfe6bf7424a79ce3af9d6657/src/libraries/System.Private.CoreLib/src/System/Collections/Generic/EqualityComparer.cs). Вот, собственно, и он:

```csharp
public sealed partial class ObjectEqualityComparer<T> : EqualityComparer<T>
{
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public override bool Equals(T? x, T? y)
    {
        if (x != null)
        {
            if (y != null) return x.Equals(y);
            return false;
        }
        if (y != null) return false;
        return true;
    }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public override int GetHashCode([DisallowNull] T obj) => obj?.GetHashCode() ?? 0;

    // Equals method for the comparer itself.
    public override bool Equals([NotNullWhen(true)] object? obj) =>
        obj != null && GetType() == obj.GetType();

    public override int GetHashCode() =>
        GetType().GetHashCode();
}

```

ObjectEqualityComparer определяет метод Equals для двух объектов следующим образом:

- Объекты равны, если они оба null (что логично);

- Объекты не равны, если только один из них null;

- Если оба объекта не null, то их эквивалентность определяется методом Equals «левого» аргумента.

Если обратиться к [документации](https://docs.microsoft.com/ru-ru/dotnet/api/system.io.fileinfo?view=net-6.0#methods), то можно увидеть, что у нашего класса FileInfo действительно есть метод Equals с пометкой «Унаследовано от Object». Что же, [туда](https://docs.microsoft.com/ru-ru/dotnet/api/system.object.equals?view=net-6.0#System_Object_Equals_System_Object_) и лежит наш путь! Там в секции «комментарии» мы можем узнать, что:

>Если текущий экземпляр является ссылочным типом, Equals(Object) метод проверяет равенство ссылок, а вызов Equals(Object) метода эквивалентен вызову ReferenceEquals метода. Равенство ссылок означает, что сравниваемые объектные переменные ссылаются на один и тот же объект.

На этом, казалось, можно было бы завершить наше путешествие, но давайте на последок придумаем, как заставить Except перестать показывать файлы в директории с моими документами.

Вариант из категории «пока так, потом пофикшу»:

```csharp
var documentsDir = new DirectoryInfo(Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments));
FileInfo[] result1 = documentsDir.GetFiles();
FileInfo[] result2 = documentsDir.GetFiles();
foreach (FileInfo item in result1
                                 .Select(i => i.FullName)
                                 .Except(result2.Select(i => i.FullName))
                                 .Select(i => new FileInfo(i)))
{
    Console.WriteLine(item.Name);
}

```

Мы, по сути, вызываем Except для двух IEnumerate<string>, а потом из результата снова собираем IEnumerate<FileInfo>.

В свежем .net6 еще можно воспользоваться [ExceptBy](https://docs.microsoft.com/en-us/dotnet/api/system.linq.enumerable.exceptby?view=net-6.0):

```csharp
var documentsDir = new DirectoryInfo(Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments));
FileInfo[] result1 = documentsDir.GetFiles();
FileInfo[] result2 = documentsDir.GetFiles();
foreach (FileInfo item in result1.ExceptBy(result2.Select(i => i.FullName), ks => ks.FullName))
{
    Console.WriteLine(item.Name);
}
```
Этот код уже выглядит приятнее и даже избавляет нас от необходимости разбирать и снова собирать изначальный массив, но можно сделать это иначе.

Следующей идеей, посетившей мою голову, было унаследовать FileInfo и переопределить Equals, но, к сожалению, FileInfo является запечатанным (sealed) классом, а это значит, что он не может быть унаследован, так что этот путь нам отрезан.

Подойдем к вопросу с другой стороны. Вспомним, что Equals имеет перегрузку, принимающую вторым аргументом IEqualityComparer, что позволяет нам создать что-то вроде такого решения:

```csharp
var documentsDir = new DirectoryInfo(Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments));
FileInfo[] result1 = documentsDir.GetFiles();
FileInfo[] result2 = documentsDir.GetFiles();

foreach (FileInfo item in result1.Except(result2, new CustomFileInfoComparer()))
{
    Console.WriteLine(item.Name);
}

public class CustomFileInfoComparer : IEqualityComparer<FileInfo>
{
    bool IEqualityComparer<FileInfo>.Equals(FileInfo? lhv, FileInfo? rhv)
       => lhv?.FullName == rhv?.FullName;
    int IEqualityComparer<FileInfo>.GetHashCode(FileInfo obj) => obj.FullName.GetHashCode();
}

```

Это решение хорошо подходит в том случае, если дальше по коду вам предстоит еще хотя бы раз сравнивать IEnumerable<FileInfo>.

На этом у меня все, надеюсь, что читатель нашел любопытным мой скромный труд.

Хочется выразить огромную благодарность моей жене за помощь в подготовке данного поста, а также сообществу [.NET Talks🎄](https://t.me/dotnettalks)