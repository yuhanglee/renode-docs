# 平台描述格式

为了满足将外围模型轻松组装成完整平台定义的需求，根据框架日常工作中的常见用例，为 Renode 创建了一个类似 YAML 的平台描述格式。

通常，这种格式的文件具有 `.repl` （REnode PLatform） 扩展名。

该格式旨在人类可读、简洁、易于解析、基于、扩展和修改。

## 缩进

在 Renode 的平台描述格式中，有意义的缩进（类似于例如 Python）与大括号 （`{`， `}`） *一起使用* 。规则如下：

1. 缩进仅使用空格，并且 indent 必须是四个空格的倍数。
2. 从语法上讲，一个缩进级别对应于一个大括号（如果我们缩进，则开始一个，如果缩进，则关闭）。
3. 大括号内的缩进没有意义，这也适用于换行符。他们都被视为白色字符。当使用有意义的缩进时，我们将其称为缩进模式（而不是非缩进模式）。要在非缩进模式下分隔元素（对应于缩进模式下的行），必须使用分号。

例如，这些文件是等效的：

``` none
line1
line2
    line3
    line4
        line5
    line6
```

``` none
line1
line2 { line3; line4 { line5 }; line 6 }
```

## 评论

有两种类型的评论：

- 行注释以行尾开始并继续;
- 多行注释由 `/*` 和 `*/` 分隔，并且可以跨越多行。

这两种注释都可以在缩进和非缩进模式下使用，但有一个特殊规则。当多行注释跨越多行时，它必须在行尾结束。否则，将很难确定应该为该行的其余部分使用什么缩进。

换句话说，这个来源是合法的：

``` none
line1 /* here a comment starts
 here it continues
and here ends*/
line2
```

但这个不是：

``` none
line1 /* here a comment starts
 here it continues
and here ends*/ line2
```

## 基本结构

每个平台描述格式都由 *条目* 组成。条目是外围描述的基本单位。条目的基本格式如下：

``` none
variableName: TypeName registrationInfo
    attribute1
    attribute2
    ...
    attributeN
```

所有 `TypeName`、`registrationInfo` 和 `attributes` 都是可选的，但必须至少存在其中一个。如果条目包含 TypeName，则它是 *创建条目* （否则它是 *更新条目* ）。

每个创建条目都声明一个变量，给定变量只能有一个声明，并且它必须是解析文件时遇到的第一个条目，除非在解析之前声明了该变量。例如，在机器中注册的所有外围设备也作为变量导入，并且可以有其更新条目（但不能创建条目）。

换句话说，此代码是合法的：

``` none
variable1: SomeType
    property: value

variable1:
    property: otherValue
```

但以下情况会导致错误：

``` none
variable1:
    property: value

variable1: SomeType
    property: otherValue
```

连续的条目（对于给定的变量）称为 updating ，因为它们可以更新前一个条目提供的一些信息。最终，与给定变量对应的所有条目都会 *被合并* ，以便合并结果包含来自所有条目的属性，其中一些可能被其他一些条目失效。

TypeName 必须提供类型所在的完整命名空间。但是，如果命名空间以 `Antmicro.Renode.Peripheral` 开头，则可以省略这部分。

创建条目可以具有可选前缀 `local`，则在此条目中声明的变量称为_局部_变量。前缀仅用于 creating 条目，而不用于 updating 条目。

For example:

``` none
local cpu: SomeCPU
    StringProp: "a"

cpu:
    IntProp: 32
```

如果变量是 local，那么我们只能在该文件中引用它。阅读下一节后，这会更清楚，但通常如果一个文件依赖于另一个文件，两个文件都可以声明相同的命名局部变量，并且它们是完全独立的，特别是它们可以有不同的类型。

## 依赖其他文件

一个描述可以依赖于另一个描述，在这种情况下，它可以使用该文件中的所有（非局部）变量。请注意，我们所依赖的文件中的所有非局部变量都不能有 creating entries。换句话说，依赖于另一个文件就像将其粘贴到文件顶部，但局部变量除外。

`using` 关键字用于声明依赖项：

``` none
using "path"
```

上面的行称为 *using 条目* 。所有 using 条目都必须位于任何其他条目之前。还有一种语法允许用户依赖一个文件，但在该文件中的所有变量前面加上前缀：

``` none
using "path" prefixed "prefix"
```

然后 `prefix` 应用于 `path` 中的每个变量。

由于 `path` 中提到的文件可以进一步依赖于其他文件，因此这有时会导致一个循环。格式解释器会检测到此问题，并生成包含周期相关信息的错误。

## 数值

*值* 是平台描述格式中广泛使用的概念。有三种类型的值：

-  *简单值* ，可进一步分为：
  - 字符串（用双引号分隔，其中 ` \"` 用作转义的双引号）;
  - 多行字符串（用三引号 `'''` 分隔， `其中 \'''` 用作转义的三引号）（示例如下）;
  - 布尔值（`true` 或 `false`）;
  - 数字（十进制或十六进制，带 `0x` 前缀）;
  - 范围 （如下所述）
- 引用值，指向变量，仅作为变量的名称给出;
- 内联对象，表示值本身中描述的对象，并且不与任何变量绑定（稍后介绍）。

范围表示一个区间，可以以两种形式提供：

- `<begin, end>` or
- `<begin, +size>` where `begin`, `end` and `size` are decimal or hexadecimal numbers.

示例: `<0, 100>`, `<0x10000, +0x200>`.

带有转义分隔符的多行字符串示例：

``` none
name: '''this is \'''
some 
multiline
name'''
```

## 注册信息

Registration info 告诉给定外设应该在哪个 register 中注册以及如何注册。外设可以在一个或多个 registers 中注册。对于单个注册，注册信息的格式如下：

``` none
@ register registrationPoint as "alias"
```

其中 `registrationPoint` 是一个值，并且是可选的。`as “alias”` 部分称为_别名，也是可选的。使用 `registrationPoint` 时，将创建或直接使用注册点（如果指定的值是注册点）：如果未提供注册点，则使用 `NullRegistrationPoint` 或（如果不接受 `NullRegistrationPoint`）没有构造函数参数或所有参数都可选的注册点。

如果注册点是简单值，则注册点与构造函数一起使用，该构造函数采用一个参数，此简单值可以转换为该参数，并且可能还有其他可选参数。请注意，上述两种情况中的任何歧义都会导致错误。

如果注册点是参考值或内联对象，则它们将直接用作注册点。

在注册期间，注册的外围设备通常被赋予与变量名称相同的名称。但是，用户可以使用上述别名用其他名称覆盖此名称。

还支持多个注册;这具有以下形式：

``` none
@ {
    register1 registrationPoint1;
    register2 registrationPoint2;
    ...
    registerN registrationPointN
} as "alias"
```

元素的含义和可选性与前一种情况相同，唯一的区别是 peripheral 被多次注册，可能在不同的 registers 中。请注意，正如本文档开头所述 - 大括号内的缩进无关紧要。

注册信息可以在任何条目（创建或更新）中提供，也可以在多个条目中提供。在这种情况下，仅进行最新条目的注册。也可以取消注册，即在不提供新注册信息的情况下被覆盖。这是使用 `@ none` 表示法完成的，例如：

``` none
variable: @none
```

## 属性

有三种属性：

-   构造函数或属性属性;
-   中断属性;
-   init 属性。

### 构造函数或属性

构造函数或属性具有以下形式：

``` none
name: value
```

`name` 是属性（如果首字母为大写）或构造函数参数（否则）的名称，`value` 是一个值。当与属性一起使用时，如果属性的值可转换为此属性类型，则将设置此类转换的值（否则会产生错误）。

但是请注意，另一个条目可能会更新属性，以便只有最后一个（即最后一个包含设置此属性的属性）条目有效。

也可以使用 `none` 关键字代替值。拥有它意味着该属性不是使用任何值设置的，并且在_应用描述之前_保留其值。当某些条目设置了一些值，而我们想更新这个条目但不设置任何值时，它可能很有用。

`empty` 关键字可用于设置 property 或 constructor 参数的默认值：

- 数值设置为 `0`;
- 字符串值设置为 `null`;
- 枚举值设置为此枚举中 `index 0` 对应的值;
- 引用类型设置为 `null`;

构造函数属性以类似的方式合并，即分析属于给定变量的所有条目的属性，对于每个名称，我们采用具有该名称的最后一个值。peripheral 的构造函数是根据合并属性集选择的。对于创建条目中指定的类型的每个可能的构造函数，我们检查是否：

-   构造函数的每个参数都有一个默认值或相应的 attribute，即 attribute 与参数名称同名;
-   相应的属性具有值 convertible （对于简单类型） 或 assignable （否则） 到 parameter type;
-   所有属性都已使用。

如果满足所有条件，则分析的构造函数将标记为可用。如果只有一个构造函数可用，则使用此构造函数创建对象。如果没有这样的构造函数或有多个构造函数，则会产生错误。

因为如果所有数据都在一个地方（即类型和构造函数属性的名称），调试构造函数选择问题要容易得多，所以每当非创建条目包含构造函数参数时，都会发出警告（有效地更新创建条目）。

请注意，只能为要创建其变量的条目提供构造函数属性，因此在处理给定描述之前，无法提供任何 on variables reresets existing peripherals。

### 中断属性

顾名思义，中断属性用于指定定义该属性的变量的哪些中断连接以及连接位置。此类属性的最简单格式如下：

``` none
-> destination@number
```

其中 `destination` 是实现 `IGPIOReceiver` 接口的变量，`number` 是目标中断号。请注意，左侧没有指定任何内容 - 仅当存在 `GPIO` 类型的单个属性，或者有多个属性，但其中一个属性标有 `DefaultInterrupt` 属性时，才有可能这样做。这是连接的。

每当用户想要指定应该连接哪个属性时，可以使用更通用的形式：

``` none
propertyName -> destination@number
```

其中 `propertyName` 是应连接的属性（`GPIO` 类型）的名称。此外，如果类型实现 `INumberedGPIOOutput`，则可以使用数字代替属性名称。

如果要将多个中断连接到同一目标外设，则可以使用以下形式的属性：

``` none
[irq1, irq2, ..., irqN] -> destination@[irqDest1, irqDest2, ..., irqDestN]
```

其中 `irq1` 连接到 `irqDest1` 等。同样， `irq` s 可以是名称或数字 （如果实现了 `INumberedGPIOOutput`），而 `irqDest` s 必须是数字。当然，源和目标的 arity 必须匹配。

还可以使用 `|` 符号将单个源连接到多个目标，该符号可用于每个中断属性格式：

``` none
-> destination@number | another_destination@number
propertyName -> destination@number | another_destination@number
[irq1, irq2, ..., irqN] -> destination@[irqDest1, irqDest2, ..., irqDestN] | another_destination@[irqDest1, irqDest2, ..., irqDestN]
```

请注意，由 `|` 符号分隔的每个属性都必须与源具有相同的 arity。

在本地中断的情况下，还有一个表示法：

``` none
source -> destination#index@interrupt
```

`destination` 必须实现 `ILocalGPIOReceiver，index` 是本地 GPIO 接收器的索引。此表示法也可以与多个中断一起使用：

``` none
[irq1, irq2, ..., irqN] -> destination#index@[irqDest1, irqDest2, ..., irqDestN]
```

就像 properties 一样，interrupt attribute 可以更新旧的 attribute。这是根据源中断完成的，即如果来自不同条目的两个属性使用相同的源中断，则仅使用来自后者的属性。同样，就像在 properties 中一样，用户可能想要取消 irq 连接而不指定不同的连接。关键字 `none` 可用于此目的：

``` none
source -> none
```

### 初始化属性

Init 属性用于对变量执行 monitor 命令。它们具有以下形式之一：

``` none
init:
    monitorStatement1
    monitorStatement2
    ...
    monitorStatementN
```

``` none
init add:
    monitorStatement1
    monitorStatement2
    ...
    monitorStatementN
```

它们之间的区别在于，在 merge 阶段，第一个会覆盖给定变量的上一个 init 属性（如果有的话），而第二个会自行连接到前一个。最终执行最后一个条目：每个语句前面都加上变量绑定到的外围设备的名称，然后由 Monitor 直接解析。请注意，这意味着 init 部分仅对已注册的变量合法。

## 内联对象

内联对象是类似于引用值的值，但不是创建一个单独的变量然后引用它，而是直接在引用位置定义它。表格如下：

``` none
new Type
    attribute1
    attribute2
    ...
    attributeN
```

效果与创建此类型并使用这些属性的条目相同，但无法更新，并且仅在引用位置可用。因此，例如，这些代码会导致相同的效果：

``` none
variable: SomeType
    SomeProperty: point

point: Point
    x: 5
    y: 3
```

``` none
variable: SomeType
    SomeProperty: new Point {x: 5; y: 3}
```
