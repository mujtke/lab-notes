## 1.Usage

2014年的文献中，UsageCPA是Lock Analysis的配置

![LockAnalysis.png](https://i.loli.net/2021/11/29/NpoLrJqHB3mQ56A.png)	

UsageCPA收集数据使用的信息，TransferRelation区分表达式中用于读写操作的变量并为变量的使用保留调用栈以及获取到的锁的集合。

变量的使用（Usage）包括：

* 获取的锁集合
* 函数调用栈
* 行号
* CFG边的类型（函数调用表达式等）
* 变量的访问类型（读还是写）

UsageCPA也用于建立变量之间的等价关系，以便于可以在分析时可以被视作相同的数据。（这里是为了区分例如不同列表中相同的字段实际上代表了不同数据的情况，减少假阳性）

```java'
//感觉用结构体的描述更易理解
//例如struct E A,B都有字段a
则通过A->a,B->a来区分不同字段的a
//否则只通过变量名a来区分，就会引起误报
```



## 2. getUsageForEdge

**注**：==extratUsages是很费时间的步骤==

==**cpa.usage**==

==实际上获取到的usage格式为：<edge, ids>，edge对应具体的边，而ids则对应该边所引起的访问==

```java
Collection<Pair<AbstractIdentifier, Access>> ids;

ids = getUsagesForEdge()
				|
    		switch (边的类型)
						|______ case: DeclarationEdge  ---> 无论是全局变量还是局部变量都不会引起竞争，因此不会生成Usage
                		|______ case: StatementEdge    ---> 会生成Usage，如果为left则对应WRITE访问，如果为right则为READ访问
						|______ case: AssumeEdge	   ---> 
						|______ case: FunctionCallEdge ---> 函数名id会被认为无关，因此不会生成Usage                           
```

==对不同的Edge，`visitStatement`对相应的statement进行处理==，`visitStatement`是获取statement中ids的核心

`handleStatement()`中，分别对三种类型的statement的进行处理：

* `CAssignment`
* `CFunctionCallStatement`
* `CExpressionStatement`

如果是`CAssignment`：

```java
// 分别提取left和right
CExpression left = assignment.getLeftHandSide();
CRightHandSide right = assignment.getRightHandSide();

// 对left和right，均使用visitStatement方法进行<Id, Access>的抽取
// 使用访问者模式，visitStatement中，构造一个visitor，然后将其传给Expression的accept方法
ExpressionHandler handler = new ExpressionHandler(access, fName, varSkipper, localCreator);  // 该类继承自某个visitor，因此是访问者模式中的visitor
expression.accept(handler);		// 根据expression的类型调用相应的accept方法
				|
               ...	[不同类型的expression对应的accept方法]
                |
              visit()	[ExpressonHandler，Usage中]	// 似乎最后都会调用ExpressionHandler中的visit
```

`ExpressionHandler`中有许多`visit`的重载版本（==用于处理不同的表达式==），以处理`CIdExpression`为例：

```java
  public Void visit(CIdExpression expression) {
    addExpression(expression);
    return null;
  }

  private void addExpression(CExpression e) {
    AbstractIdentifier id = idCreator.createIdentifier(e, 0);
    if (isRelevantForAnalysis(id)) {	// 通过相关性分析排除局部变量，函数名等不会引起竞争的内容
      result.add(Pair.of(id, accessMode));
    }
  }

// 相关性分析可以筛选出对全局变量的操作
  private boolean isRelevantForAnalysis(AbstractIdentifier pId) {
    if (varSkipper.shouldBeSkipped(pId, fName)) {     // 首先判断是否可以跳过，如果不属于提前标记的需要跳过的，则是分析相关的
      return false;
    }

    if (pId instanceof LocalVariableIdentifier && pId.getDereference() <= 0) {      // 判断是否是局部变量
      // we don't save in statistics ordinary local variables
      return false;
    }
    if (pId instanceof StructureIdentifier && !pId.isGlobal() && !pId.isDereferenced()) {     // 判断是否是非全局的结构变量
      // skips such cases, as 'a.b'
      return false;
    }

    if (pId instanceof FunctionIdentifier) {      // 判断是否是函数名Id
      return false;
    }

    return true;
  }

shouldBeSkipped()
    	|
    checkId()	[根据不同的情况来过滤Id，这些不同的情况在配置文件中进行了说明, 如果Id符合相应的要求，则会被忽略]
    	|________ byName			[null]
    	|________ byNamePrefix		["ldv", "emg_", "__ldv"]
    	|________ byType			["struct ath_tx_stats"]
    	|________ byFunction		["entry_point", "INIT_LIST_HEAD", "__list_del", "__list_add", "list_add"]
    	|________ byFunctionPrefix	["ldv_initialize"]
```

对应的[配置文件](/home/mujueke/Documents/javaCode/cpachecker-CPALockator-combat-mode/config/MyConfigure/both)设置：

```c
cpa.usage.skippedfunctions = mfree_annotated
cpa.usage.skippedvariables.byFunction = entry_point, INIT_LIST_HEAD, __list_del, __list_add, list_add
cpa.usage.skippedvariables.byFunctionPrefix = ldv_initialize
cpa.usage.skippedvariables.byNamePrefix = ldv_, emg_, __ldv
cpa.usage.skippedvariables.byType = struct ath_tx_stats
```

**例如，对语句“d=1”**：

left = “1”，对表达式“1”不会生成Usage

right = “d”，对表达式“d”生成interId：<d, WRITE>

最终生成的Usage为==<N12 -- d=1 --> N19, <d, WRITE>>==





## 3.获取后继

### 3.1 UsageTransferRelation

* UsageCPA获取边的后继：

```java
  @Override
  public Collection<? extends AbstractState> getAbstractSuccessorsForEdge(        //     ------->  应该是UsageState获取后继的核心
      AbstractState pState, Precision pPrecision, CFAEdge pCfaEdge)
      throws CPATransferException, InterruptedException {

    statistics.transferForEdgeTimer.start();

    UsageState oldState = (UsageState) pState;

    statistics.checkForSkipTimer.start();
    CFAEdge currentEdge = changeIfNeccessary(pCfaEdge); //如果pCfaEdge的调用了abort函数或者skipped函数，则返回空边或者summary边（相当于跳过函数分析）
    statistics.checkForSkipTimer.stop();

    if (currentEdge == null) {  //return <1> -------->如果调用了abort函数，则后继状态为空集
      // Abort function
      statistics.transferForEdgeTimer.stop();
      return ImmutableSet.of();
    }

    statistics.innerAnalysisTimer.start();
    //先根据composite的transferRelation来获取传入的WrappedStates的后继WrappedStates
    Collection<? extends AbstractState> newWrappedStates =
        transferRelation
            .getAbstractSuccessorsForEdge(oldState.getWrappedState(), pPrecision, currentEdge);   //这里的transferRelation是传入的封装的transferRelation，即composite的transferRelation
    statistics.innerAnalysisTimer.stop();

    statistics.bindingTimer.start();
    creator.setCurrentFunction(getCurrentFunction(oldState));   //获取当前的函数调用，oldState的函数调用可能在处理函数调用表达式之后发生了改变（跳过的函数只是分析上跳过，调用栈还是要正常计算？）
    // Function in creator could be changed after handleFunctionCallExpression call

    Collection<? extends AbstractState> result =
        handleEdge(currentEdge, newWrappedStates, oldState);      // handleEdge -----> 根据边的类型来进行不同的处理，重要
    statistics.bindingTimer.stop();

    if (currentEdge != pCfaEdge) {
      callstackTransfer.disableRecursiveContext();
    }
    statistics.transferForEdgeTimer.stop();
    return ImmutableList.copyOf(result);
  }
```

* `changeIfNeccessary()`：获取边的后继，如果后继状态是函数状态，根据函数的类型对边进行变更

```java
// case1：函数是abortFunction，则返回NULL
// case2：函数是skippedFunction，则返回summary边，相当于函数被跳过
```

* `handleEdge()`分情况对不同的边进行处理：

```java
private Collection<? extends AbstractState>
      handleEdge(
          CFAEdge pCfaEdge,
          Collection<? extends AbstractState> newWrappedStates,
          UsageState oldState)
          throws CPATransferException {

    Collection<AbstractState> result = new ArrayList<>();

    switch (pCfaEdge.getEdgeType()) {
      case StatementEdge:
      case FunctionCallEdge:
        {     //这里应该是对应没有skipped的函数调用

        CStatement stmt;
        if (pCfaEdge.getEdgeType() == CFAEdgeType.StatementEdge) {  //表达式边，应该是类似"a = foo()"之类的
          CStatementEdge statementEdge = (CStatementEdge) pCfaEdge;
          stmt = statementEdge.getStatement();
        } else if (pCfaEdge.getEdgeType() == CFAEdgeType.FunctionCallEdge) {   //函数调用边，应该类似"func()"之类的
          stmt = ((CFunctionCallEdge) pCfaEdge).getRawAST().get();
        } else {
          // Not sure what is it
          break;
        }

        Collection<Pair<AbstractIdentifier, AbstractIdentifier>> newLinks = ImmutableSet.of();

        if (stmt instanceof CFunctionCallAssignmentStatement) {
          // assignment like "a = b" or "a = foo()"
          CAssignment assignment = (CAssignment) stmt;
          CFunctionCallExpression right =
              ((CFunctionCallAssignmentStatement) stmt).getRightHandSide();   //用于获取函数名
          CExpression left = assignment.getLeftHandSide();      // LHS
          newLinks =
              handleFunctionCallExpression(
                  left,
                  right,
                  (pCfaEdge.getEdgeType() == CFAEdgeType.FunctionCallEdge && bindArgsFunctions));   //进行绑定分析
        } else if (stmt instanceof CFunctionCallStatement) {
          /*
           * Body of the function called in StatementEdge will not be analyzed and thus there is no
           * need binding its local variables with its arguments.
           * 如果只是函数调用而没有LSH，则不用进行绑定分析
           */
          newLinks =
              handleFunctionCallExpression(
                  null,
                  ((CFunctionCallStatement) stmt).getFunctionCallExpression(),
                  (pCfaEdge.getEdgeType() == CFAEdgeType.FunctionCallEdge && bindArgsFunctions));
        }

        // Do not know why, but replacing the loop into lambda greatly decreases the speed
        for (AbstractState newWrappedState : newWrappedStates) {  //这里的WrappedState是compositeState，WrappedStates是WrappedStates的数组
          UsageState newState = oldState.copy(newWrappedState);   //其实是求WrappedStates的后继状态（有多个）？

          if (!newLinks.isEmpty()) {
            newState = newState.put(newLinks);
          }

          result.add(newState);
        }
        return ImmutableList.copyOf(result);      //UsageState的内容：stats（数据相关）、variableBindingRelation、WrappedStates（compositeState）
      }

      case FunctionReturnEdge: {
        // Data race detection in recursive calls does not work because of this optimisation
        CFunctionReturnEdge returnEdge = (CFunctionReturnEdge) pCfaEdge;
        String functionName =
            returnEdge.getSummaryEdge()
                .getExpression()
                .getFunctionCallExpression()
                .getDeclaration()
                .getName();
        for (AbstractState newWrappedState : newWrappedStates) {
          UsageState newState = oldState.copy(newWrappedState);
          result.add(newState.removeInternalLinks(functionName));	//去除与局部变量相关的变量绑定
        }
        return ImmutableList.copyOf(result);
      }

      case AssumeEdge:
      case DeclarationEdge:
      case ReturnStatementEdge:
      case BlankEdge:
      case CallToReturnEdge:
        {
          break;
        }

      default:
        throw new UnrecognizedCFAEdgeException(pCfaEdge);
    }
    for (AbstractState newWrappedState : newWrappedStates) {
      result.add(oldState.copy(newWrappedState));
    }
    return ImmutableList.copyOf(result);
  }
```

变量关系绑定与论文中提到的等价关系似乎对应



## 4.expandUsageAndAdd

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



## 5.usageContainer

向container中添加usage（`<id, unrefinedUsageInfoSet>`）时，==是否将usage放到topUsages中需要判断覆盖性（和已有的topUsages之间进行判断）==。

```java
unrefinedUsagesInfo:
					topUsages
                    usageInfoSets
```

不过usage对应的UsageInfo一定会添加到UsageInfoSets中的==UsageInfoSet==中。

有关覆盖的判断细节：

将usage对应的usagePoint放到topUsages中时，

* 如果point与topUsages中的某个point相同，则不添加该point
* WRITE方式的point会覆盖READ方式的pint（前提是锁集合必须均为空）
* 否则一次判断threadState和LockState的覆盖情况
  * threadState：threadSet大的覆盖threadSet小的
  * LockState：locks小的覆盖Locks大的
