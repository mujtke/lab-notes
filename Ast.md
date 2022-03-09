## 1. cfa.ast

ast提供了关于程序表达式和语句的必要信息.

```java
ast
  |________ c 
  |________ java
  |________ 一些共同的抽象类
```



表达式`Expression`中，`accept()`方法接受特定的`visitor`实例，返回值为`visitor`中`visit()`方法的返回值：对`Expression`进行`visit`的结果，例如`Formular`

`visitor`中针对不同的表达式类型，对`visit()`方法进行了重载.



### 1.1 访问者模式visitor

访问者模式（Visitor）是一种操作一组对象的操作，它的目的是不改变对象的定义，但允许新增不同的访问者，来定义新的操作.

[访问者模式](https://www.liaoxuefeng.com/wiki/1252599548343744/1281319659110433)



### 1.2 visitor类

使用访问者模式进行类似

```java
expression.accept(visitor)
visitor.visit(expression)
```

对表达式的操作.
