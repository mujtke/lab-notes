**<font color=red>注：</font>梳理时使用的配置文件为：[lockator-Linux-without-shared_single.properties](/home/mujueke/Documents/javaCode/cpachecker-CPALockator-combat-mode/config/MyConfigure/both)**

***

```java
********************************************************
    				可达集的计算
********************************************************
runAlgorithm()		[CPAchecker]
    	|
    	algorithm.run()		[CEGARAlgorithm]
    			|
    			algorithm.run()		[CPAAlgorithm]
    					|
    					run0()		[CPAAlgorithm]
    					|
    					handleState()	[CPAAlgorithm]
    					|
    					transferRelation.getAbstractSuccessors		[CPAAlgorithm]
    					|
    					getAbstractSuccessors()						[TransferRelation, this == BAMTransferRelation]
    					|
    					super.getAbstractSuccessors()				[BAMTransferRelation]
    					|	
    					getAbstractSuccessorsWithoutWrapping()		[AbstractBAMTransferRelation, this == BAMTransferRelation]
    					|
    					getWrappedTransferSuccessor()				[AbstractBAMTransferRelation, this == BAMTransferRelation]
    					|
    					transferRelation.getAbstractSuccessors()	[BAMTransferRelation]
    					|
    					transferRelation.getAbstractSuccessors()	[ARGTransferRelation]
    					|
    					getAbstractSuccessorsForEdge()				[UsageTransferRelation]
    					|
    					transferRelation.getAbstractSuccessorsForEdge()		[UsageTransferRelation]
    					|
    					getAbstractSuccessorForSimpleEdge()			[CompositeTransferRelation]
    					|
    					callTransferRelation()						[CompositeTransferRelation]
    					|
    					调用compositeCPA中的每个CPA的transferRelation来获取相应的后继
    					// 获取后继时，真正计算后继的只是compositeCPA封装的那些CPA，compositeCPA之上的CPA只是调用了相应的transferRelation
```

<img src="https://i.loli.net/2021/11/29/NpoLrJqHB3mQ56A.png" alt="CPA的封装关系"  />

