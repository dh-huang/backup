# Java反射(二) - 泛型

**什么是Java泛型？**

泛型的设计是为了应用在Java的类型系统，**提供"用类型或者方法操作各种类型的对象从而提供编译期的类型安全功能(原文：a type or method to operate on objects of various types while providing compile-time type safety)"**。**泛型的一个最大的优点就是：提供编译期的类型安全**。

- **Java虚拟机中不存在泛型，只有普通的类和方法，但是字节码中存放着泛型相关的信息**，这也就是为什么我们能够通过反射**有限的**获取到泛型信息

- 所有的类型参数都使用它们的限定类型替换（**泛型擦除**）

- 桥方法(Bridge Method)由编译器合成，用于保持多态(**Java虚拟机利用方法的参数类型、方法名称和方法返回值类型确定一个方法**)

- 为了保持类型的安全性，必要时需要进行类型的强制转换

比如在使用各种`Json`序列化框架的时候，可能会遇到的一个问题（以`FastJson`为例）：

```java
// 这样的数据结构，反序列话的时候，如何能得到正确的Data?
public class Result {
    public T data;
}

public class Data {
    public String a;    
    public Long b;
}

String json = ...;
// 为什么一定要通过TypeReference的方式才能获取到真正需要的类型信息?
TypeReference ref = new TypeReference>() {};
Result result = JSON.parseObject(json, ref.getType);
```


