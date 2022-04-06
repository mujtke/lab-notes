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



## 3.10

==plan_B_v0==

修改了`CEGARAlgorithm`中的逻辑，每次计算出后新的后继状态之后，就判断一下是否存在Unsafe

还没有添加全局的list用于存放访问信息，新添加的`haveUnsafeOrNot`与之前的`refinementNecessary`作用类似，利用reached中的信息判断是否存在可能的race

提取Usage变更为串行方式，先仿照`ConcurrentUsageExtractor`，修改得到可以用于串行提取Usage的`UsageExtractor`



## 3.11

==plan_B_v0==

* `UsageProcessor`中，改变原来利用child节点获取Usage信息的方式，==利用当前节点与父节点之间的边来获取Usage==，将`getUsagesForState`方法改写为`getUsagesForStateByParentState`方法
* `UsageContainer`中，改写了检查container中是否存在Unsafe的逻辑，将`hasUnsafes`方法改写为`hasUnsafesForUsageContainer`，修改了运行逻辑

能够检测出thread_test14.c中的不安全

==refine的过程存在异常==

### refine的修改

==利用refine的过程来初步检查race的可行性==

==如果不进行细化，会存在漏报的问题==



==plan_B_v1==

在==plan_B_v0==的基础上，如果可达图探索完毕之后，没有真实的race，但是存在虚假反例，则进行细化



## 3.14

实现细化的思路：

* 在`UsageReachedSet`中添加变量`newPrecisionFound`用来标记是否有新的谓词产生
* 当可达图探索完成(`!reach.hasWaitingState()`)，且有谓词产生时，从头开始重新计算
* 为清空之前的可达集合，在`CEGARAlgorithm`中添加了`restart`方法
  * 将`processedUnsafes`、`finalPrecision`以及`precisionMap`等变量添加到`UsageReachedSet`中

* 在`restart`中，清理可达集合，添加initialState以及对应的精度

==bug==

细化之后产生的精度没有得到应用？或者说精度一直为空

每次产生的可达图都一样，会不断地循环下去



## 3.15

在`PredicateRefinerAdapter`中添加获取全局精度的方法`GetNewPrecisionGlobal`

改用全局谓词之后可达图的探索发生了变化，但是thread_test07的结果依然为*TRUE*



## 3.16

==**Plan_B_v1**==

在`ARGPathRestorator`中：

* 将`computePath`中`checkRepeatitionOfState`的内容注释掉

* 在`nextPath`中跳过`computeOnePath`的判断

(`nextPath`方法用于在`PathPairIterator`中计算路径)

==thread_test07==的测试结果为*FALSE*

*推测：之前误报的原因应该是在细化的迭代过程中，**后续迭代中没有计算出相应的路径***

==使用全局谓词和局部谓词都能得到相同的结果==，但***使用局部谓词覆盖关系更多***

###### **CEGAR-time初步对比**

|                    | thread_test07 | ~Unsafes/u__linux-concurrency_safety__drivers---net---caif---caif_serial.ko.cil.i~ | ~Unsafes/u__linux-concurrency_safety__drivers---net---can---slcan.ko.cil.i~ |
| :----------------: | :-----------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| ***lockator-lab*** |    0.525s     |         局部谓词：==*TRUE(漏报)*==  全局谓词：2.737s         |                       全局谓词：9.592s                       |
|    ***combat***    |    3.364s     |                            8.646s                            |                         700+(未结束)                         |



## 3.17

==benchmark初步测试对比==

![3.17benchmark测试对比](https://raw.githubusercontent.com/mujtke/pics/main/Screenshot%20from%202022-03-18%2011-53-38.png)

![CPU时间对比图](https://raw.githubusercontent.com/mujtke/pics/main/Screenshot%20from%202022-03-18%2017-41-17.png)

## 3.18

在配置文件[llws_withoutBAM_07_forLabs.properties](/home/mujueke/Documents/Cpachecker/lockator-lab/config/MyConfigure/both)中关闭输出选项之后，benchmark测是的结果如下：

![03.18数据对比图](https://raw.githubusercontent.com/mujtke/pics/main/Screenshot%20from%202022-03-18%2017-42-35.png)

![03.18CPU时间对比](https://raw.githubusercontent.com/mujtke/pics/main/Screenshot%20from%202022-03-18%2017-44-13.png)



## 3.22

lockator-lab中，每次清空可达集合，从头开始的时候，打印一下获取到的usage信息。以Unsafes中的`u__linux-concurrency_safety__drivers---net---ethernet---marvell---pxa168_eth.ko.cil.i`（==误报的一个例子==）为例，首次清空可达集时获取的usage信息（打印的主要是topUsage）如下：

```txt
(?.dev)|---> 	
	READ:[ldv_thread_12:{ldv_thread_12=CREATED_THREAD, ldv_thread_14=PARENT_THREAD, ldv_thread_15=PARENT_THREAD, ldv_thread_16=PARENT_THREAD}, []]
	READ:[ldv_thread_16:{ldv_thread_12=CREATED_THREAD, ldv_thread_14=PARENT_THREAD, ldv_thread_16=CREATED_THREAD}, []]
	READ:[ldv_thread_16:{ldv_thread_12=CREATED_THREAD, ldv_thread_14=PARENT_THREAD, ldv_thread_16=CREATED_THREAD}, [spin_lock]]
```

第二次清空可达集时，usage信息：

```txt
(?.dev)|---> 	
	WRITE:[ldv_thread_12:{ldv_thread_12=CREATED_THREAD}, []]
	READ:[ldv_thread_12:{ldv_thread_12=CREATED_THREAD, ldv_thread_14=PARENT_THREAD, ldv_thread_16=PARENT_THREAD}, []]
	READ:[ldv_thread_16:{ldv_thread_12=CREATED_THREAD, ldv_thread_14=PARENT_THREAD, ldv_thread_16=CREATED_THREAD}, []]
	READ:[ldv_thread_16:{ldv_thread_12=CREATED_THREAD, ldv_thread_14=PARENT_THREAD, ldv_thread_16=CREATED_THREAD}, [spin_lock]]
```



## 3.28

```c
/**
 * 加锁，测试同步之后线程交替对与条件分支的影响
 */ 
#include<pthread.h>

int b = 0, c = 3;

pthread_t t1, t2;
pthread_mutex_t l1;

void *thread1(void *arg) {
	int a = 0;
	
	pthread_mutex_lock(&l1);
	if (c == 4) {
		pthread_mutex_unlock(&l1);
		b = 7;   // 竞争点1
	}
}

void *thread2(void *arg) {
	pthread_mutex_lock(&l1);
	c++;
	pthread_mutex_unlock(&l1);
	b = 8;		// 竞争点2
}

void main() {

	pthread_create(&t2, NULL, thread2, NULL);

	pthread_create(&t1, NULL, thread1, NULL);

	return;
}
```

Plan_B_v1对与该竞争报*TRUE*，即**$ \color{red}{误报}$**

lockator的结果也为误报

初度推断原因：==没有考虑线程交替的影响，直接将线程创建视作函数调用的话有问题==

==尝试使用ThreadingCPA==
