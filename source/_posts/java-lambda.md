---
title: Java lambda 表达式
date: 2017-04-09
tags:
  - Java
---

### 为什么 Java 需要 Lambda 表达式
我先说下我的结论吧：Lambda 的作用是让函数像变量一样可以作为参数传递。
为了说明这个结论，我们可以看下 [stackoverflow](http://stackoverflow.com/questions/23097484/why-lambda-expression-are-introduced-in-java8) 上的一个回答。
假如我们需要输出一个人的信息，我们有以下代码：
```java
public static void printPersons(
    List<Person> roster, CheckPerson tester) {
    for (Person p : roster) {
        if (tester.test(p)) {
            p.printPerson();
        }
    }
} 

interface CheckPerson {
    boolean test(Person p);
}

class CheckPersonEligibleForSelectiveService implements CheckPerson {
    public boolean test(Person p) {
        return p.gender == Person.Sex.MALE &&
            p.getAge() >= 18 &&
            p.getAge() <= 25;
    }
}
```
<!--more-->
那么你大概会这样使用：
```java
printPersons(roster, new CheckPersonEligibleForSelectiveService());
```
显然的，`Person`是一个重要的类，但是相对而言`CheckPerson`和`CheckPersonEligibleForSelectiveService`就没那么重要。我们使用这两个类只是实现了一个具体的函数而已，如果我们能够将这个函数作为参数传递进去，就可以不用`CheckPersonEligibleForSelectiveService`的帮助了。嗯，这一切我们可以借助 lambda 来实现：
```java
printPersons(
    roster,
    (Person p) -> p.getGender() == Person.Sex.MALE
        && p.getAge() >= 18
        && p.getAge() <= 25
);
```
这样的代码有更好的可读性，而且我们不需要`CheckPersonEligibleForSelectiveService`，也不需要它的对象。下面，我们用匿名类来解决这个问题：
```java
printPersons(
    roster,
    new CheckPerson() {
        public boolean test(Person p) {
            return p.getGender() == Person.Sex.MALE
                && p.getAge() >= 18
                && p.getAge() <= 25;
        }
    }
);
```
但是，你认为有必要为了测试来创建一个对象吗？最重要的是，如果需要传入的是一个函数而不是一个对象，那么我们应该有一种机制可以让我们直接传入函数，这种机制就是 lambda。
### Lambda 语法
Lambda 表达式的语法格式如下：
```java
(parameters) -> expression
or
(parameters) -> { statements; }
```
Lambda 有如下重要的特征：
- 可选类型声明：Java 编译器会根据上下文推断出来
- 可选的参数圆括号：一个参数无需定义圆括号，但多个参数需要定义圆括号
- 可选的大括号：如果代码块仅包含一个表达式，就不需要使用大括号
- 可选`return`关键字：如果代码块中仅包含一个表达式返回值且没有使用大括号包裹，那么无需显式指定`return`
Lambda 写法示例：
```java
interface MathOpreration {
    int operation(int a, int b);
}

interface MathOpreration {
    int operation(int a, int b);
}

interface GreetingService {
    void sayMessage(String msg);
}

// 类型声明
MathOpreration addition = (int a, int b) -> a + b;

// 不用类型声明
MathOpreration subtraction = (a, b) -> a - b;

// 在大括号中指定 return
MathOpreration multiplication = (int a, int b) -> { return a * b; };

// 无需指定 return
MathOpreration division = (int a, int b) -> a / b;

// 只有一个参数，可以不用圆括号
GreetingService greetingService1 = msg -> System.out.println("Hello " + msg);

// 只有一个参数，也可以使用圆括号
GreetingService greetingService2 = (msg) -> System.out.println("Hello " + msg);
```
### 什么是函数式接口
**函数式接口（Functional Interface）**是 Java 8 对一类特殊类型接口的称呼，这类接口只定义了唯一的抽象方法的接口（除了隐含的 Object 对象的公共方法），因此函数式接口也称为**SAM（Single Abstract Method）**类型的接口。
#### 函数式接口的用途
函数式接口主要用在 Lambda 表达式上面。
如下定义了一个函数式接口：
```java
@FunctionalInterface
interface MathOpreration{
    int operation(int a, int b);

}
```
#### 关于@FunctionalInterface注解
Java 8 为函数式接口引入了一个新的注解`@FunctionalInterface`，主要用于错误检查，加上该注解后，当你写的接口不符合函数式接口定义的时候，就会报错。
```java
@FunctionalInterface
interface MathOpreration{
    int operation(int a, int b);
    int anotherOperation();
}
```
在编译的时候会报如下错误：
```java
Error:(8, 35) java: 不兼容的类型: LambdaTest.MathOpreration 不是函数接口
    在 接口 LambdaTest.MathOpreration 中找到多个非覆盖抽象方法
```
#### 函数式接口允许定义默认方法
因为默认方法不是抽象方法，所以是符合函数式接口定义的。
```java
@FunctionalInterface
interface MathOpreration {
    int operation(int a, int b);
    default void doSomething() {
        // do something
    }
}
```
#### 函数式接口允许定义静态方法
因为静态方法不能是抽象方法，只能是一个已经实现了的方法，所以是符合函数式接口的定义的。
```java
@FunctionalInterface
interface MathOpreration {
    int operation(int a, int b);

    static void sayHello() {
        System.out.println("Hello");
    }
}
```
#### 函数式接口可以额外定义多个抽象方法，但这些抽象方法签名必须和 Object 的 public 方法一致
接口最终有确定的类实现，而类的最终父类是 Object，因此函数式接口可以定义 Object 的 public 方法。
```java
@FunctionalInterface
interface MathOpreration {
    int operation(int a, int b);

    String toString(); // same to Object.toString
    int hashCode();    // same to Object.hashCode
    boolean equals(Object o);   // same to Object.equals
}
```
因为接口中定义的方法都是`public`类型的，所以 Object 中非`public`的方法不能放入函数式接口中：
```java
@FunctionalInterface
interface MathOpreration {
    int operation(int a, int b);
    Object clone(); // clone() is a protect method, not public
}
```
### Lambda 表达式与匿名类的区别
#### `this`关键字的指向不同
对于匿名类，关键字`this`表示的是匿名类，而对于 Lambda 表达式，`this`表示 Lambda 的外部类。
#### 编译方法的不同
Java 编译器编译 Lambda 表达式并将他们转化为类里面的私有函数，它使用 Java 7 中新加的`invokedynamic`指令动态绑定该方法。
### 参考文章
1. http://blog.oneapm.com/apm-tech/226.html
2. http://colobu.com/2014/10/28/secrets-of-java-8-functional-interface/
3. http://blog.oneapm.com/apm-tech/226.html

