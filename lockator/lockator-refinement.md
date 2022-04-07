**<font color=red>注：</font>梳理时使用的配置文件为：[lockator-Linux-without-shared_single.properties](/home/mujueke/Documents/javaCode/cpachecker-CPALockator-combat-mode/config/MyConfigure/both)**

***

## 1.是否需要细化？

```java
/*******************************************************************************************/
							对于是否需要细化的判断，即是否发现了Unsafe
/*******************************************************************************************/
// CEGARAlgorithm中
// algorithm.run()中，对于是否有需要细化的判断
        // if there is any target state do refinement
        if (refinementNecessary(reached, previousLastState)) {
          refinementSuccessful = refine(reached);
          refinedInPreviousIteration = true;
          // Note, with special options reached set still contains violated properties
          // i.e (stopAfterError = true) or race conditions analysis
        }

refinementNeccessary()	[CEGARAlgorithm]	[globalRefinement变量在输入的配置文件lockator-linux-without-shared_single.properties中有设置]
    	|
    	hasViolateProperties()	[UsageReachedSet，对reachedSet的判断]
    			|			|
    			|			extractUsage()  [ConcurrentUsageExtractor]
    			|						|
    			|						run()	[用ReachedSetExecutor重写Runnable接口，同时重写run()方法]
    			|							|_____ expandUsageAndAdd(argState)	[将ARGState变成类似"READ access to ((ser_release::*ser).dev) with [rtnl_lock]"，相当于提取Usage]
    			|							|_____ **对于当前的argState，如果需要dump，则将提取到的Usages放入container中
    			|							|_____ **否则，将ARGState以及他们的Usage放到stateToUsage中
    			|							|_____ BAM的处理
    			|      
    			return container.hasUnsafe()	[因此那些需要dump的状态才是用来判断是否发生了Unsafe的状态]
    								|
    				   			hasUnsafe()	[UsageContainer]
    								|	|
    								|	saveStableUnsafes()		[UsageContainer]
    								|					|
    								|					calculateUnsafesIfNecessary()	[]	
                          			|												|
    								|												detector.isUnsafe()	[UnsafeDetector]
    								|																|
    								|																isUnsafe(AbstractUsagePointSet)
    								|																							|
    								|																							isUnsafe(UnrefinedUsagePointSet)
    								|																									|
    								|																									isUnsafe(SortedSet<UsagePoint>) 
									|_____ return !stableUnsafe.isEmpty()	[Unsafe为空则表明hasUnsafe为false]
/**
 * calculateUnsafesIfNecessary会将unrefinedIds中存在Unsafe的条目放置到initialUnsafes中，同时unrefinedIds中不存在Unsafe的条目也会被移除
 * stableUnsafes中的一个条目是<某个字段, 对该字段的两次访问组成的pair>
 */
 stableUnsafes的一个条目：
 {StructureFieldIdentifier@11684} "(?.debugfs_tty_dir)" -> {Pair@12720} "(WRITE access to ((debugfs_init::*ser).debugfs_tty_dir) without locks, READ access to ((debugfs_deinit::*ser).debugfs_tty_dir) with [rtnl_lock])"
 key = (?.debugfs_tty_dir)
 value = (WRITE access to ((debugfs_init::*ser).debugfs_tty_dir) without locks, READ access to ((debugfs_deinit::*ser).debugfs_tty_dir) with [rtnl_lock])
    value.first  = "WRITE access to ((debugfs_init::*ser).debugfs_tty_dir) without locks" 
    value.second = "READ access to ((debugfs_deinit::*ser).debugfs_tty_dir) with [rtnl_lock]" 
```

### 1.1 expandUsageAndAdd

==`getUsageForState`都在`expandUsageAndAdd`中完成==

* 向`expandedUsage`中添加key为`covered`的usage（usageInfo），这一步用到了`stateToUsage`
* 向`expandedUsage`中添加key为`parent`的usage（usageInfo），这一步用到了`stateToUsage`

* `getUsageForState(当前的State)`，获取usage之后进行expand

如果`needToDumpUsages`为true，则向container中添加相应的UsageInfo（来自expandUsages = expandUsagesAndAdd(argState)），否则将argState和expandedUsages放到stateToUsage中.

**所以`stateToUsages`应该是已经分析过的状态**

==向container中添加的UsageInfo用来查询是否存在不安全==，UsageInfo包含三个部分：`core`、`point`和`expandedStack`

core包括：`Access`、`CFANode`、`singleIdentifier`、`path`和`keyState`

point包括：`Access`和`compatibleNodes`

expandedStack包含了一些状态：`list<AbstractState>`

### 1.2 calculateUnsafesIfNecessary

```java
   * container在添加usageInfo时，会将identifier单独提取出来
   * 如果该id尚未被添加到unrefinedIds中，
   * 则将UsageInfo添加到该id所对应的UnrefinedUsagePointSet集合中
   * UnrefinedUsagePointSet中包含了对应id的topUsages和UsageInfoSets
   * UsageInfoSets： 集合，key为UsagePoint，value为UsageInfo的集合
```

==calculateUnsafesIfNecessary会将unrefinedIds中存在Unsafe的条目放置到**initialUnsafes**中，同时unrefinedIds中不存在Unsafe的条目也会被移除==

initialUnsafes只保留了unrefinedIds中的key值，即id



```java
isUnsafe(SortedSet<UsagePoint>)	[UnsafeDetector]
    	|
    	isUnsafePair(UsagePoint, UsagePint)	[]
    			|
    			isCompatible()	[根据两个point的compatibleNodes检测两个point是否相容，主要检测的是ThreadState和LockState]
    					|	|______ isCompatible() [ThreadState]
    					|	|______ isCompatible() [LockState]
    					|
    					|__________ 如果相容，则根据config.getUnsafeMode()判断相应的Unsafe类型，例如RACE
    								如果是RACE:则return isRace(point1, point2)
                                        					|
                                        					if (point1.getAccess() == Access.WRITE || point2.getAccess() == Access.WRITE)
                                                                	if (config.ignoreEmptyLockset() && point1.isEmpty() && point2.isEmpty())
                                                                        return false;
																return true;
```

```java
//UsagePoint
UsagePoint@12320} "READ:[ldv_thread_7:{ldv_thread_7=CREATED_THREAD}, []]"
UsagePoint@12319} "WRITE:[main:{}, []]"
    
//UsageInfoSet
{UsagePoint@12319} "WRITE:[main:{}, []]" -> {UsageInfoSet@12327}  size = 1
{UsagePoint@12320} "READ:[ldv_thread_7:{ldv_thread_7=CREATED_THREAD}, []]" -> {UsageInfoSet@12328}  size = 3
```



```java
/*******************************************************************************************/
								extractUsages的过程
/*******************************************************************************************/
extractUsage()	[ConcurrentUsageExtractor]
          | 转移至并行执行的run方法
          run()
			|
			expandUsagesAndAdd()	[ConcurrentUsageExtractor]
                   |         |
                   |        getUsageForState()	[UsageProcessor]
                   |             |		|
                   |             |	    *getUsageForEdge()	[只是getUsageForState中的一种情况，不过是所有Usage的来源，函数返回<Id, Access>的集合]
                   |             |							[getUsageForEdge会根据边的类型来进行不同的处理，例如Statement边，FunctionCall边等]
                   |			 |_____ *creatUsages()	[将获取到的Usage放入result中，作为getUsageForState的返回结果]
				   usage.expand()	[对从getUsageForStateusage中获取到的usages，每一个usage调用expand]																			
// getUsageForState()

// createUsage()	[usageProcessor]
getAllPossibleAliases()		[为某个Id获取所有可能的别名（字面理解）]
filterAliases()				[过滤掉一些别名（字面理解）]
createUsageAndAdd()			[参数与createUsage相同]
        |        |
        |     createUsageInfo()	[UsageInfo，提取出Usage信息，“访问方式”以及“compatibleNode（ThreadState和LockState）”，然后将结果放入result中]
        |
	[Id: "((__list_splice::*next).prev)"]
                | 对应
	[GeneralId: "(?.prev)"]     
	[ComposedIdentifiers: "__list_splice::*next"]
                       | 对应
	[GeneralId: "::*next"]                                                          
```



## 2.细化过程

```java
//关于细化的粗略调用过程
CPAchecker.run()
          |
    runAlgorithm()
          |
    	algorithm.run() ----> 1.algorithm.run()
    					----> 2.refine()
    								|
    								mRefiner.performRefinement()	[CEGARAlgorithm]
    									|
    									performBlockRefinement()	[IdentifierIterator]
    									| 		
    									performBlockRefinemen()		[GenericIterator]
    									| | 两者之间存在相互调用(其实是因为Iterator是逐层封装的)
    									iterate()					[GenericIterator]

*************************
//firstUsage和SecondUsage
firstUsage = "WRITE access to ((caifdev_setup::*serdev).dev) without locks"
secondUsage = "READ access to ((ser_release::*ser).dev) with [rtnl_lock]"
*************************
```

````java
************************************************************************
    	              封装的Refiner和具体细化过程
************************************************************************
配置文件中的refinementChain：/* cpa.usage.refinement.refinementChain = IdentifierIterator, PointIterator, UsageIterator, PathIterator, PredicateRefiner */
algorithm	[CPAChecker]
    |
    algorithm.mRefiner = IdentifierIterator
    	|
    	IdentifierIterator.wrappedRefiner = PointerIterator
    		|
    		PointerIterator.wrappedRefiner = UsagePairIterator
    			|
    			UsagePairIterator.wrappedRefiner = PathPairIterator
    				|
    				PathPairIterator.wrappedRefiner = PredicateRefinerAdapter
    					|
    					PredicateRefinerAdapter.refiner = PredicateCPARefiner
    					PredicateRefinerAdapter.wrappedRefiner = refinementPairStub( this is null)


----> 2.refine()
			|
			mRefiner.performRefinement()	[CEGARAlgorithm]
				|
				performBlockRefinement()	[IdentifierIterator, IdentifierIterator]
				| 		
				performBlockRefinement()	[GenericIterator, PointerIterator]
				|
				iterate()					[GenericIterator, PointerIterator]
    			|
    			performBlockRefinement()	[GenericIterator, UsagePairIterator]
    			|
    			iterate()					[GenericIterator, UsagePairIterator]
    			|
    			performBlockRefinement()	[GenericIterator, PathPairIterator]
    			|
    			iterator()					[GenericIterator, PathPairIterator]
    			|
    			performBlockRefinement()	[GenericSinglePathRefiner, PredicateRefinerAdapter]
    								|
    								refinePath()	[GenericSinglePathRefiner, PredicateRefinerAdapter]
    										|
    			_________________________ call()	[GenericSinglePathRefiner, PredicateRefinerAdapter]
    			|
    			refiner.performRefinementForPath()	[PredicateRefinerAdapter, PredicateRefinerAdapte]
    			|
    			performRefinementForPath()			[PredicateCPARefiner, PredicateCPARefiner]
/**
 * 细化不一定会执行到最后一步,可能在前面就返回了，例如不一定会进入到GenericSinglePathRefiner中
 * 真正的细化器似乎只有PredicateCPARefiner
 */
// 在GenericSinglePathRefiner的performBlockRefinement()中：
      ExtendedARGPath firstPath = pInput.getFirst();
      ExtendedARGPath secondPath = pInput.getSecond();

      //Refine paths separately
      RefinementResult result = refinePath(firstPath);
      if (result.isFalse()) {
        return result;
      }
      Iterable<AdjustablePrecision> completePrecisions = result.getPrecisions();
      result = refinePath(secondPath);
      if (!result.isFalse()) {	// 如果两条路径都可行，则发现了真反例
        completePrecisions = Iterables.concat(completePrecisions, result.getPrecisions());
        result = wrappedRefiner.performBlockRefinement(pInput);		// 此时的wrappedRefiner为RefinementPairStub，这一步只做简单的处理，将结果表示成发现了真反例
      }

      // Do not need add any precision if the result is confirmed
      result.addPrecisions(completePrecisions);
      return result;
----> RefinementPairStub.performBlockRefinement()
  public RefinementResult performBlockRefinement(Pair<ExtendedARGPath, ExtendedARGPath> pInput) throws CPAException, InterruptedException {
    return RefinementResult.createTrue(pInput.getFirst(), pInput.getSecond());
  }    

不同Iterator中的iterate
PointerIterator ---> Pair<UsageInfoSet, UsageInfoSet>
UsagePairIterator ---> Pair<UsageInfo, UsageInfo>
PathPairIterator ---> Pair<ExtendedARGPath, ExtendedARGPath>
````



## 3.细化的停止条件

什么样的情况下退出细化过程，细化的次数由什么决定？

每次细化时，会对reached的container中的UnrefinedUnsafe中的所有UnrefinedIds分别进行细化处理

------> newPrecisionFound变量，记录对每一个UnrefinedId的细化过程中是否发现了新的谓词

------> 如果某个UnrefinedId的细化结果为true（表示是真反例），则没有新的谓词产生，同时将其移入RefinedIds中，

------> 如果某个UnrefinedId的细化结果为false（表示是假反例），则产生新的谓词，newPrecisionFound设置为true，会进行下一次的细化过程

只要有新的谓词产生，就会进行下一轮细化

-------> 新的谓词产生之后，需要进行新一轮的迭代，直到最终不再产生虚假反例

```java
// if there is any target state do refinement
if (refinementNecessary(reached, previousLastState)) {
	refinementSuccessful = refine(reached);
	refinedInPreviousIteration = true;
	// Note, with special options reached set still contains violated properties
	// i.e (stopAfterError = true) or race conditions analysis
}
```

`refinementNecessary(reached, previousLastState)`的结果决定了是否需要进行细化，实际上也是由`stableUnsfae`是否为空决定的

`stableUnsafes`包括`refinedIds`和`unrefinedIds`



## 4.unrefinedIds

一个`unrefinedIds`条目包括`id`和`UnrefinedUsageInfoSet`，`UnrefinedUsageInfoSet`包含`topUsages`和`usageInfoSets`

`usageInfoSets`的一个条目包括`UsagePoint`和`UsageInfoSet`(usageInfo的集合)



## 5.针对threadState和LockState的相容性判断

==判断两个usagePoint是否会构成Unsafe时，需要先判断两个usagePoint是否相容==

**相容则表明可能会存在Unsafe**（相容性判断相当于提前去掉那些不可能包含Unsafe的情况，且相容意味着可能存在竞争，这在投影方法中也是适用的）

不满足拓扑排序的情况在相容性判断中会被排除

**threadState**:

```java
     * 状态是否能够相容，取决于两个状态的threadSet的threadStatus关系（前提是这两个threadState包含相同的进程名）：
     * case1：只要有一个threadState的threadStatus为SELF_PARALLEL_THREAD，则两个threadState相容
     * case2：两个threadState的threadStatus不能同时为PARENT_THERAD，否则说明为PARENT_THREAD的线程都是当前线程所创建的，即两个threadState来自同一个线程，不会构成竞争
     * case3：如果两个threadState的threadStatus都为CREATED_THREAD，则对比其他的threadStatus
     * 相容对应的是可能形成竞争的情况
```

**lockState**:

```java
* lockState的相容取决于两个LockState的locks交集是否为空
* 交集为空则相容
* 相容对应的是可能形成竞争的情况
```
