# 设计模式 —— 原型模式


原型模式是一种创建型设计模式，使你能够复制已有对象，而又无需使代码依赖它们所属的类。

---

## 原型模式结构

### 原型模式基本实现

{{< image src="/images/design_patterns/04_01.jpg" caption="原型模式基本实现结构图" title="原型模式基本实现结构图" >}}

1. 原型（Prototype）：声明克隆方法。
2. 具体原型（Concrete Prototype）：实现克隆方法。
3. 客户端（Client）：客户端可以调用克隆方法来复制实现了原型接口的任何对象。

### 原型模式注册表实现

{{< image src="/images/design_patterns/04_02.jpg" caption="原型模式注册表实现结构图" title="原型模式注册表实现结构图" >}}

- 原型注册表（Prototype Registry）：供了一种访问常用原型的简单方法，其中存储了一系列可供随时复制的预生成对象。

---

## 原型模式的应用场景

1. 需要复制一些对象，同时又希望代码独立于这些对象所属的具体类，可以使用原型模式。

2. 如果子类的区别仅在于其对象的初始化方式，可以使用该模式来减少子类的数量。

---

## 原型模式的优缺点

### 优点

1. 可以克隆对象，而无需与它们所属的具体类相耦合。

2. 可以克隆预生成原型， 避免反复运行初始化代码。

3. 可以更方便地生成复杂对象。

4. 可以用继承以外的方式来处理复杂对象的不同配置。

### 缺点

克隆包含循环引用的复杂对象可能会非常麻烦。

---

## 代码示例

C# 的 ICloneable 接口就是立即可用的原型模式。

``` C#
using Prototype.Conceptual;

Client.Main();

namespace Prototype.Conceptual
{
    public class Person
    {
        public int Age;
        public DateTime BirthDate;
        public string Name;
        public IdInfo IdInfo;

        public Person(int age, DateTime birthDate, string name, IdInfo idInfo)
        {
            Age = age;
            BirthDate = birthDate;
            Name = name;
            IdInfo = idInfo;
        }

        public Person ShallowCopy()
        {
            return (Person)MemberwiseClone();
        }

        public Person DeepCopy()
        {
            Person clone = (Person)MemberwiseClone();
            clone.IdInfo = new IdInfo(IdInfo.IdNumber);
            clone.Name = Name;
            return clone;
        }
    }

    public class IdInfo
    {
        public int IdNumber;

        public IdInfo(int idNumber)
        {
            IdNumber = idNumber;
        }
    }

    public class Client
    {
        public static void Main()
        {
            var age = 42;
            var birthDate = Convert.ToDateTime("1977-01-01");
            var name = "Jack Daniels";
            var idInfo = new IdInfo(666);
            var p1 = new Person(age, birthDate, name, idInfo);

            // Perform a shallow copy of p1 and assign it to p2.
            Person p2 = p1.ShallowCopy();
            // Make a deep copy of p1 and assign it to p3.
            Person p3 = p1.DeepCopy();

            // Display values of p1, p2 and p3.
            Console.WriteLine("Original values of p1, p2, p3:");
            Console.WriteLine("   p1 instance values: ");
            DisplayValues(p1);
            Console.WriteLine("   p2 instance values:");
            DisplayValues(p2);
            Console.WriteLine("   p3 instance values:");
            DisplayValues(p3);

            // Change the value of p1 properties and display the values of p1,
            // p2 and p3.
            p1.Age = 32;
            p1.BirthDate = Convert.ToDateTime("1900-01-01");
            p1.Name = "Frank";
            p1.IdInfo.IdNumber = 7878;
            Console.WriteLine("\nValues of p1, p2 and p3 after changes to p1:");
            Console.WriteLine("   p1 instance values: ");
            DisplayValues(p1);
            Console.WriteLine("   p2 instance values (reference values have changed):");
            DisplayValues(p2);
            Console.WriteLine("   p3 instance values (everything was kept the same):");
            DisplayValues(p3);
        }

        public static void DisplayValues(Person p)
        {
            Console.WriteLine("      Name: {0:s}, Age: {1:d}, BirthDate: {2:MM/dd/yy}",
                p.Name, p.Age, p.BirthDate);
            Console.WriteLine("      ID#: {0:d}", p.IdInfo.IdNumber);
        }
    }
}

```

---

