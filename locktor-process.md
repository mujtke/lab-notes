## 流程梳理

```java
/* 可达集的计算 */
1.CPAchecker中runAlgorithm(algorithm, reached, stats)
2.方法runAlgorithm()中，algorithm.run(reacheded)
3.CEGARAlgorithm中，algorithm.run(reached) 
4.CEGARAlgorithm中，algorithm.run中的run方法调用run0方法对可达集进行计算更新

/* 细化 */
1.CEGARAlgorithm中，refinementNecessary(reached, previousLastState)判断是否需要细化，即是否发现了潜在的错误条件
2.refinementSuccessful = refine(reached)，进行细化
    
```

