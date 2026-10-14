<a id="csharp9"></a>
## C# 9：简化入口与数据模型

### 9.1 record：用简短声明表达数据

`record` 是适合表示数据的引用类型。编译器提供成员值比较、哈希值、可读的 `ToString()` 等；位置参数形式还会生成属性和解构方法。

```csharp
// 类型声明
public record Student(string Name, int Grade);

// 调用
var first = new Student("小林", 3);
var second = new Student("小林", 3);
Console.WriteLine(first == second); // True

var nextYear = first with { Grade = 4 };
Console.WriteLine(first.Grade);     // 3，原对象没有被修改
Console.WriteLine(nextYear.Grade);  // 4

var (name, grade) = first;
Console.WriteLine($"{name}：{grade} 年级");
```

**通俗理解：** 普通 class 默认更关心“是不是同一个对象”，record 更关心“对象中记录的数据是否相同”。两个分别创建的 `Student` 只要成员值相同，`==` 就认为它们相等。`with` 可以理解为“复印一份，再改几个地方”。

**属性能否直接修改：** 对于 `public record Student(string Name, int Grade);` 这种位置参数形式的 record（引用类型），如果没有自行声明对应属性，编译器会自动生成 `get; init;` 属性。因此，初始化时可以赋值，但初始化结束后不能直接重新赋值；`with` 则会创建一个新对象，在新对象的初始化阶段设置属性。

```csharp
var student = new Student("小林", 3) { Grade = 4 }; // 初始化时可以赋值
// student.Grade = 5; // 编译错误：初始化结束后不能直接赋值
var updated = student with { Grade = 5 }; // 创建新对象，student.Grade 仍为 4
```

record 本身不强制不可变。若自行声明带有 `get; set;` 的属性，创建后就可以直接修改该属性。这里的默认 `init` 规则针对位置式 record 引用类型；C# 10 的普通位置式 `record struct` 默认生成 `get; set;` 属性，详见 10.1 节。此外，`init` 只限制属性重新赋值，不会禁止修改属性所引用的可变对象的内容。

`with` 默认是**浅复制**，不会自动复制成员引用的对象：

```csharp
public record Team(string Name, List<string> Members);

// 方法内调用
var first = new Team("开发组", new List<string> { "小林" });
var second = first with { Name = "研发组" };

second.Members.Add("小陈");
Console.WriteLine(first.Members.Count); // 2，两个 record 共用同一个列表
```

### 9.2 init：只在初始化阶段设置属性

以前的 `set` 允许创建后继续修改；`init` 允许对象初始化器赋值，但禁止普通调用代码在初始化结束后重新赋值。

```csharp
public class Product
{
    public string Name { get; init; } = "";
    public decimal Price { get; init; }
}

// 调用
var product = new Product
{
    Name = "键盘",
    Price = 299m
};

// product.Name = "鼠标"; // 编译错误
```

**通俗理解：** `set` 像“以后随时可以修改”，`init` 像“登记时填写，登记结束后封存”。它适合名称、订单编号和创建时间等不希望在对象创建后被随意改变的信息。

普通 class 和 struct 都可以使用 `init`，并不要求是 record。它只限制“何时可以赋值”，并不要求调用者“一定赋值”：`new Product()` 仍然合法，属性会采用默认值或初始化值。强制调用者赋值是 C# 11 `required` 的职责。

### 9.3 顶级语句：入口程序可以直接写

```csharp
using System;

Console.WriteLine("开始运行");
Console.WriteLine($"参数数量：{args.Length}");

await System.Threading.Tasks.Task.Delay(100);
Console.WriteLine("异步等待结束");
```

**通俗理解：** 编译器替你生成了 `Program` 和 `Main`，所以小程序不用先写一层类和入口方法。它只是省略入口样板代码，不会让 C# 变成解释执行语言。

一次编译只能有一个源文件包含顶级语句，且顶级语句必须写在该文件的命名空间和类型声明之前。顶级语句可以直接使用 `args`、`await` 和适当的 `return`。见[顶级语句规范](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-9.0/top-level-statements.md)。

### 9.4 模式匹配增强：范围、组合、否定和括号

| 新能力 | 示例 | 含义 |
| --- | --- | --- |
| 关系模式 | `score is >= 60` | 与常量比较 |
| `and` | `score is >= 60 and <= 100` | 同时满足 |
| `or` | `day is 6 or 7` | 满足任意一个 |
| `not` | `value is not null` | 不符合某个模式 |
| 括号模式 | `value is int and (< 0 or > 100)` | 明确组合顺序 |
| 类型模式 | `value switch { int => "整数", _ => "其他" }` | 只检查类型，不必声明临时变量 |

例如，可以把成绩范围直接写成接近自然语言的分类规则：

```csharp
static string GetResult(int score) => score switch
{
    < 0 or > 100 => "无效成绩",
    >= 90 => "优秀",
    >= 60 and < 90 => "合格",
    _ => "不合格"
};

Console.WriteLine(GetResult(95));  // 优秀
Console.WriteLine(GetResult(70));  // 合格
Console.WriteLine(GetResult(120)); // 无效成绩
```

同样的分类逻辑，在 **C# 8** 中可以用 `switch` 表达式配合 `when` 条件：

```csharp
static string GetResult(int score) => score switch
{
    var n when n < 0 || n > 100 => "无效成绩",
    var n when n >= 90 => "优秀",
    var n when n >= 60 && n < 90 => "合格",
    _ => "不合格"
};
```


**通俗理解：** 以前通常是写一串“怎样计算出 true 或 false”，模式匹配则可以直接描述“这个值应该符合什么形状或范围”。这些模式可以用在 `is`、`switch` 语句和 `switch` 表达式等位置。

`and/or/not` 在这里组合的是模式，不能在普通布尔表达式中随意替换 `&&/||/!`；关系模式右侧也必须是适用的常量。

### 9.5 函数指针：直接保存可调用的方法地址

```csharp
public static class NativeStyle
{
    private static int Double(int x) => x * 2;

    public static unsafe int Run()
    {
        delegate*<int, int> operation = &Double;
        return operation(7); // 14
    }
}
```

项目还需开启不安全代码：

```xml
<PropertyGroup>
  <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
</PropertyGroup>
```

**通俗理解：** 普通委托像一个安全、方便的“方法对象”，函数指针则更接近 C/C++ 中直接保存函数地址的做法。`delegate*<参数类型, 返回类型>` 的最后一项是返回类型，它不是普通委托。

函数指针支持托管和非托管调用约定，适合本机互操作和极端性能敏感代码。普通事件、按钮回调和业务策略通常继续使用 `Action`、`Func` 或自定义委托，因为它们更安全、更直接。

### 9.6 分部方法增强：声明和实现可以承担更多工作

```csharp
public partial class PriceRules
{
    public partial decimal GetDiscount(int level);
}

public partial class PriceRules
{
    public partial decimal GetDiscount(int level) => level >= 3 ? 0.9m : 1m;
}
```

```csharp
// 调用
var rules = new PriceRules();
Console.WriteLine(rules.GetDiscount(4)); // 0.9
```

**通俗理解：** 一部分代码规定“必须有这个方法”，另一部分代码提供具体实现。这很适合开发者写声明、代码生成器自动补实现的场景。

C# 9 的分部方法可以显式指定访问级别、返回非 `void` 值、使用 `out` 等。采用这些增强形式时必须有匹配的声明和实现，而且包含它们的类型也必须是 `partial`；

### 9.7 目标类型 new：类型明确时不再重复写

```csharp
List<string> names = new();
Dictionary<string, int> stock = new() { ["键盘"] = 8 };
```

目标类型也可以来自方法参数或返回值：

```csharp
// 类型声明
public record Student(string Name, int Grade);

// 以下方法放在适当的类型中
static void Register(Student student) => Console.WriteLine(student.Name);
static Student CreateGuest() => new("访客", 1);
```

```csharp
// 调用
Register(new("小林", 3));
Student guest = CreateGuest();
```

**通俗理解：** 左侧、方法参数或返回值已经说明“要创建什么”，右侧就不用再重复一次完整类型名。目标类型不只可以来自赋值左侧，也可以来自参数、返回值等上下文。

它必须依赖上下文提供目标类型，因此 `var names = new();` 仍然不行：`var` 等待右侧提供类型，而 `new()` 又等待左侧提供类型，双方都没有足够信息。目标类型明确也不代表一定能实例化，例如接口和抽象类仍然遵守正常的对象创建规则。

### 9.8 static 匿名函数：明确禁止捕获外部状态

```csharp
Func<int, int> square = static x => x * x;

int factor = 3;
// Func<int, int> multiply = static x => x * factor;
// 编译错误：static Lambda 不能捕获外部局部变量 factor
```

**通俗理解：** 普通 Lambda 可以“顺手带走”外部变量，`static` Lambda 则明确承诺“我不依赖当前对象和外部局部状态”。静态 Lambda 或匿名方法不能捕获外部局部变量、参数或 `this`，但可以引用允许访问的静态成员和常量。

### 9.9 目标类型条件表达式：两条分支可以借助目标类型统一

```csharp
bool found = false;
int? result = found ? 42 : null;

bool success = false;
object message = success ? "操作成功" : 404;
Console.WriteLine(message); // 404
```

**通俗理解：** 条件表达式的两个候选结果自己无法确定统一类型时，接收方可以告诉它们“都转换成我要求的类型”。例如字符串和整数都能转换成 `object`，`42` 和 `null` 都能转换成 `int?`。

两个分支必须都能隐式转换到上下文要求的目标类型，并不是任意类型都能组合。使用 `var` 时没有这种目标信息，因此 `var message = success ? "操作成功" : 404;` 不能编译；如果两个分支本身就有明确共同类型，则仍可使用 `var`。

### 9.10 协变返回类型：重写方法可以返回更具体的类型

```csharp
public class Animal { }
public class Cat : Animal { }

public class Factory
{
    public virtual Animal Create() => new Animal();
}

public class CatFactory : Factory
{
    public override Cat Create() => new Cat();
}

// 调用
var factory = new CatFactory();
Cat cat = factory.Create(); // 不需要再从 Animal 强制转换
```

**通俗理解：** 基类承诺“我会生产某种动物”，子类可以给出更精确的承诺：“我生产的一定是猫”。调用者通过 `CatFactory` 访问时可以直接得到 `Cat`，不用再手动转换。

### 9.11 扩展 GetEnumerator：已有类型也能参与 foreach

假设已有类型只表示一个数字范围，但没有实现枚举接口：

```csharp
public class NumberRange
{
    public int Start { get; }
    public int End { get; }

    public NumberRange(int start, int end)
    {
        Start = start;
        End = end;
    }
}

public sealed class NumberEnumerator
{
    private int _next;
    private readonly int _end;

    public int Current { get; private set; }

    public NumberEnumerator(int start, int end)
    {
        _next = start;
        _end = end;
    }

    public bool MoveNext()
    {
        if (_next > _end)
            return false;

        Current = _next++;
        return true;
    }
}

public static class NumberRangeExtensions
{
    public static NumberEnumerator GetEnumerator(this NumberRange range)
        => new NumberEnumerator(range.Start, range.End);
}

// 调用
var range = new NumberRange(3, 5);
foreach (int number in range)
    Console.WriteLine(number); // 依次输出 3、4、5
```

**通俗理解：** `foreach` 会寻找符合约定的 `GetEnumerator()`。即使不能修改原类型，也可以在外部用扩展方法补上这个入口。

扩展 `GetEnumerator()` 是后备方案；只有实例枚举模式和适用接口没有解决枚举时，查找过程才考虑它，不能用它覆盖类型已有的正常枚举行为。返回的枚举器仍须提供合适的公共 `MoveNext()` 和可读 `Current` 属性等成员。

### 9.12 Lambda 弃元参数：明确表示参数不用

```csharp
Action<int, string> notify = (_, _) => Console.WriteLine("收到通知");
notify(100, "订单已创建");

// 只有一个名为 _ 的参数时，它仍是可以引用的普通参数：
Func<int, int> identity = _ => _;
Console.WriteLine(identity(5)); // 5
```

**通俗理解：** 委托规定 Lambda 必须接收这些参数，但当前逻辑完全不关心它们，所以用 `_` 明确表示忽略。一个 Lambda 或匿名方法的参数列表中出现多个同名 `_` 时，它们才是弃元，不能在方法体内引用。

为了兼容旧代码，只有一个 `_` 参数时，它就是普通参数名；在 `(_, message) => ...` 中也只有一个 `_`，所以它仍是普通参数，而不是弃元。

### 9.13 局部函数可以使用特性

```csharp
using System.Diagnostics;

static void ProcessOrder()
{
    [Conditional("DEBUG")]
    static void DebugMessage(string message)
    {
        Console.WriteLine($"调试信息：{message}");
    }

    DebugMessage("开始处理订单");
    Console.WriteLine("订单处理完成");
}
```

定义了 `DEBUG` 编译符号时会调用 `DebugMessage`；没有定义时，这次调用会被省略。

**通俗理解：** 局部函数是定义在方法内部的辅助函数。C# 9 允许像普通方法一样，在它以及适用的参数、返回值、类型参数等位置附加特性。特性是否产生效果仍由具体特性的规则决定，不是贴上任意特性都会自动执行逻辑。

<a id="csharp10"></a>
## C# 10：减少重复代码

### 10.1 record struct：值类型也有 record 的便利

```csharp
public record struct GridPoint(int Row, int Column);
public readonly record struct Size(int Width, int Height);
```

它们是值类型，提供成员值相等比较、格式化和解构。普通位置式 `record struct` 的生成属性默认可读写；`readonly record struct` 的相应属性采用仅初始化形式。同时可写 `record class` 明确表示引用类型。见 [record struct 规范](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-10.0/record-structs.md)。

### 10.2 普通 struct 和匿名类型也支持 with

```csharp
var original = new { Name = "小周", Level = 1 };
var updated = original with { Level = 2 };
```

`with` 不再仅用于 C# 9 的 record；普通结构体也能先复制再修改可赋值成员。普通 class 仍不能因此直接使用 `with`。见 [with 的结构体扩展规则](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-10.0/record-structs.md#allow-with-expression-on-structs)。

### 10.3 struct 可以声明无参构造函数和实例字段初始化器

```csharp
public struct RetryOptions
{
    public int Count = 3;
    public RetryOptions() { }
}

// 调用
var configured = new RetryOptions();  // Count = 3
var zeroed = default(RetryOptions);   // Count = 0
```

显式无参构造函数必须是 `public`。`default(S)` 和创建结构体数组时的元素清零，不会执行这个自定义无参构造函数；不要把 `new S()` 和 `default(S)` 当成永远等价。见[结构体构造函数规范](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-10.0/parameterless-struct-constructors.md)。

### 10.4 global using：一个项目统一导入命名空间

```csharp
// 例如 GlobalUsings.cs
global using System;
global using System.Collections.Generic;
global using static System.Math;
```

这些导入作用于同一次编译的所有源码文件，不再每个文件重复声明；也支持别名。它不会自动影响另一个项目。`.NET 6` 模板的隐式 using 则是 SDK 自动生成部分全局导入，属于配套工具行为。见 [global using 规范](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-10.0/GlobalUsingDirective.md)。

### 10.5 文件范围命名空间：去掉最外层大括号

```csharp
namespace Shop.Orders;

public class OrderService
{
    public void Submit() { }
}
```

整个文件的类型都位于该命名空间，少一层缩进。一个文件只能有一个文件范围命名空间，不能在同一文件混用块式命名空间或顶级语句。见[文件范围命名空间规范](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-10.0/file-scoped-namespaces.md)。

### 10.6 扩展属性模式：嵌套成员可以连着写

```csharp
// 原有写法
if (order is { Customer: { Address: { City: "杭州" } } }) { }

// C# 10 写法
if (order is { Customer.Address.City: "杭州" }) { }
```

表达的匹配要求相同；中间对象为 `null` 时不匹配，不会因为这条属性模式直接触发空引用异常。见[扩展属性模式规范](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-10.0/extended-property-patterns.md)。

### 10.7 Lambda 三项增强：自然类型、显式返回类型、特性

```csharp
// 参数类型明确，编译器可以推断委托类型
var doubleValue = (int number) => number * 2;

// 显式指定返回类型，让分支结果统一为 object
var describe = object (bool isNumber) => isNumber ? 7 : "七";

// Lambda 可以标注特性
var oldOperation = [Obsolete("仅用于演示")] () => 1;
```

自然类型推断也覆盖能够唯一确定签名的方法组。`var f = x => x + 1;` 仍缺少参数类型信息。特性可用于 Lambda、参数及返回值等适当位置；是否在调用时产生特定效果取决于对应特性的规则。见 [Lambda 改进规范](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-10.0/lambda-improvements.md)。

### 10.8 常量内插字符串

```csharp
const string Area = "orders";
const string Route = $"/api/{Area}";
```

当内插部分都是符合要求的常量字符串时，结果可以用于 `const`。并不是所有常量表达式都可以插入：`const string S = $"{123}";` 不符合这种常量字符串规则。见[常量内插字符串规范](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-10.0/constant_interpolated_strings.md)。

### 10.9 内插字符串处理程序：让格式化按需进行

库作者可以定义插值处理程序，控制 `$"用户 {id}"` 如何拼接、格式化，甚至在无需记录日志时跳过插值内容的求值。调用方通常仍使用熟悉的 `$"..."` 语法。

这是减少临时字符串和无效计算的机制，但前提是被调用 API 使用相应处理程序；不能推断任意日志方法都会自动跳过计算。

下面是一个完整示例，可在 C# 10 + .NET 6 或更高版本中运行。日志关闭时，连插值中的 `Calculate()` 都不会执行：

```csharp
using System;
using System.Runtime.CompilerServices;

var logger = new Logger { Enabled = false };
logger.Log($"结果是 {Calculate()}"); // 不执行 Calculate，没有输出

logger.Enabled = true;
logger.Log($"结果是 {Calculate()}"); // 输出“正在计算”和“结果是 123”

static int Calculate()
{
    Console.WriteLine("正在计算");
    return 123;
}

public class Logger
{
    public bool Enabled { get; set; }

    public void Log(
        [InterpolatedStringHandlerArgument("")]
        LogHandler handler)
    {
        if (Enabled)
            Console.WriteLine(handler.GetText());
    }
}

[InterpolatedStringHandler]
public ref struct LogHandler
{
    private DefaultInterpolatedStringHandler _builder;

    public LogHandler(
        int literalLength,
        int formattedCount,
        Logger logger,
        out bool shouldAppend)
    {
        shouldAppend = logger.Enabled;
        _builder = default;

        if (shouldAppend)
        {
            _builder = new DefaultInterpolatedStringHandler(
                literalLength, formattedCount);
        }
    }

    public void AppendLiteral(string text)
        => _builder.AppendLiteral(text);

    public void AppendFormatted<T>(T value)
        => _builder.AppendFormatted(value);

    public string GetText()
        => _builder.ToStringAndClear();
}
```

`LogHandler` 决定“要不要构建字符串”，`DefaultInterpolatedStringHandler` 是 .NET 提供的现成工具，负责“怎样追加文字、格式化值并构建字符串”。普通的 `Log(string message)` 会先计算插值并生成字符串，再进入方法；本例的 `Log` 接收处理程序，让编译器能在插值求值前判断是否需要构建。

- `[InterpolatedStringHandler]` 是固定特性，标记处理程序类型；`LogHandler` 这个类型名可以自定义。
- `[InterpolatedStringHandlerArgument("")]` 中的空字符串表示把调用对象 `logger` 传给处理程序构造函数。
- 构造函数前两个 `int` 参数依次接收固定文字长度和插值数量；本例中分别是 `4`（“结果是 ”）和 `1`。
- 构造函数末尾的 `out bool` 是可选的条件开关。参数名 `shouldAppend` 可以自定义，但编译器按这个参数的类型和位置识别它。
- `AppendLiteral`、`AppendFormatted` 是编译器查找的固定方法名，分别处理固定文字和插入值；`GetText()`、`_builder` 则是本例自行定义的名称。

对于 `logger.Log($"结果是 {Calculate()}")`，编译器生成的逻辑可以近似理解为下面这样：

```csharp
var handler = new LogHandler(4, 1, logger, out bool shouldAppend);

if (shouldAppend)
{
    handler.AppendLiteral("结果是 ");
    handler.AppendFormatted(Calculate());
}

logger.Log(handler);
```

因此，`shouldAppend` 为 `false` 时，`AppendLiteral`、`Calculate()` 和 `AppendFormatted` 都不会执行；`Log` 方法本身仍会被调用，只是它检查到日志关闭后不读取字符串。如果手动调用 `handler.AppendLiteral(...)`，则不会自动获得这个条件判断。

### 10.10 record 可以封闭 ToString 的重写

```csharp
public record Ticket(int Id)
{
    public sealed override string ToString() => $"Ticket:{Id}";
}
```

后续派生 record 不再生成自己的 `ToString()` 覆盖它，方便整个继承体系统一显示格式。

### 10.11 明确赋值和空状态分析更准确

```csharp
// 假设 map 为 Dictionary<string, int>?
if (map?.TryGetValue("stock", out var count) == true)
{
    Console.WriteLine(count);
}
```

编译器能识别更多“进入分支就表示输出变量已赋值、对象非空”的情况，减少错误诊断或不必要的绕行代码。这是编译期分析改进，不是新增运行时判空。

### 10.12 解构时可以混合旧变量和新变量

```csharp
int code = 0;
(code, string message) = (200, "成功");
```

一次解构可以给已有变量赋值，同时声明新变量。此前往往要拆开处理。

### 10.13 CallerArgumentExpression：获取调用时写的表达式

```csharp
using System.Runtime.CompilerServices;

static void Check(
    bool condition,
    [CallerArgumentExpression("condition")] string expression = "")
{
    if (!condition)
        throw new ArgumentException($"条件不满足：{expression}");
}

int quantity = -1;
Check(quantity > 0); // 异常信息包含 quantity > 0
```

编译器将调用处的源码表达式文字填入参数，方便断言和参数校验。它得到的是表达式文字，不是运行时反射得到的变量历史。见[调用方表达式规范](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-10.0/caller-argument-expression.md)。

<a id="csharp11"></a>
## C# 11：字符串、泛型数学与引用安全

### 11.1 原始字符串：大段文本不必反复转义

```csharp
string json = """
    {
      "name": "键盘",
      "path": "C:\\Data"
    }
    """;

int count = 5;
string result = $$"""{"count": {{count}}}""";
```

三个或更多连续双引号构成分隔符，内部双引号、反斜杠通常可以直接写。多行内容的缩进由结尾分隔符的位置决定。插值时 `$` 的数量决定要用几层大括号包住表达式，上例中 `{{count}}` 才是插值，单层 JSON 大括号保留原样。

### 11.2 普通插值字符串的表达式内部可以换行

```csharp
int score = 85;
string message = $"等级：{score switch
{
    >= 90 => "优秀",
    >= 60 => "合格",
    _ => "待提高"
}}";
```

换行发生在 `{ ... }` 表达式里，便于写复杂条件，并不意味着生成的字符串自动包含这些排版换行。见[插值表达式换行规范](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-11.0/new-line-in-interpolation.md)。

### 11.3 列表模式：按集合的形状匹配

```csharp
int[] numbers = { 10, 20, 30, 40 };

bool beginsWithTen = numbers is [10, ..];
bool hasTwoItems = numbers is [_, _];
bool endsWithForty = numbers is [.., 40];

if (numbers is [var first, .. var middle, var last])
{
    Console.WriteLine($"首项 {first}，末项 {last}");
}
```

`..` 匹配零个或多个元素，一个列表模式最多有一个这样的切片。

### 11.4 Span<char> 可以匹配常量字符串

```csharp
using System;

// 直接查看请求中的前四个字符，不创建子字符串
string request = "POST /orders";
ReadOnlySpan<char> verb = request.AsSpan(0, 4);
ReadOnlySpan<char> expected = "POST".AsSpan();

Console.WriteLine(verb == expected); // False：指向不同的内存位置
Console.WriteLine(verb is "POST");   // True：字符内容恰好是 POST（C# 11）

// C# 11 之前，比较内容可以使用 SequenceEqual
Console.WriteLine(verb.SequenceEqual(expected)); // True
```

`Span<char>` 和 `ReadOnlySpan<char>` 可以用字符串常量模式检查内容，不必先创建 `string`。这里的 `verb` 是原字符串中一段字符的视图，而不是单独的字符串。

**通俗理解：** 两个 span 使用 `==` 时，检查的是“是否指向相同的起始内存位置，并且长度相同”，不是逐个比较字符；`verb is "POST"` 则检查完整字符内容是否等于 `POST`，且区分大小写。本例中两段数据内容相同，但位置不同，所以 `==` 为 `False`，常量字符串模式为 `True`。

### 11.5 接口支持静态抽象和静态虚拟成员：通过泛型调用实现类型的静态成员

接口可以要求实现类型提供静态成员，并让泛型代码通过受约束的类型参数 `T` 调用这些成员。普通接口的实例成员通过对象调用，例如 `student.Work()`；这里的静态成员则通过类型参数调用，例如 `T.CreateDefault()`。

**static abstract：要求实现类型提供实现。**

```csharp
using System;

var student = Creator.Build<Student>();
Console.WriteLine(student.Name); // 访客

// 1. 接口要求实现类型提供静态创建方法
public interface IHasDefault<TSelf> where TSelf : IHasDefault<TSelf>
{
    static abstract TSelf CreateDefault();
}

// 2. 实现类型提供静态方法，不写 override
public class Student : IHasDefault<Student>
{
    public string Name { get; set; } = "";

    public static Student CreateDefault()
        => new Student { Name = "访客" };
}

// 3. 泛型代码根据接口约束调用实现类型的静态方法
public static class Creator
{
    public static T Build<T>() where T : IHasDefault<T>
        => T.CreateDefault();
}
```

`Build<Student>()` 中的 `T` 是 `Student`，因此调用 `Student.CreateDefault()`。如果另一个类型也实现了 `IHasDefault<自身类型>`，同一份 `Build<T>()` 就能调用那个类型提供的创建方法。若具体类型没有提供要求的静态方法，会编译报错。

### 11.6 泛型数学：同一份算法用于多种数字类型

```csharp
using System.Numerics;

static T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    T total = T.Zero;
    foreach (T value in values)
        total += value;
    return total;
}

Console.WriteLine(Sum(new[] { 2, 4, 6 }));     // int
Console.WriteLine(Sum(new[] { 1.5m, 2.5m })); // decimal
```

过去普通泛型约束无法表达“T 支持加法”，往往要写多个重载。现在语言与 .NET 7 数学接口配合，可以在泛型中使用数字运算。`INumber<T>` 是类库接口，不是 C# 关键字；结果仍遵循具体数字类型的精度和溢出规则。

**原理：11.5 是语言机制，泛型数学是它的重要应用。** 数学接口通过静态抽象成员约定“零”“加法”等能力；泛型代码有了接口约束，就可以通过 `T` 使用这些能力。下面是用于解释原理的简化接口，并不是 `INumber<T>` 的实际定义：

```csharp
public interface IAddable<TSelf> where TSelf : IAddable<TSelf>
{
    static abstract TSelf Zero { get; }
    static abstract TSelf operator +(TSelf left, TSelf right);
}
```

`Zero` 要求实现类型提供表示零的静态属性；`operator +` 要求实现类型提供加法运算符。实际的 `INumber<T>` 通过一组数学接口提供相应约定，`int`、`double`、`decimal` 等类型已经实现了它们。因此，`where T : INumber<T>` 让编译器知道上面的 `T.Zero` 和 `total + value` 是合法的，而不是仅凭成员名字猜测 `T` 是否支持这些操作。

### 11.7 泛型特性：特性可以直接接收类型参数

```csharp
public class ConverterAttribute<T> : Attribute { }

[ConverterAttribute<int>]
public class NumberField { }
```

以前常把类型以 `typeof(int)` 传给特性构造函数；现在也可以用泛型类型参数表达，并利用泛型约束。应用特性时必须使用符合特性规则的封闭类型，不能把未确定的类型参数随意传进去。见[泛型特性规范](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-11.0/generic-attributes.md)。

### 11.8 required：创建对象时必须提供成员

```csharp
public class Account
{
    public required string Name { get; init; }
    public required int Level { get; set; }
}

// 调用
var account = new Account { Name = "小陈", Level = 1 };
// var incomplete = new Account(); // 编译错误
```

`required` 可用于字段和属性，要求调用方在初始化时设置，或调用标记为 `[SetsRequiredMembers]` 的构造函数。该特性是构造函数的承诺，编译器不会逐一证明它已赋好所有必需成员。

### 11.9 自动默认结构体：未显式赋值的字段自动补默认值

```csharp
public struct Measurement
{
    public int Value;
    public bool IsValid;

    public Measurement(int value)
    {
        Value = value;
        // C# 11 自动使 IsValid 为 false
    }
}
```

以前结构体构造函数往往必须手动给所有字段赋值；现在编译器能为缺少赋值的字段补上默认初始化。

### 11.10 nameof 的作用域扩大到参数相关特性

C# 11 之前，这个位置需要手写参数名字符串：

```csharp
using System.Diagnostics.CodeAnalysis;

[return: NotNullIfNotNull("text")]
static string? Normalize(string? text) => text?.Trim();
```

C# 11 起，可以直接用 `nameof(text)` 引用参数名：

```csharp
using System.Diagnostics.CodeAnalysis;

[return: NotNullIfNotNull(nameof(text))]
static string? Normalize(string? text) => text?.Trim();
```

方法、局部函数或 Lambda 的相关特性中，可以在 `nameof` 内引用相应参数等名称。这样参数重命名时，特性表达的关联更容易保持一致。

以上是同一方法的两种写法，应分别使用。`nameof(text)` 生成的是字符串 `"text"`，不是参数的运行时值；区别在于它是编译器识别的名称引用，IDE 重命名参数时可以同步更新，避免手写字符串被漏改。

### 11.11 ref 字段、scoped 和 UnscopedRef：更明确地控制引用寿命

这组功能解决的是：引用直接指向一块存储位置，怎样保证使用它时，那块存储仍然有效。这里主要讨论 `ref`、`Span<T>` 这类引用，而不是普通的类变量。

**1. ref 字段：保存变量的位置，而不是复制它的值。** C# 11 允许在 `ref struct` 中声明 `ref` 字段：

```csharp
public ref struct IntSlot
{
    private ref int _value;

    public IntSlot(ref int value) => _value = ref value;
    public void Set(int value) => _value = value;
}
```

使用时，修改字段引用的位置，就是修改原变量：

```csharp
// 方法内调用
int number = 10;
var slot = new IntSlot(ref number);
slot.Set(20);
Console.WriteLine(number); // 20
```

注意，`_value = ref value;` 是让字段引用另一个存储位置；`_value = value;` 则是向字段当前引用的位置写入数值。前者设置引用，后者修改值。

**2. scoped：限制引用向外传递。** 方法可以使用参数，但不能把这个参数携带的引用返回出去：

```csharp
static void Inspect(scoped ReadOnlySpan<int> data)
{
    Console.WriteLine(data.Length); // 可以读取和使用
}

// 编译错误的示例：不能让这个引用逃出方法
// static ReadOnlySpan<int> Escape(scoped ReadOnlySpan<int> data)
// {
//     return data;
// }
```

`scoped` 不禁止读取或计算，而是限制引用流向更长寿命的位置。它不是运行时销毁变量的指令，而是编译器检查引用安全范围时使用的约束。

**3. UnscopedRef：调整某些默认的引用限制。** 普通结构体的实例方法默认不能返回自身普通字段的引用，下面的写法会编译失败：

```csharp
public struct Counter
{
    private int _value;

    // public ref int GetValue() => ref _value; // 编译错误
}
```

在这个方法上添加 `[UnscopedRef]` 后，可以声明这种引用返回。下面是一个独立的完整示例，可在 C# 11 + .NET 7 或更高版本中运行：

```csharp
using System;
using System.Diagnostics.CodeAnalysis;

var counter = new Counter();
ref int value = ref counter.GetValue();

value = 10; // 修改 counter 的 _value
Console.WriteLine(counter.GetValueCopy()); // 10

public struct Counter
{
    private int _value;

    [UnscopedRef]
    public ref int GetValue() => ref _value;

    // 只返回值的副本，不需要 UnscopedRef
    public int GetValueCopy() => _value;
}
```

这里的特性放宽了结构体实例方法隐式 `this` 的引用范围限制，但不会延长变量的实际寿命，也不会关闭引用安全检查。返回的引用仍不能超出 `counter` 存储位置的安全寿命。它只适用于语言允许的声明位置，不能把它当成任意返回引用的通行证。

**通俗理解：** `ref` 字段保存引用；`scoped` 限制引用外传；`UnscopedRef` 调整特定声明的默认限制。见[引用安全说明](https://learn.microsoft.com/en-us/dotnet/csharp/write-safe-efficient-code)和[引用范围规则](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/variables)。

<a id="csharp12"></a>
## C# 12：构造对象和集合更简洁

### 12.1 主构造函数扩展到普通 class 和 struct

```csharp
public class GreetingService(string prefix)
{
    public string Greet(string name) => $"{prefix}，{name}";
}

// 调用
var service = new GreetingService("你好");
Console.WriteLine(service.Greet("小王"));
```

构造参数写在类型名后面，可在类型内部使用，适合依赖注入和简单对象。**普通 class/struct 的主构造参数不会自动成为公开属性**，也不是天然只读字段；record 的位置参数规则不同。

### 12.2 集合表达式：统一用方括号构造集合

```csharp
int[] first = [2, 4];
List<int> values = [0, .. first, 6];
ReadOnlySpan<int> lookup = [10, 20, 30];
int[] empty = [];
```

`[ ... ]` 表示构造集合，`.. first` 表示把已有序列的元素逐个加入新集合。支持数组、Span、特定集合接口，以及满足构造规则或指定集合构建器的自定义类型。

### 12.3 Lambda 参数支持默认值和 params

```csharp
var greet = (string name = "访客") => $"你好，{name}";
Console.WriteLine(greet());
Console.WriteLine(greet("小张"));

var count = (params int[] items) => items.Length;
Console.WriteLine(count(2, 4, 6));
```

编译器可以为这些签名推断或合成合适的委托；方法组的自然类型推断也会考虑可选参数和参数数组。可选值取决于调用处看到的委托签名，不能假定转换成任意 `Func<...>` 后仍可省略参数。

### 12.4 ref readonly 参数：明确接受只读引用

```csharp
static int Read(ref readonly int value) => value;

int number = 8;
Console.WriteLine(Read(in number));
Console.WriteLine(Read(ref number));
```

被调用方法不能通过这个参数修改原值。相比 `in`，这种声明更强调调用方提供已有存储位置；省略调用修饰符或传入某些临时值可能产生警告，而非所有情况都直接报错。

### 12.5 类型别名放宽：元组、数组、指针等也可以命名

```csharp
using Position = (double Latitude, double Longitude);
using Scores = int[];

Position office = (30.2, 120.1);
Scores scores = [80, 90];
```

别名只是方便书写，不会创建一个与原类型不同的新类型。虽然常称“为任意类型创建别名”，仍有规则限制：例如不能直接给可空引用类型 `string?` 建别名；可空值类型 `int?` 则可以。指针别名涉及 `unsafe`。

### 12.6 内联数组：结构体内嵌固定数量的元素

```csharp
using System.Runtime.CompilerServices;

[InlineArray(4)]
public struct SmallBuffer
{
    private int _element0;
}

// 调用
var buffer = new SmallBuffer();
buffer[0] = 100;
buffer[3] = 400;
```

声明中只写一个实例字段，运行时按特性指定数量提供连续存储，语言提供索引、切片等支持。它不是“结构体里放了一个普通数组对象”，适合低分配缓冲区；需要相应运行时支持。

### 12.7 ExperimentalAttribute：标明 API 尚在试验

```csharp
using System.Diagnostics.CodeAnalysis;

[Experimental("DEMO001")]
public class PreviewParser { }
```

类型、成员或程序集可标记为试验性，使用时编译器报告带相应标识的诊断。它告诉调用者 API 可能变化；这是正式支持的“标记试验 API 的机制”，不表示 `ExperimentalAttribute` 本身只是语言预览功能。

### 12.8 nameof 可以在更多位置引用实例成员

```csharp
public class Label
{
    public string Text { get; set; } = "";
    public const string SizeMember = nameof(Text.Length); // "Length"
}
```

即使当前位置不能实际访问实例，`nameof` 也可以在新增支持的成员表达式中只取名字。它不读取 `Text` 的值，不会调用 `Length`。

<a id="csharp13"></a>
## C# 13：集合参数与高性能类型更实用

### 13.1 params 集合：可变参数不再只支持数组

```csharp
static int Total(params ReadOnlySpan<int> values)
{
    int total = 0;
    foreach (int value in values)
        total += value;
    return total;
}

Console.WriteLine(Total(3, 6, 9));
```

`params` 可用于 Span、ReadOnlySpan、受支持的集合接口，以及满足集合构造规则的类型。调用方仍可像传多个参数一样使用；以 Span 为目标时，编译器有机会避免传统参数数组分配。

### 13.2 新的 lock 类型和编译规则

```csharp
public class Counter
{
    private readonly System.Threading.Lock _gate = new();
    private int _count;

    public void Increment()
    {
        lock (_gate)
        {
            _count++;
        }
    }
}
```

当锁表达式的静态类型是 .NET 9 的 `System.Threading.Lock` 时，编译器采用它的 `EnterScope()`/释放机制。普通对象的 `lock` 仍走既有机制。

不要先把 `Lock` 转成 `object` 再锁：静态类型改变会影响编译结果。`lock` 语句体仍不能包含 `await`。

### 13.3 对象初始化器中支持隐式倒数索引

```csharp
public class RecentValues
{
    public int[] Items { get; } = new int[4];
}

// 调用：修改已创建数组的最后两个位置
var recent = new RecentValues
{
    Items = { [^1] = 9, [^2] = 8 }
};
```

`^1` 表示最后一项，`^2` 表示倒数第二项。`^` 本身早已存在，这次新增的是在这种对象初始化器位置使用隐式索引的能力，不会自动扩容集合。

### 13.4 async / 迭代器中放宽 ref、ref struct 和 unsafe 限制

```csharp
static async Task<int> GetLengthAsync()
{
    int length;
    {
        ReadOnlySpan<char> text = "example";
        length = text.Length;
    }

    await Task.Delay(1);
    return length;
}
```

异步方法和迭代器可以在符合安全规则的位置使用 `ref` 局部变量、Span 等 `ref struct` 局部值。**这些引用类值不能跨越 `await` 或 `yield return` 暂停点继续使用。**

迭代器对 `unsafe` 的限制也放宽，但 `yield return/yield break` 必须处于安全上下文，`await` 也不能因此随意放进不安全上下文。async 方法的 `ref/out` 参数限制并未一并取消。

### 13.5 ref struct 可以实现接口

```csharp
public interface ICounted
{
    int Count { get; }
}

public ref struct CountView : ICounted
{
    public int Count => 3;
}
```

这让栈受限类型能表达共同能力。但仍不能把实例转换为接口变量，因为那会涉及不允许的装箱；接口中适用的实例成员也必须显式实现，不能依赖默认接口实现绕过限制。

### 13.6 allows ref struct：泛型可以接受栈受限类型

```csharp
static int ReadCount<T>(T value)
    where T : ICounted, allows ref struct
    => value.Count;
```

`allows ref struct` 表示“T 可以是 ref struct”，而不是“T 必须是 ref struct”。方法内部因此必须遵守相关引用安全规则，不能把 T 随意装箱、保存到普通对象字段或跨暂停点使用。

### 13.7 分部属性和分部索引器

```csharp
public partial class Profile
{
    public partial string Nickname { get; set; }
}

public partial class Profile
{
    private string _nickname = "";

    public partial string Nickname
    {
        get => _nickname;
        set => _nickname = value.Trim();
    }
}
```

属性、索引器也可以拆成声明部分和实现部分，方便源生成器补充校验或通知逻辑。必须配对，且实现部分不能只写成普通自动属性后就当作完整实现。

### 13.8 重载解析优先级

库作者可用 `System.Runtime.CompilerServices.OverloadResolutionPriorityAttribute` 为适用重载指定相对优先级。比如同时保留旧数组 API 和新 Span API，在满足规则的候选中引导编译器选择更合适的版本。

它不是让不匹配的重载强行可用，也不是跨所有类型、所有候选的无条件排名；只作用于规范允许的比较范围。通常是库作者的迁移工具。

<a id="csharp14"></a>
## C# 14：扩展成员与日常语法完善

### 14.1 扩展成员：增加扩展属性和静态扩展成员

```csharp
public static class TextExtensions
{
    extension(string text)
    {
        public bool IsBlank => string.IsNullOrWhiteSpace(text);
        public string AddExclamation() => text + "!";
    }

    extension(string)
    {
        public static string DefaultGreeting => "你好";
    }
}

// 调用
Console.WriteLine("   ".IsBlank);          // True
Console.WriteLine("欢迎".AddExclamation());
Console.WriteLine(string.DefaultGreeting);
```

新的 `extension(...)` 块把同一类型的扩展成员放在一起，既支持面向实例的方法、属性，也支持面向类型的静态成员。旧式 `this` 扩展方法继续可用。

### 14.2 扩展运算符：可为已有类型补充运算

在扩展块中，还可以按规范声明静态扩展运算符，例如为某种序列类型定义 `+`，表达把两个序列连接起来。它仍是编译期根据类型、作用域和候选规则解析，不会改写运行时原类型，也不会覆盖所有既有运算。

这适合设计库 API，但最好让符号含义符合使用者直觉。

### 14.3 field：属性里直接访问编译器生成的后备字段

```csharp
public class Person
{
    public string Name
    {
        get;
        set => field = string.IsNullOrWhiteSpace(value)
            ? throw new ArgumentException("姓名不能为空")
            : value.Trim();
    } = "未命名";
}
```

以前加校验通常要手写 `_name` 字段；现在可以保留自动 getter，只给 setter 添加逻辑，通过 `field` 操作隐藏字段。一个或两个访问器都可以有方法体。

`field` 是上下文关键字。如果旧代码正好有同名成员，可能需要改名或用 `this.field` / `@field` 消歧。

### 14.4 nameof 支持未绑定泛型类型

```csharp
Console.WriteLine(nameof(List<>));          // List
Console.WriteLine(nameof(Dictionary<,>));    // Dictionary
```

只想取泛型类型名称时，不再必须填写无关紧要的具体类型参数。结果仍是简单名称，不包含命名空间或实际泛型参数。见 [C# 14 nameof 说明](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-14#unbound-generic-types-and-nameof)。

### 14.5 Lambda 参数带修饰符时也可以省略类型

```csharp
public delegate bool IntParser(string text, out int value);

// 调用
IntParser parser = (text, out value) => int.TryParse(text, out value);
```

目标委托已给出参数类型时，可以省略重复类型，同时保留 `ref`、`out`、`in`、`ref readonly`、`scoped` 等适用修饰符。`params` 仍要求显式类型，不能套用这一省略规则。

### 14.6 分部实例构造函数和分部事件

```csharp
public partial class Session
{
    public partial Session(string name);
}

public partial class Session
{
    public string Name { get; }

    public partial Session(string name)
    {
        Name = name;
    }
}
```

实例构造函数和事件也可拆分声明与实现，继续完善源生成器协作方式。两者都必须恰好有一份声明和一份实现。

构造函数的 `this(...)` / `base(...)` 初始化器只能放在实现部分；分部事件的声明部分像普通字段式事件，实现部分必须提供 `add/remove` 访问器。

### 14.7 自定义复合赋值，以及实例自增/自减运算符

```csharp
public class RunningTotal
{
    public int Value { get; private set; }

    public void operator +=(int amount) => Value += amount;
    public void operator ++() => Value++;
}

// 调用
var total = new RunningTotal();
total += 10;
total++;
Console.WriteLine(total.Value); // 11
```

以前通常通过重载 `+` 间接支持 `+=`；现在可直接定义实例复合赋值运算符，就地更新对象。相应实例 `++/--` 也可定义。复合赋值实例运算符返回 `void`、接收一个右操作数；实例 `++/--` 返回 `void`、没有显式参数。

### 14.8 空条件赋值：对象存在时才写入

```csharp
// 假设 customer 为可能为 null 的引用，包含 Name 和 Points 属性
customer?.Name = "小赵";
customer?.Points += 10;

int[]? numbers = null;
numbers?[0] = 99;
```

以前 `?.` / `?[]` 主要用于读取或调用，现在也能出现在赋值、复合赋值左侧。接收者为 `null` 时跳过赋值，右侧表达式也不求值；存在时仍会进行正常属性或索引检查。它不会自动创建对象，也不会忽略数组越界。
