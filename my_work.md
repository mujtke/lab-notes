## 2.28 MHP分析

一种数据竞争检测思路：

* 检测访问对：确定race的候选
* 线程模型：用于判断是否有可能同时发生，即[MHP分析](https://dl.acm.org/doi/pdf/10.1145/2892208.2897144)

MHP分析：

*improved MHP Analysis* （x10语言）

调用happe-before分析



## 2.29 初始的race方案

<img src="https://raw.githubusercontent.com/mujtke/pics/main/IMG_20220302_111908.jpg" alt="图片1" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/mujtke/pics/main/IMG_20220302_112017.jpg" alt="图片2" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/mujtke/pics/main/IMG_20220302_112039.jpg" alt="图片3" style="zoom:50%;" />



## 3.3

combat采用不同的配置文件，对thread_test09进行测试，结果为：

[lockator-threadmodular.properties](/home/mujueke/Documents/Cpachecker/cpachecker-CPALockator-combat-mode/config) : *UNKNOWN*

[lockator-threadmodular-linux.properties](/home/mujueke/Documents/Cpachecker/cpachecker-CPALockator-combat-mode/config) : *TRUE*

[lockator-threadmodular-shared.properties](/home/mujueke/Documents/Cpachecker/cpachecker-CPALockator-combat-mode/config): *TRUE*



## 3.4

[lockator-lws_single_modified.properties](/home/mujueke/Documents/Cpachecker/cpachecker-CPALockator-combat-mode/config/MyConfigure/both) : UNKNOWN

lockator不支持同一个线程创建多次





## 3.5

docker连接服务器失败



## 3.7

将BAM取消之后，使用普通的PredicateCPA，能够运行，但是结果不正确：

**thread_test14_modified_configfile**（取消了BAM之后的的总体配置文件）

configfile:[llws_withoutBAMPredicates.properties](/home/mujueke/Documents/Cpachecker/cpachecker-CPALockator-combat-mode/config/MyConfigure/both)

* 对于***thread_test07.i***：

验证结果为*：TRUE*

原因是在`needToDumpUsages`中，会将thread2中的对全局变量b进行访问的记录去除。同时这也证明用于进行后续竞争分析的，是那些需要dump的

在`needToDumpUsage`中添加代码：

```java
if (true) {
	return !predicateState.isAbstractionState() || (predicateState.isAbstractionState()
                && !predicateState.getAbstractionFormula().isFalse());
}
```

之后的验证结果为：*UNKNOWN*，同时会不断细化

(`GenericSinglePathRefiner.java`中的`refinePath`要求路径是可达的)

将`GenericSinglePathRefiner.java`中的代码进行更改：

改之前：

```java
if (result.isFalse()) {
        completePrecisions = Iterables.concat(completePrecisions, result.getPrecisions());
        result = wrappedRefiner.performBlockRefinement(pInput);
}
```

改之后：

```java
if (!result.isFalse()) { //第二条路径不可行，则进行细化
        completePrecisions = Iterables.concat(completePrecisions, result.getPrecisions());
        result = wrappedRefiner.performBlockRefinement(pInput);
}
```

修改之后能够得出*FALSE*的结果

~~这里的逻辑有点奇怪，如果`result.isFalse()`的结果true，则说明第二条路径应该是不可达的，此时才应该有下面的细化。同时若结果为false，则说明两条路径都是可达的，此时才不需要添加任何精度~~



* 对于***thread_test14.i***：

验证结果为：TRUE

在`needToDumpUsage`中添加代码之后：

验证结果为：FALSE



## 3.8

| 编号             | 配置文件                                                     | 测试文件                                                     | 修改                                                         | 结果                  |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | --------------------- |
| ${\color{red}1}$ | [llws_withoutBAMPredicates.properties](/home/mujueke/Documents/Cpachecker/cpachecker-CPALockator-combat-mode/config/MyConfigure/both) | thread_test07.c                                              | 在`needToDumpUsages`中添加if(true)语句                       | *UNKNOWN*，不停地细化 |
| ${\color{red}2}$ | [llws_withoutBAMPredicates.properties](/home/mujueke/Documents/Cpachecker/cpachecker-CPALockator-combat-mode/config/MyConfigure/both) | thread_test07.c                                              | 在`needToDumpUsages`中添加if(true)语句<br />&&<br />`GenericSinglePathRefiner`中将`completePrecision`的修改放到if(!result.isFalse)之后 | *UNKNOWN*，不停地细化 |
| ${\color{red}3}$ | [llws_withoutBAMPredicates.properties](/home/mujueke/Documents/Cpachecker/cpachecker-CPALockator-combat-mode/config/MyConfigure/both) | thread_test07.c                                              | `GenericSinglePathRefiner`中将`completePrecision`的修改放到if(!result.isFalse)之后 | *TRUE*                |
| ${\color{red}4}$ | [llws_withoutBAMPredicates.properties](/home/mujueke/Documents/Cpachecker/cpachecker-CPALockator-combat-mode/config/MyConfigure/both) | thread_test07.c                                              |                                                              | *TRUE*                |
| ${\color{red}5}$ | [llws_withoutBAM_15](/home/mujueke/Documents/Cpachecker/lockator-lab/config/MyConfigure/both) | thread_test15.c<br />==不包含创建线程的操作，主要是为了查看重复添加后继的原因== | 移除`UsageTransferRelation`中重复添加的操作                  | ------                |

在$\color{red}2$中，前两次可达图计算生成的arg：

第一次：

<img src="https://raw.githubusercontent.com/mujtke/pics/main/Screenshot%20from%202022-03-08%2011-32-25.png" style="zoom: 80%;" />

第二次：

<img src="https://raw.githubusercontent.com/mujtke/pics/main/Screenshot%20from%202022-03-08%2011-11-38.png" style="zoom:80%;" />

新增的测试，**thread_test15_modified_configFile**中，如果移除`UsageTransferRelation`中重复添加的操作，得到的ARG为：

![](https://raw.githubusercontent.com/mujtke/pics/main/Screenshot%20from%202022-03-08%2017-46-43.png)

所以，**$\color{red}重复添加的操作有问题?$**



## 3.9

```java
// 多线程程序的CFA克隆  
@Option(secure=true, name="cfa.useCFACloningForMultiThreadedPrograms",
      description="clone functions of the CFA, such that there are several "
          + "identical CFAs for each function, only with different names.")
private boolean useCFACloningForMultiThreadedPrograms = false;

// 线程创建函数的特殊处理
@Option(
    secure = true,
    name = "analysis.threadOperationsTransform",
    description =
        "Replace thread creation operations with a special function calls"
            + "so, any analysis can go through the function"
  )
private boolean enableThreadOperationsInstrumentation = false;
```



==**plan_B_v0**==:

* 不使用BAM对CFA进行展开，将线程创建视作==普通的函数调用==
* 在可达图的探索中一边探索一边抽取访问信息，只要发现一对可行的pair，就进行初步的可行性判断，利用一个全局的list来保存已经抽取到的UsageInfo
* 先不考虑同名线程的情况
* 先利用lockator的方式进行happen-before分析，参考lockator中threadState和LockState相容的情况



