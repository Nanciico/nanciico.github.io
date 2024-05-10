# 设计模式 —— 桥接模式


桥接模式是一种结构型设计模式，可将一个大类或一系列紧密相关的类拆分为**抽象**和**实现**两个独立的层次结构，从而能在开发时分别使用。

---

## 桥接模式结构

适配器实现了其中一个对象的接口，并对另一个对象进行封装。

{{< image src="/images/design_patterns/07_01.jpg" caption="桥接模式结构图" title="桥接模式结构图" >}}

1. 抽象部分（Abstraction）：提供高层控制逻辑，依赖于完成底层实际工作的实现对象。
2. 实现部分（Implementation）：为所有具体实现声明通用接口。抽象部分仅能通过在这里声明的方法与实现对象交互。
3. 具体实现（Concrete Implementations）：具体实现中包括特定于平台的代码。
4. 精确抽象（Refined Abstraction）：提供控制逻辑的变体。与其父类一样，它们通过通用实现接口与不同的实现进行交互。
5. 客户端（Client）：客户端需要将抽象对象与一个实现对象连接起来。但仅关心如何与抽象部分合作。

---

## 桥接模式的应用场景

1. 如果你想要拆分或重组一个具有多重功能的庞杂类（例如能与多个数据库服务器进行交互的类），可以使用桥接模式。

2. 如果你希望在几个独立维度上扩展一个类， 可使用该模式。

3. 如果你需要在运行时切换不同实现方法， 可使用桥接模式。

---

## 桥接模式的优缺点

### 优点

1. 客户端代码仅与高层抽象部分进行互动，不会接触到平台的详细信息。因此可以创建与平台无关的类和程序。

2. 开闭原则。你可以新增抽象部分和实现部分，且它们之间不会相互影响。

3. 单一职责原则。抽象部分专注于处理高层逻辑，实现部分处理平台细节。

### 缺点

- 对高内聚的类使用该模式可能会让代码更加复杂。

---

## 代码示例

``` C#
using Bridge.Conceptual;

Client.Main();

namespace Bridge.Conceptual
{
    // The Abstraction defines the interface for the "control" part of the two
    // class hierarchies. It maintains a reference to an object of the
    // Implementation hierarchy and delegates all of the real work to this
    // object.
    class Abstraction
    {
        protected IImplementation _implementation;

        public Abstraction(IImplementation implementation)
        {
            _implementation = implementation;
        }

        public virtual string Operation()
        {
            return "Abstract: Base operation with:\n" +
                _implementation.OperationImplementation();
        }
    }

    // You can extend the Abstraction without changing the Implementation
    // classes.
    class ExtendedAbstraction : Abstraction
    {
        public ExtendedAbstraction(IImplementation implementation) : base(implementation)
        {
        }

        public override string Operation()
        {
            return "ExtendedAbstraction: Extended operation with:\n" +
                _implementation.OperationImplementation();
        }
    }

    // The Implementation defines the interface for all implementation classes.
    // It doesn't have to match the Abstraction's interface.
    public interface IImplementation
    {
        string OperationImplementation();
    }

    // Each Concrete Implementation corresponds to a specific platform and
    // implements the Implementation interface using that platform's API.
    class ConcreteImplementationA : IImplementation
    {
        public string OperationImplementation()
        {
            return "ConcreteImplementationA: The result in platform A.\n";
        }
    }

    class ConcreteImplementationB : IImplementation
    {
        public string OperationImplementation()
        {
            return "ConcreteImplementationB: The result in platform B.\n";
        }
    }

    class Client
    {
        public static void Main()
        {
            var abstraction = new Abstraction(new ConcreteImplementationA());
            ClientCode(abstraction);

            Console.WriteLine();

            abstraction = new Abstraction(new ConcreteImplementationB());
            ClientCode(abstraction);
        }

        // Except for the initialization phase, where an Abstraction object gets
        // linked with a specific Implementation object, the client code should
        // only depend on the Abstraction class. This way the client code can
        // support any abstraction-implementation combination.
        public static void ClientCode(Abstraction abstraction)
        {
            Console.Write(abstraction.Operation());
        }
    }
}
```

---

