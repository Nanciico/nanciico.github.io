# 设计模式 —— 组合模式


组合模式是一种结构型设计模式，你可以使用它将对象组合成树状结构，并且能像使用独立对象一样使用它们。

---

## 组合模式结构

{{< image src="/images/design_patterns/08_01.jpg" caption="组合模式结构图" title="组合模式结构图" >}}

1. 组件（Component）：接口描述了树中简单项目和复杂项目所共有的操作。
2. 叶节点（Leaf）：树的基本结构，它不包含子项目。
3. 容器（Container）：又名“组合（Composite）”。是包含叶节点或其他容器等子项目的单位。容器不知道其子项目所属的具体类，它只通过通用的组件接口与其子项目交互。
4. 客户端（Client）：客户端只通过组件接口与所有项目交互。

---

## 组合模式的应用场景

1. 如果你需要实现树状对象结构，可以使用组合模式。组合模式为你提供了两种共享公共接口的基本元素类型：简单叶节点和复杂容器。容器中可以包含叶节点和其他容器。这使得你可以构建树状嵌套递归对象结构。

2. 如果你希望客户端代码以相同方式处理简单和复杂元素，可以使用该模式。

---

## 桥接模式的优缺点

### 优点

1. 可以利用多态和递归机制更方便地使用复杂树结构。

2. 开闭原则。无需更改现有代码，你就可以在应用中添加新元素，使其成为对象树的一部分。

### 缺点

- 对于功能差异较大的类，提供公共接口或许会有困难。

---

## 代码示例

组合可以通过将同一抽象或接口类型的实例放入树状结构的行为方法来轻松识别。

``` csharp
using Composite.Conceptual;

Client.Main();

namespace Composite.Conceptual
{
    // The base Component class declares common operations for both simple and
    // complex objects of a composition.
    abstract class Component
    {
        public Component() { }

        // The base Component may implement some default behavior or leave it to
        // concrete classes (by declaring the method containing the behavior as
        // "abstract").
        public abstract string Operation();

        public virtual void Add(Component component)
        {
            throw new NotImplementedException();
        }

        public virtual void Remove(Component component)
        {
            throw new NotImplementedException();
        }

        // You can provide a method that lets the client code figure out whether
        // a component can bear children.
        public virtual bool IsComposite()
        {
            return true;
        }
    }

    // The Leaf class represents the end objects of a composition. A leaf can't
    // have any children.
    //
    // Usually, it's the Leaf objects that do the actual work, whereas Composite
    // objects only delegate to their sub-components.
    class Leaf : Component
    {
        public override string Operation()
        {
            return "Leaf";
        }

        public override bool IsComposite()
        {
            return false;
        }
    }

    // The Composite class represents the complex components that may have
    // children. Usually, the Composite objects delegate the actual work to
    // their children and then "sum-up" the result.
    class Composite : Component
    {
        protected List<Component> _children = [];

        public override void Add(Component component)
        {
            _children.Add(component);
        }

        public override void Remove(Component component)
        {
            _children.Remove(component);
        }

        // The Composite executes its primary logic in a particular way. It
        // traverses recursively through all its children, collecting and
        // summing their results. Since the composite's children pass these
        // calls to their children and so forth, the whole object tree is
        // traversed as a result.
        public override string Operation()
        {
            int i = 0;
            string result = "Branch(";

            foreach (Component component in _children)
            {
                result += component.Operation();
                if (i != this._children.Count - 1)
                {
                    result += "+";
                }
                i++;
            }

            return result + ")";
        }
    }

    class Client
    {
        public static void Main()
        {
            // This way the client code can support the simple leaf
            // components...
            var leaf = new Leaf();
            Console.WriteLine("Client: I get a simple component:");
            ClientCode(leaf);

            // ...as well as the complex composites.
            var tree = new Composite();
            var branch1 = new Composite();
            branch1.Add(new Leaf());
            branch1.Add(new Leaf());
            var branch2 = new Composite();
            branch2.Add(new Leaf());
            tree.Add(branch1);
            tree.Add(branch2);
            Console.WriteLine("Client: Now I've got a composite tree:");
            ClientCode(tree);

            Console.Write("Client: I don't need to check the components classes even when managing the tree:\n");
            ClientCode2(tree, leaf);
        }

        // The client code works with all of the components via the base
        // interface.
        public static void ClientCode(Component leaf)
        {
            Console.WriteLine($"RESULT: {leaf.Operation()}\n");
        }

        // Thanks to the fact that the child-management operations are declared
        // in the base Component class, the client code can work with any
        // component, simple or complex, without depending on their concrete
        // classes.
        public static void ClientCode2(Component component1, Component component2)
        {
            if (component1.IsComposite())
            {
                component1.Add(component2);
            }

            Console.WriteLine($"RESULT: {component1.Operation()}");
        }
    }
}
```

---

