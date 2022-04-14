---
title: PHP的编码规范
date: 2018-07-18 10:36:48
tags: 技术
---

# PHP编码规范
版本：v1.0

日期：2018-10-10

## 前言
### 目的
为了更好的提高技术部的工作效率，保证开发的有效性和合理性，并可最大程度的提高程序代码的可读性和可重复利用性，指定此规范。

### 整体要求
本规范以 [PSR2](https://www.php-fig.org/psr/psr-2/) 规范为基础制定,是 [PSR2](https://www.php-fig.org/psr/psr-2/) 规范的继承与扩展。

新项目需完全遵守本规范，之前旧系统代码可以可以继续遵守原有的规范。

### 「能愿动词」的使用

为了避免歧义，文档大量使用了「能愿动词」([RFC 2119](http://www.ietf.org/rfc/rfc2119.txt))，对应的解释如下：

* 必须 (MUST)：绝对，严格遵循，请照做，无条件遵守；
* 一定不可 (MUST NOT)：禁令，严令禁止；
* 应该 (SHOULD) ：强烈建议这样做，但是不强求；
* 不该 (SHOULD NOT)：强烈不建议这样做，但是不强求；

## 规范

### 1. 关于 PHP 版本选择

* **必须** 使用 `PHP 7.0` 以上的版本。
* **应该** 使用最新的稳定版本。

### 文件

* 源文件 **必须** 只使用 `<?php`。
* 源文件中 PHP 代码的编码格式 **必须** 只使用 `不带 BOM 的 UTF-8`。
* 源文件在文件结尾处 **必须** 忽略掉 `?>` 且 **必须** 以一个空行结尾。
* 类定义文件 **必须** 使用类名作为文件名, 用首字母大写。
* 类文件 **必须** 使用命名空间为路径存储。
* 除类文件外，其他文件全部用小写字母加下划线命名。

### 严格模式

* 源文件中 **必须** 声明为严格模式 `declare(strict_types=1);`。
* **必须** 为类方法和函数声明参数类类型、返回值类型。
* 一个类方法和函数 **必须** 只有一种类型的返回参数。

### 基础规范

* 一个源文件 **应该** 只用来做声明（类，函数，常量等）或者只用来做一些辅助作用的操作（例如：输出信息，修改 .ini 配置等），但不应当同时做这两件事。
* 类名 **必须** 使用 `StudlyCaps` 写法,比如 `SampleController`。
* 类中的常量 **必须** 只由大写字母和下划线(_)组成。
* 方法名 **必须** 使用 `camelCase（驼峰式)` 写法，比如 `getTotalById`。
* 变量名 **必须** 使用 `camelCase（驼峰式)` 写法。
* 代码 **必须** 使用 4 个空格符进行缩进。
    备注: 使用空格而不是 `tab` 键 缩进的好处在于，避免在比较代码差异、打补丁、重阅代码以及注释时产生混淆。并且，使用空格缩进，让对齐变得更方便。
* 关键字 以及 `true` / `false` / `null`
    PHP 所有关键字 **必须** 全部小写。
    常量 `true` 、`false` 和 `null` 也必须全部小写。
* 运算符(`=, +, -, *, /, %, +=`等等)左右两边 **必须要有** 一个空格。
* **应该** 使用 `===` 运算符代替 `==` 运算符，应该使用 `true` / `false` 常来代替 `1` / `0`。
* 每行 **一定不可** 有多条语句。
* 非空行后 **一定不可** 有多余的空格。
* 运算表达式可以在运算符处换行，且运算符应 **该放** 在下一行的开始。
* 缩写词（含专有名词）与其他词组合命名， **必须** 使用 `camelCase（驼峰式)` 写法，如： `ApiUrl`。

### 行 与 空行

* 每行的字符数 **应该** 保持在 80 个以内， 通常情况 **不该** 超过 120 个。如果换行可能破坏语意或降低可阅读性，可作为例外超出限制。
* 代码块开始之前与结束符 `}` 之后，**应该** 有一个空行。
* 方法和函数的 `return` 语句之前，**应该** 有一个空行。
* 完成一定功能的代码块前后，**应该** 各有一个空行。
* **一定不可** 存在连续的 2 个以上的空行。

### 类、Interface、Trait

* 每个 `namespace` 命名空间声明语句和 `use` 声明语句块后面，必须插入一个空白行。所有 `use` **必须** 在 `namespace` 后声明。每条 `use` 声明语句 **必须** 只有一个 `use` 关键词。`use` 声明语句块后 **必须** 要有一个空白行。例如：

    ```php
    <?php

    declare(strict_types=1);

    namespace Vendor\Package;

    use FooClass;
    use BarClass as Bar;
    use OtherVendor\OtherPackage\BazClass;

    // ... additional PHP code ...
    ```

* 扩展与继承，关键词 `extends` 和 `implements`  **必须** 写在类名称的同一行，类的开始花括号 **必须** 独占一行，结束花括号也 **必须** 在类主体后独占一行。例如:

    ```php
    <?php

    declare(strict_types=1);

    namespace Vendor\Package;

    use FooClass;
    use BarClass as Bar;
    use OtherVendor\OtherPackage\BazClass;

    class ClassName extends ParentClass implements ArrayAccess, Countable
    {
        // constants, properties, methods
    }
    ```

	`implements` 的继承列表也可以分成多行，这样的话，每个继承接口名称都 **必须** 分开独立成行，包括第一个。

    ```php
    <?php

    declare(strict_types=1);

    namespace Vendor\Package;

    use FooClass;
    use BarClass as Bar;
    use OtherVendor\OtherPackage\BazClass;

    class ClassName extends ParentClass implements
        ArrayAccess,
        Countable,
        Serializable
    {
        // constants, properties, methods
    }
    ```

* 类的开始花括号(`{`) **必须** 写在函数声明后自成一行，结束花括号(`}`)也 **必须** 写在函数主体后自成一行。
* 方法的开始花括号(`{`) **必须** 写在函数声明后自成一行，结束花括号(`}`) **也必须** 写在函数主体后自成一行。
    一个标准的方法声明可参照以下范例，留意其括号、逗号、空格以及花括号的位置。

    ```php
    <?php

    declare(strict_types=1);

    namespace Vendor\Package;

    class ClassName
    {
        public function fooBarBaz($arg1, &$arg2, $arg3 = [])
        {
            // method body
        }
    }
    ```

* 类的属性和方法必须添加访问修饰符（`private`、`protected` 以及 `public`）， `abstract` 以及 `final` 必须声明在访问修饰符之前，而 `static` **必须** 声明在访问修饰符之后。
    以下是属性声明的一个范例：

    ```php
    <?php

    declare(strict_types=1);

    namespace Vendor\Package;

    class ClassName
    {
        public $foo = null;
    }
    ```

    需要添加 `abstract` 或 `final` 声明时， **必须** 写在访问修饰符前，而 `static` 则 **必须** 写在其后。

    ```php
    <?php

    declare(strict_types=1);

    namespace Vendor\Package;

    abstract class ClassName
    {
        protected static $foo;

        abstract protected function zim();

        final public static function bar()
        {
            // method body
        }
    }
    ```

* 类的属性和方法 **一定不可** 使用 `_` 开头命名来区分私有方法。
* 方法的参数声明过多时，每个参数可以独占一行
    ```php
    <?php

    declare(strict_types=1);

    namespace Vendor\Package;

    class ReturnTypeVariations
    {
        public function anotherFunction(
            string $foo,
            string $bar,
            int $baz
        ): string {
            return 'foo';
        }
    }
    ```

### 类方法与函数

* 参数和变量列表中逗号(，)前 **必须不能有** 空格，而逗号后 **必须要有** 空格。

    ```php
    <?php
    
    declare(strict_types=1);

    namespace Vendor\Package;

    class ClassName
    {
        public function foo($arg1, &$arg2, $arg3 = [])
        {
            // method body
        }
    }
    ```

* 参数列表可以分列成多行，这样，包括第一个参数在内的每个参数都 **必须** 单独成行。
    拆分成多行的参数列表后，结束括号以及方法开始花括号 **必须** 写在同一行，中间用一个空格分隔。

    ```php
    <?php

    declare(strict_types=1);

    namespace Vendor\Package;

    class ClassName
    {
        public function aVeryLongMethodName(
            ClassTypeHint $arg1,
            &$arg2,
            array $arg3 = []
        ) {
            // method body
        }
    }
    ```

* 方法及函数调用时，方法名或函数名与参数左括号之间 **一定不可** 有空格，参数右括号前也 **一定不可** 有空格。每个参数前 **一定不可** 有空格，但参数 `，` 之后 **必须** 有一个空格。

### 控制结构

* 控制结构的基本规范如下
    * 控制结构的关键字后 **必须** 要有一个空格符，而调用方法或函数时则一定不能有。
    * 控制结构的开始花括号(`{`) **必须** 写在声明的同一行，而结束花括号(`}`)必须写在主体后自成一行。
    * 控制结构的开始左括号后和结束右括号前，都 **一定不可** 有空格符。

* `case` 语句 **必须** 相对 `switch` 进行一次缩进，而 `break` 语句以及 `case` 内的其它语句都 **必须** 相对 `case` 进行一次缩进。
如果存在非空的 `case` 直穿语句，主体里 **必须** 有类似 `// no break` 的注释。例如:

    ```php
    <?php

    declare(strict_types=1);

    switch ($expr) {
        case 0:
            echo 'First case, with a break';
            break;
        case 1:
            echo 'Second case, which falls through';
            // no break
        case 2:
        case 3:
        case 4:
            echo 'Third case, return instead of break';
            return;
        default:
            echo 'Default case';
            break;
    }
    ```

* **应该** 使用关键词 `elseif` 代替所有 `else if`，以使得所有的控制关键字都像是单独的一个词。例如:

    ```php
    <?php

    declare(strict_types=1);

    if ($expr1) {
        // if body
    } elseif ($expr2) {
        // elseif body
    } else {
        // else body;
    }
    ```

* while 和 do while。一个规范的 while 语句 **应该** 如下所示，注意其 括号、空格以及花括号的位置。

    ```php
    <?php

    declare(strict_types=1);

    while ($expr) {
        // structure body
    }
    ```

* 标准的 `do while` 语句如下所示，同样的，注意其 括号、空格以及花括号的位置。

    ```php
    <?php

    declare(strict_types=1);

    do {
        // structure body;
    } while ($expr);
    ```

* 标准的 `for` 语句如下所示，注意其括号、空格以及花括号的位置。

    ```php
    <?php

    for ($i = 0; $i < 10; $i++) {
        // for body
    }
    ```

* 标准的 `foreach` 语句如下所示，注意其括号、空格以及花括号的位置。例如:

    ```php
    <?php

    foreach ($iterable as $key => $value) {
        // foreach body
    }
    ```

* 标准的 `try catch` 语句如下所示，注意其括号、空格以及花括号的位置。例如:
    ```php
    <?php

    declare(strict_types=1);

    try {
        // try body
    } catch (FirstExceptionType $e) {
        // catch body
    } catch (OtherExceptionType $e) {
        // catch body
    }
    ```

### 闭包

* 闭包声明时，关键词 `function` 后以及关键词 `use` 的前后都必须要有一个空格。

    ```php
    <?php

    $closureWithArgs = function ($arg1, $arg2) {
        // body
    };

    $closureWithArgsAndVars = function ($arg1, $arg2) use ($var1, $var2) {
        // body
    };

    $closureWithArgsVarsAndReturn = function ($arg1, $arg2) use ($var1, $var2): bool {
        // body
    };

    $longArgs_longVars = function (
        $longArgument,
        $longerArgument,
        $muchLongerArgument
    ) use (
        $longVar1,
        $longerVar2,
        $muchLongerVar3
    ) {
        // body
    };

    $foo->bar(
        $arg1,
        function ($arg2) use ($var1) {
            // body
        },
        $arg3
    );
    ```

* 开始花括号 **必须** 写在声明的同一行，结束花括号 **必须** 紧跟主体结束的下一行。
* 同事遵守 `函数` 和 `类方法` 的相关原则。

### 匿名类

* 匿名类 **必须** 遵守 `闭包` 和 `类` 的相关原则。

    ```PHP
    <?php

    declare(strict_types=1);

    $instance = new class extends \Foo implements \HandleableInterface {
        // Class content
    };

    $instance = new class extends \Foo implements
        \ArrayAccess,
        \Countable,
        \Serializable
    {
        // Class content
    };
    ```

### 注释

* 文件头部注释 (可选)
    * `author`： 作者以及联系邮箱。
    * `createTime`：文件创建时间 - 可以通过git来追踪。
    * `description`：当前文件的详细介绍。

    ```PHP
    /**
    * createTime : 18-10-8 12:23
    * description: 测试通知服务
    */
    ```

* 方法注释 (可选)
    使用 `PHPdoc` 方式，描述方法功能、参数等。

    * `@param` 参数，语法：`@param ["Type"] [name] [<description>]`，标签用于记录函数或方法的单个参数，且可以有多行描述，不需要明确的分隔。
    * `@return` 返回，语法：`@return <"Type"> [description]`，标签用于记录函数或方法的返回类型，同样支持多行描述。
    * `@throws` 参数，语法：`@throws [Type] [<description>]`，标签用于记录函数或方法的抛出的错误异常，注明发生情况。

    ```PHP
    /**
     * 创建商户 union_id
     * - 向 points_account 插入一条数据，获取 id
     * - 使用该 id 向 account base service 申请 union_id
     * - 更新记录
     *
     * @param int $roleType 角色类型
     * @return PointsAccount 账号 
     * @throws \Throwable 生成 union_id 或者 创建账号失败抛出异常
     */
    public static function generateUnionIdAndCreateAccount(int $roleType): PointsAccount
    {
        // code
    }
    ```

* 变量注释 (可选)
    * `@var` 变量，语法：`@var ["Type"] [element_name] [<description>]`，标签用于记录变量类型，也用于记录常量等

    ```PHP
    /**
     * @var integer 待审核
     */
    const STATUS_PENDING = 0;
    ```


## Example 

```PHP
<?php

declare(strict_types=1);

namespace Vendor\Package;

use Vendor\Package\{ClassA as A, ClassB, ClassC as C};
use Vendor\Package\SomeNamespace\ClassD as D;

use function Vendor\Package\{functionA, functionB, functionC};

use const Vendor\Package\{ConstantA, ConstantB, ConstantC};

class Foo extends Bar implements FooInterface
{
    public function sampleFunction(int $a, int $b = null): array
    {
        if ($a === $b) {
            bar();
        } elseif ($a > $b) {
            $foo->bar($arg1);
        } else {
            BazClass::bar($arg2, $arg3);
        }
    }

    final public static function bar()
    {
        // method body
    }
}
```