# 设计模式 —— 适配器模式


适配器模式是一种结构型设计模式，它能使接口不兼容的对象能够相互合作。

---

## 适配器模式结构

### 对象适配器

适配器实现了其中一个对象的接口，并对另一个对象进行封装。

{{< image src="/images/design_patterns/06_01.jpg" caption="对象适配器结构图" title="对象适配器结构图" >}}

1. 客户端（Client）：包含当前程序业务逻辑的类。客户端代码只需通过接口与适配器交互即可，无需与具体的适配器类耦合。
2. 客户端接口（Client Interface）：描述了其他类与客户端代码合作时必须遵循的协议。
3. 服务（Service）：服务中有一些功能类（通常来自第三方或遗留系统）。客户端与其接口不兼容，因此无法直接调用其功能。
4. 适配器（Adapter）是一个可以同时与客户端和服务交互的类：它在实现客户端接口的同时封装了服务对象。适配器接受客户端通过适配器接口发起的调用，并将其转换为适用于被封装服务对象的调用。

### 类适配器

适配器同时继承两个对象的接口。这种方式仅能在支持多重继承的编程语言中实现，例如 C++。

{{< image src="/images/design_patterns/06_02.jpg" caption="类适配器结构图" title="类适配器结构图" >}}

- 类适配器不需要封装任何对象，因为它同时继承了客户端和服务的行为。适配功能在重写的方法中完成。最后生成的适配器可替代已有的客户端类进行使用。

---

## 适配器模式的应用场景

- 当你希望使用某个类，但是其接口与其他代码不兼容时，可以使用适配器类。适配器模式允许你创建一个中间层类，其可作为代码与遗留类、第三方类或提供怪异接口的类之间的转换器。

---

## 适配器模式的优缺点

### 优点

1. 符合单一职责原则，可以将接口或数据转换代码从程序主要业务逻辑中分离。

2. 开闭原则。只要客户端代码通过客户端接口与适配器进行交互，你就能在不修改现有客户端代码的情况下在程序中添加新类型的适配器。

### 缺点

- 代码整体复杂度增加，因为你需要新增一系列接口和类。

---

## 代码示例

适配器可以通过以不同抽象或接口类型实例为参数的构造函数来识别。当适配器的任何方法被调用时，它会将参数转换为合适的格式，然后将调用定向到其封装对象中的一个或多个方法。

``` C#
using Adapter.Conceptual;

Client.Main();

namespace Adapter.Conceptual
{
    // The Target defines the domain-specific interface used by the client code.
    public interface ITarget
    {
        string GetRequest();
    }

    // The Adaptee contains some useful behavior, but its interface is
    // incompatible with the existing client code. The Adaptee needs some
    // adaptation before the client code can use it.
    class Adaptee
    {
        public string GetSpecificRequest()
        {
            return "Specific request.";
        }
    }

    // The Adapter makes the Adaptee's interface compatible with the Target's
    // interface.
    class Adapter : ITarget
    {
        private readonly Adaptee _adaptee;

        public Adapter(Adaptee adaptee)
        {
            _adaptee = adaptee;
        }

        public string GetRequest()
        {
            return $"This is '{_adaptee.GetSpecificRequest()}'";
        }
    }

    public class Client
    {
        public static void Main()
        {
            var adaptee = new Adaptee();
            var target = new Adapter(adaptee);

            Console.WriteLine("Adaptee interface is incompatible with the client.");
            Console.WriteLine("But with adapter client can call it's method.");

            Console.WriteLine(target.GetRequest());
        }
    }
}
```

---

