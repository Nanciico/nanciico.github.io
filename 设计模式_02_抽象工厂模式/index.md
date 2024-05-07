# 设计模式 —— 抽象工厂模式


抽象工厂模式是一种创建型设计模式，它能创建一系列相关的对象，而无需指定其具体类。

---

## 抽象工厂模式结构

{{< image src="/images/design_patterns/02_01.jpg" caption="抽象工厂模式结构图" title="抽象工厂模式结构图" >}}

1. 抽象产品（Abstract Product）：为构成系列产品的一组不同但相关的产品声明接口。
2. 具体产品（Concrete Product）：是抽象产品的多种不同类型实现。所有变体都必须实现相应的抽象产品。
3. 抽象工厂（Abstract Factory）：抽象工厂接口声明了一组创建各种抽象产品的方法。
4. 具体工厂（Concrete Factory）：实现抽象工厂的构建方法。每个具体工厂都对应特定产品变体，且仅创建此种产品变体。具体工厂会对具体产品进行初始化，其构建方法签名必须返回相应的抽象产品。
5. 客户端（Client）：客户端只需通过抽象接口调用工厂和产品对象，就能与任何具体工厂/产品变体交互。

---

## 抽象工厂模式的应用场景

1. 如果代码需要与多个不同系列的相关产品交互，但是由于无法提前获取相关信息，或者出于对未来扩展性的考虑，你不希望代码基于产品的具体类进行构建，在这种情况下，你可以使用抽象工厂。

2. 如果你有一个基于一组抽象方法的类，且其主要功能因此变得不明确，那么在这种情况下可以考虑使用抽象工厂模式。

{{< admonition type=tip open=true >}}

在设计良好的程序中，每个类**仅负责一件事**。如果一个类与多种类型产品交互，就可以考虑将工厂方法抽取到独立的工厂类或具备完整功能的抽象工厂类中。

{{< /admonition >}}

---

## 抽象工厂模式的优缺点

### 优点

1. 可以确保同一工厂生成的产品相互匹配。

2. 可以避免客户端和具体产品代码的耦合。

3. 单一职责原则。你可以将产品生成代码抽取到同一位置，使得代码易于维护。

4. 开闭原则。向应用程序中引入新产品变体时，你无需修改客户端代码。

### 缺点

由于采用该模式需要向应用中引入众多接口和类，代码可能会比之前更加复杂。

---

## 代码示例

``` csharp
using AbstractFactory.Conceptual;

Client.Main();

namespace AbstractFactory.Conceptual
{
    // The Abstract Factory interface declares a set of methods that return
    // different abstract products.
    public interface IAbstractFactory
    {
        IAbstractProductA CreateProductA();

        IAbstractProductB CreateProductB();
    }

    // Concrete Factories produce a family of products that belong to a single
    // variant.
    class ConcreteFactory1 : IAbstractFactory
    {
        public IAbstractProductA CreateProductA()
        {
            return new ConcreteProductA1();
        }

        public IAbstractProductB CreateProductB()
        {
            return new ConcreteProductB1();
        }
    }

    // Each Concrete Factory has a corresponding product variant.
    class ConcreteFactory2 : IAbstractFactory
    {
        public IAbstractProductA CreateProductA()
        {
            return new ConcreteProductA2();
        }

        public IAbstractProductB CreateProductB()
        {
            return new ConcreteProductB2();
        }
    }

    // Each distinct product of a product family should have a base interface.
    // All variants of the product must implement this interface.
    public interface IAbstractProductA
    {
        string UsefulFunctionA();
    }

    // Concrete Products are created by corresponding Concrete Factories.
    class ConcreteProductA1 : IAbstractProductA
    {
        public string UsefulFunctionA()
        {
            return "The result of the product A1.";
        }
    }

    class ConcreteProductA2 : IAbstractProductA
    {
        public string UsefulFunctionA()
        {
            return "The result of the product A2.";
        }
    }

    // Here's the the base interface of another product.
    public interface IAbstractProductB
    {
        // Product B is able to do its own thing...
        string UsefulFunctionB();

        // Product B also can collaborate with the ProductA.
        string AnotherUsefulFunctionB(IAbstractProductA collaborator);
    }

    // Concrete Products are created by corresponding Concrete Factories.
    class ConcreteProductB1 : IAbstractProductB
    {
        public string UsefulFunctionB()
        {
            return "The result of the product B1.";
        }

        public string AnotherUsefulFunctionB(IAbstractProductA collaborator)
        {
            var result = collaborator.UsefulFunctionA();

            return $"The result of the B1 collaborating with the ({result})";
        }
    }

    class ConcreteProductB2 : IAbstractProductB
    {
        public string UsefulFunctionB()
        {
            return "The result of the product B2.";
        }

        public string AnotherUsefulFunctionB(IAbstractProductA collaborator)
        {
            var result = collaborator.UsefulFunctionA();

            return $"The result of the B2 collaborating with the ({result})";
        }
    }

    // The client code works with factories and products only through abstract
    // types: AbstractFactory and AbstractProduct. This lets you pass any
    // factory or product subclass to the client code without breaking it.
    class Client
    {
        public static void Main()
        {
            // The client code can work with any concrete factory class.
            Console.WriteLine("Client: Testing client code with the first factory type...");
            ClientMethod(new ConcreteFactory1());

            Console.WriteLine();

            Console.WriteLine("Client: Testing the same client code with the second factory type...");
            ClientMethod(new ConcreteFactory2());
        }

        public static void ClientMethod(IAbstractFactory factory)
        {
            var productA = factory.CreateProductA();
            var productB = factory.CreateProductB();

            Console.WriteLine(productB.UsefulFunctionB());
            Console.WriteLine(productB.AnotherUsefulFunctionB(productA));
        }
    }
}

```

---

