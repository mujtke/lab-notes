## 5.5 ARG生成中的节点信息和节点颜色

生成过程调用：`ReportGeneration`->`buildArgGrapDate`->`createArgNode`

**`createArgNode`**：

* `argNode.label`:决定dot节点的文字内容

  * ```java
    argState.toDotLabel()
        | // 逐层调用封装状态的toDotLabel()函数
        argState.wrappedState.toDotLabel()
        	|
        	...
    ```

* `argNode.type`:决定dot节点的填充颜色

  * ```java
    determineNodeType(argState)
        |
        argState.shouldBeHighlight()
        | // 逐层调用封装状态的shouldBeHighlight()函数
        argState.wrappedState.shouldBeHighlight()
        	|
        	...
    ```

总结：**==ARG节点要显示特定信息以及颜色，需要对相应的State实现Graphable接口==**，然后实现`toDotLabel()`方法和`shouldBeHighlight()`方法