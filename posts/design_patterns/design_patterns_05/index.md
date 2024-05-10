# 设计模式 —— 单例模式


单例模式是一种创建型设计模式，让你能够保证一个类只有一个实例，并提供一个访问该实例的全局节点。

单例模式解决了两个问题：

1. 保证一个类只有一个实例；
2. 为该实例提供一个全局访问节点。

单例模式类，既需要提供创建实例的操作，又需要提供获取实例的操作，因此**违反了单一职责原则**。

---

## 单例模式结构

{{< image src="/images/design_patterns/04_01.jpg" caption="单例模式结构结构图" title="单例模式结构结构图" >}}

- 单例（Singleton）：单例类声明 get­Instance 静态方法来返回其所属类的一个相同实例。单例的构造函数必须对客户端代码隐藏。调用 getInstance 方法必须是获取单例对象的唯一方式。

---

## 单例模式的应用场景

程序中的某个类对于所有客户端只有一个可用的实例，可以使用单例模式。

---

## 单例模式的优缺点

### 优点

1. 保证一个类只有一个实例。

2. 获得了一个指向该实例的全局访问节点。

3. 仅在首次请求单例对象时对其进行初始化。

### 缺点

1. 违反了单一职责原则。

2. 单例模式可能掩盖不良设计，比如程序各组件之间相互了解过多等。

3. 该模式在多线程环境下需要进行特殊处理，避免多个线程多次创建单例对象。

4. 单例的客户端代码单元测试可能会比较困难。

---

## 代码示例

### 基础单例

``` C#
using Singleton.Conceptual;

Client.Main();

namespace Singleton.Conceptual
{
    // The Singleton class defines the `GetInstance` method that serves as an
    // alternative to constructor and lets clients access the same instance of
    // this class over and over.

    // EN : The Singleton should always be a 'sealed' class to prevent class
    // inheritance through external classes and also through nested classes.
    public sealed class Singleton
    {
        // The Singleton's constructor should always be private to prevent
        // direct construction calls with the `new` operator.
        private Singleton() { }

        private static Singleton? _instance;

        // This is the static method that controls the access to the singleton
        // instance. On the first run, it creates a singleton object and places
        // it into the static field. On subsequent runs, it returns the client
        // existing object stored in the static field.
        public static Singleton GetInstance()
        {
            _instance ??= new Singleton();
            return _instance;
        }

        // Finally, any singleton should define some business logic, which can
        // be executed on its instance.
        public void SomeBusinessLogic()
        {
            // ...
        }
    }

    public class Client
    {
        public static void Main()
        {
            // The client code.
            var s1 = Singleton.GetInstance();
            var s2 = Singleton.GetInstance();

            if (s1 == s2)
            {
                Console.WriteLine("Singleton works, both variables contain the same instance.");
            }
            else
            {
                Console.WriteLine("Singleton failed, variables contain different instances.");
            }
        }
    }
}
```

### 线程安全单例

``` C#
using Singleton.Conceptual;

Client.Main();

namespace Singleton.Conceptual
{
    // This Singleton implementation is called "double check lock". It is safe
    // in multithreaded environment and provides lazy initialization for the
    // Singleton object.
    public sealed class Singleton
    {
        private static readonly object _locker = new();

        public string Value { get; }

        private Singleton(string value)
        {
            Value = value;
        }

        private static Singleton? _instance;


        public static Singleton GetInstance(string value)
        {
            if (_instance == null)
            {
                lock (_locker)
                {
                    _instance ??= new Singleton(value);
                }
            }
            return _instance;
        }
    }

    public class Client
    {
        public static void Main()
        {
            Console.WriteLine(
                "{0}\n{1}\n\n{2}\n",
                "If you see the same value, then singleton was reused (yay!)",
                "If you see different values, then 2 singletons were created (booo!!)",
                "RESULT:"
            );

            var process1 = new Thread(() =>
            {
                TestSingleton("FOO");
            });
            var process2 = new Thread(() =>
            {
                TestSingleton("BAR");
            });

            process1.Start();
            process2.Start();

            process1.Join();
            process2.Join();
        }

        public static void TestSingleton(string value)
        {
            Singleton singleton = Singleton.GetInstance(value);
            Console.WriteLine(singleton.Value);
        }
    }
}
```

---

