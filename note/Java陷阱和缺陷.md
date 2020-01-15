* [null访问static成员变量和方法](#null访问static成员变量和方法)
* [YYYY-MM-dd](#YYYY-MM-dd)

### null访问static成员变量和方法
```java
public class Test {
    static int value = 1;

    static void test() {
        System.out.println("value is " + value);
    }

    public static void main(String[] args) {
        Test test = null;
        System.out.println(test.value);
        test.test();
    }
}
```

运行`Test#main`未抛出异常，而是输出了如下结果：
```
1
value is 1
```

这是违反直觉的，所以不要使用对象调用类成员变量，另外上述代码在编译时会报警告“Access static member via instance reference”，所以在编码时尽量不要忽略警告。

C#规定只有类才能调用类成员变量，例如：

```C++
using System;

namespace CSharp
{
    class Program
    {
        static int value = 1;
        static void Main(string[] args)
        {
            Program p = new Program();
            p.test();
        }

        static void test() {
            Console.WriteLine("test");
        }
    }
}
```

编译时将报错“Member 'Program.test()' cannot be accessed with an instance reference; qualify it with a type name instead (CS0176) [CSharp]”

### YYYY-MM-dd

假设时间为 `2019年12月29日13时14分15秒`，此值时间戳为 `1577596455000L`，若使用如下代码将时间戳转为日期，将得到错误的值。

```java
System.err.println(new SimpleDateFormat("YYYY-MM-dd", Locale.US).format(1577596455000L));
```

运行代码输出 `2020-12-29`。

解决方法为将上述代码中的 `YYYY-MM-dd` 换为 `yyyy-MM-dd`。

`YYYY` 是 `week-based-year`，表示当天所在的周属于的年份，一周从周日开始，周六结束，只要本周跨年，那么这周就算入下一年。时间戳 `1577596455000L` 所属为周日，所以导致了此问题。