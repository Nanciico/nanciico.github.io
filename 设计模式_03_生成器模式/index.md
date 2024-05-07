# 设计模式 —— 生成器模式


生成器模式是一种创建型设计模式，使你能够分步骤创建复杂对象。该模式允许你使用相同的创建代码生成不同类型和形式的对象。

---

## 生成器模式结构

{{< image src="/images/design_patterns/03_01.jpg" caption="生成器模式结构图" title="生成器模式结构图" >}}

1. 生成器（Builder）：生成器接口负责声明在所有类型生成器中通用的产品构造步骤。
2. 具体生成器（Concrete Builders）：提供构造过程的不同实现。
3. 产品（Products）：最终生成的对象。
4. 主管（Director）：定义调用构造步骤的顺序，这样你就可以创建和复用特定的产品配置。
5. 客户端（Client）：将某个生成器对象与主管类关联。

---

## 生成器模式的应用场景

1. 需要简化构造函数的重载方法时，可以使用生成器模式。

2. 如果希望使用代码创建不同形式的产品，它们的制造过程相似且仅有细节上的差异，可使用生成器模式。

---

## 生成器模式的优缺点

### 优点

1. 可以分步创建对象，暂缓创建步骤或递归运行创建步骤。

2. 生成不同形式的产品时，你可以复用相同的制造代码。

3. 单一职责原则。你可以将复杂构造代码从产品的业务逻辑中分离出来。

### 缺点

由于采用该模式需要向应用中引入众多接口和类，代码可能会比之前更加复杂。

---

## 代码示例

``` csharp
using Builder.Conceptual;

Client.Main();

namespace Builder.Conceptual
{

    // The Builder interface specifies methods for creating the different parts
    // of the Product objects.
    public interface IBuilder
    {
        void BuildPartA();

        void BuildPartB();

        void BuildPartC();
    }

    // The Concrete Builder classes follow the Builder interface and provide
    // specific implementations of the building steps.
    public class ConcreteBuilder : IBuilder
    {
        private Product _product = new();

        // A fresh builder instance should contain a blank product object, which
        // is used in further assembly.
        public ConcreteBuilder()
        {
            Reset();
        }

        public void Reset()
        {
            _product = new Product();
        }

        // All production steps work with the same product instance.
        public void BuildPartA()
        {
            _product.Add("PartA1");
        }

        public void BuildPartB()
        {
            _product.Add("PartB1");
        }

        public void BuildPartC()
        {
            _product.Add("PartC1");
        }

        public Product GetProduct()
        {
            Product result = _product;

            Reset();

            return result;
        }
    }

    // It makes sense to use the Builder pattern only when your products are
    // quite complex and require extensive configuration.
    public class Product
    {
        private readonly List<object> _parts = [];

        public void Add(string part)
        {
            _parts.Add(part);
        }

        public string ListParts()
        {
            string str = string.Empty;

            for (int i = 0; i < _parts.Count; i++)
            {
                str += _parts[i] + ", ";
            }

            str = str.Remove(str.Length - 2); // removing last ",c"

            return "Product parts: " + str + "\n";
        }
    }

    // The Director is only responsible for executing the building steps in a
    // particular sequence. It is helpful when producing products according to a
    // specific order or configuration. Strictly speaking, the Director class is
    // optional, since the client can control builders directly.
    public class Director
    {
        private readonly IBuilder _builder;

        public Director(IBuilder builder)
        {
            _builder = builder;
        }

        // The Director can construct several product variations using the same
        // building steps.
        public void BuildMinimalViableProduct()
        {
            _builder.BuildPartA();
        }

        public void BuildFullFeaturedProduct()
        {
            _builder.BuildPartA();
            _builder.BuildPartB();
            _builder.BuildPartC();
        }
    }

    class Client
    {
        public static void Main()
        {
            // The client code creates a builder object, passes it to the
            // director and then initiates the construction process. The end
            // result is retrieved from the builder object.
            var builder = new ConcreteBuilder();
            var director = new Director(builder);

            Console.WriteLine("Standard basic product:");
            director.BuildMinimalViableProduct();
            Console.WriteLine(builder.GetProduct().ListParts());

            Console.WriteLine("Standard full featured product:");
            director.BuildFullFeaturedProduct();
            Console.WriteLine(builder.GetProduct().ListParts());

            // The Builder pattern can be used without a Director class.
            Console.WriteLine("Custom product:");
            builder.BuildPartA();
            builder.BuildPartC();
            Console.Write(builder.GetProduct().ListParts());
        }
    }
}

```

---

