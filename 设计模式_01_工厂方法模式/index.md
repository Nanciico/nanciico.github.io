# 设计模式 —— 工厂方法模式


工厂方法模式是一种创建型设计模式，其在**父类**中提供一个创建对象的方法，允许子类决定实例化对象的类型。

---

## 工厂方法模式结构

{{< image src="/images/design_patterns/01_01.jpg" caption="工厂方法模式结构图" title="工厂方法模式结构图" >}}

1. 产品（Product）：对接口进行声明。
2. 具体产品（Concrete Products）：产品接口的不同实现。
3. 创建者（Creator）：声明返回产品对象的工厂方法。
4. 具体创建者（Concrete Creators）：重写基础工厂方法，使其返回不同类型的产品。并不一定每次调用工厂方法都会创建新的实例。工厂方法也可以返回缓存、对象池或其他来源的已有对象。

---

## 工厂方法模式的应用场景

1. 在编写代码的过程中，如果无法预知对象确切类别及其依赖关系时，可使用工厂方法。工厂方法将创建产品的代码与实际使用产品的代码分离，从而能在不影响其他代码的情况下扩展产品创建部分代码。

2. 希望用户能扩展你软件库或框架的内部组件，可使用工厂方法。

3. 希望复用现有对象来节省系统资源，而不是每次都重新创建对象，可使用工厂方法。

---

## 工厂方法模式的优缺点

### 优点

1. 可以避免创建者和具体产品之间的紧密耦合。

2. 单一职责原则。你可以将产品创建代码放在程序的单一位置，从而使得代码更容易维护。

3. 开闭原则。无需更改现有客户端代码，你就可以在程序中引入新的产品类型。

### 缺点

应用工厂方法模式需要引入许多新的子类，代码可能会因此变得更复杂。

---

## 代码示例

工厂方法模式识别方法：工厂方法可通过构建方法来识别，它会创建具体类的对象，但以抽象类型或接口的形式返回这些对象。

``` csharp
using FactoryMethod.Conceptual;

Client.Main();

namespace FactoryMethod.Conceptual
{
    // The Creator class declares the factory method that is supposed to return
    // an object of a Product class.
    abstract class Creator
    {
        public abstract IProduct FactoryMethod();

        // Creator's primary responsibility is not creating products.
        // Usually, it contains some core business logic that relies on Product objects,
        // returned by the factory method. Subclasses can indirectly change that business logic
        // by overriding the factory method and returning a different type of
        // product from it.
        public string SomeOperation()
        {
            var product = FactoryMethod();

            var result = "Creator: The same creator's code has just worked with "
                + product.Operation();

            return result;
        }
    }

    class ConcreteCreator1 : Creator
    {
        // Note that the signature of the method still uses the abstract product
        // type, even though the concrete product is actually returned from the
        // method. This way the Creator can stay independent of concrete product
        // classes.
        public override IProduct FactoryMethod()
        {
            return new ConcreteProduct1();
        }
    }

    class ConcreteCreator2 : Creator
    {
        public override IProduct FactoryMethod()
        {
            return new ConcreteProduct2();
        }
    }

    // The Product interface declares the operations that all concrete products
    // must implement.
    public interface IProduct
    {
        string Operation();
    }

    class ConcreteProduct1 : IProduct
    {
        public string Operation()
        {
            return "{Result of ConcreteProduct1}";
        }
    }

    class ConcreteProduct2 : IProduct
    {
        public string Operation()
        {
            return "{Result of ConcreteProduct2}";
        }
    }


    class Client
    {
        public static void Main()
        {
            Console.WriteLine("App: Launched with the ConcreteCreator1.");
            ClientCode(new ConcreteCreator1());

            Console.WriteLine();

            Console.WriteLine("App: Launched with the ConcreteCreator2.");
            ClientCode(new ConcreteCreator2());
        }

        public static void ClientCode(Creator creator)
        {
            Console.WriteLine("Client: I'm not aware of the creator's class," +
                "but it still works.\n" + creator.SomeOperation());
        }
    }
}
```

---

