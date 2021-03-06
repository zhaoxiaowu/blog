## 泛型是什么

> 泛型的本质是参数化类型，也就是说所操作的数据类型被指定为一个参数。提供了编译时类型安全检测机制，该机制允许开发者在编译时检测到非法的类型。

不确定的数据类型  提供了编译时类型安全检测机制

## 泛型的好处

之前使用Object

1. 省略了强转的代码
2. 可以把运行时的问题提前到编译时期

## 泛型中通配符

### 常用的 T，E，K，V，？

本质上这些个都是通配符，没啥区别，只不过是编码时的一种约定俗成的东西。比如上述代码中的 T ，我们可以换成 A-Z 之间的任何一个 字母都可以，并不会影响程序的正常运行，但是如果换成其他的字母代替 T ，在可读性上可能会弱一些。**通常情况下，T，E，K，V，？ 是这样约定的：**

- ？ 表示不确定的 java 类型
- T (type) 表示具体的一个java类型
- K V (key value) 分别代表java键值中的Key Value
- E (element) 代表Element

### ？ **无界通配符**

> 对于不确定或者不关心实际要操作的类型，可以使用无限制通配符（尖括号里一个问号，即 <?> ）

### 上界通配符 < ? extends E>

> 上届：用 extends 关键字声明，表示参数化的类型可能是所指定的类型，或者是此类型的子类。

### 下界通配符 < ? super E>

下界: 用 super 进行声明，表示参数化的类型可能是所指定的类型，或者是此类型的父类型，直至 Object

在类型参数中使用 super 表示这个泛型中的参数必须是 E 或者 E 的父类。

```
private <T> void test(List<? super T> dst, List<T> src){
    for (T t : src) {
        dst.add(t);
    }
}

public static void main(String[] args) {
    List<Dog> dogs = new ArrayList<>();
    List<Animal> animals = new ArrayList<>();
    new Test3().test(animals,dogs);
}
// Dog 是 Animal 的子类
class Dog extends Animal {

}
 
```

dst 类型 “大于等于” src 的类型，这里的“大于等于”是指 dst 表示的范围比 src 要大，因此装得下 dst 的容器也就能装 src 。

### ？ 和 T 的区别

> ？和 T 都表示参数化类型
>
> ?是通配符（占位符），可以表示任意一个，T只是一种替代，只能表示其中一个
> 假设有A，B，C三个类
> <?>可以是A，B，C任意一个,每一个<?>之间没有关联
> <T>如果确定了是A那之后的都是A，每一个<T>代表的是相同的
> 它们都是在类型不确定的时候或者为了支持多种类型的一种替代写法

所以可以对 T 进行操作，但是对 ？ 不行，比如如下这种 ：

```
// 可以
T t = operate();

// 不可以
？ car = operate();
 
```

**简单总结下：**

**T 是一个 确定的 类型，通常用于泛型类和泛型方法的定义，？是一个 不确定 的类型，通常用于泛型方法的调用代码和形参，不能用于定义类和泛型方法。**



有两种情况可以传入"?"：

1、使用过程中仅用到Object的方法，**跟T的具体类型无关，**像equals()等等，因为任何一个泛型肯定是Object的子类；

2、**使用过程中不依赖于泛型**。最典型的是Class<?>，因为Class类的方法大多跟泛型无关。



#### 区别1：通过 T 来 确保 泛型参数的一致性

```
// 通过 T 来 确保 泛型参数的一致性
public <T extends Number> void
test(List<T> dest, List<T> src)

//通配符是 不确定的，所以这个方法不能保证两个 List 具有相同的元素类型
public void
test(List<? extends Number> dest, List<? extends Number> src)
```

#### 区别2：类型参数可以多重限定而通配符不行

![image-20200831144734529](..\images\image-20200831144734529.png)

使用 & 符号设定多重边界（Multi Bounds)，指定泛型类型 T 必须是 MultiLimitInterfaceA 和 MultiLimitInterfaceB 的共有子类型，此时变量 t 就具有了所有限定的方法和属性。对于通配符来说，因为它不是一个确定的类型，所以不能进行多重限定。

#### 区别3：通配符可以使用超类限定而类型参数不行

类型参数 T 只具有 一种 类型限定方式：

```
T extends A
 
```

但是通配符 ? 可以进行 两种限定：

```
? extends A
? super A
```

## `Class<T>` 和 `Class<?>` 区别

前面介绍了 ？ 和 T 的区别，那么对于，`Class<T>` 和 `<Class<?>` 又有什么区别呢？
`Class<T>` 和 `Class<?>`

最常见的是在反射场景下的使用，这里以用一段发射的代码来说明下。

```
// 通过反射的方式生成  multiLimit 
// 对象，这里比较明显的是，我们需要使用强制类型转换
MultiLimit multiLimit = (MultiLimit)
Class.forName("com.glmapper.bridge.boot.generic.MultiLimit").newInstance();
 
```

对于上述代码，在运行期，如果反射的类型不是 MultiLimit 类，那么一定会报 java.lang.ClassCastException 错误。

对于这种情况，则可以使用下面的代码来代替，使得在在编译期就能直接 检查到类型的问题：

![image-20200831150212756](..\images\image-20200831150212756.png)

`Class<T>` 在实例化的时候，T 要替换成具体类。`Class<?>` 它是个通配泛型，? 可以代表任何类型，所以主要用于声明时的限制情况。比如，我们可以这样做申明：

```
// 可以
public Class<?> clazz;
// 不可以，因为 T 需要指定类型
public Class<T> clazzT;
```

所以当不知道定声明什么类型的 Class 的时候可以定义一 个Class<?>。

那如果也想 `public Class<T> clazzT;` 这样的话，就必须让当前的类也指定 T ，

```
public class Test3<T> {
    public Class<?> clazz;
    // 不会报错
    public Class<T> clazzT;

```

## 参考

[<?>与<T>的区别](https://www.cnblogs.com/youmingDDD/p/9503218.html)

[List<?>和List<T>的区别？](https://www.zhihu.com/question/31429113)

[聊一聊-JAVA 泛型中的通配符 T，E，K，V，？](https://juejin.im/post/6844903917835419661#heading-7)