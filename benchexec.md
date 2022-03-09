#### benchexec

网址：https://github.com/sosy-lab/benchexec/blob/main/doc/INDEX.md

```xml
<?xml version="1.0"?>
<!DOCTYPE benchmark PUBLIC "+//IDN sosy-lab.org//DTD BenchExec benchmark 1.0//EN" "http://www.sosy-lab.org/benchexec/benchmark-1.0.dtd">
<benchmark tool="cpachecker" timelimit="900" hardtimelimit="1000" memlimit="8000 MB" cpuCores="2">
  <!-- This file contains regression tests from the LDV project.
       It expects the git repository git@bitbucket.org:dbeyer/ldv-benchmarks.git
       to be checked out at test/programs/ldv-benchmarks. -->

  <option name="-setprop">statistics.memory=true</option>
  <option name="-heap">10000M</option>
  <option name="-setprop">counterexample.export.exportWitness=false</option>
  <option name="-setprop">cpa.thread.skipTheSameThread=true</option>
  <option name="-setprop">cpa.arg.export=false</option>
  <option name="-setprop">counterexample.export.exportAsSource=false</option>
  <option name="-setprop">cpa.lock.refinement=false</option>

  <propertyfile>./data-race.prp</propertyfile>
  
  <tasks name="DeviceDrivers64">
    <includesfile>Tasks.set</includesfile>
  </tasks>

  <rundefinition name="base">
    <!-- option包含了传递给cpachecker的参数 -->
    <option name="-lockator-linux-without-shared"/> <!-- 属性文件名 -->
    <option name="-setprop">cpa.predicate.refinement.predicateBasisStrategy=subgraph</option>
    <option name="-setprop">cpa=cpa.arg.ARGCPA</option>
    <option name="-setprop">CompositeCPA.cpas=cpa.location.LocationCPA,cpa.callstack.CallstackCPA,cpa.thread.ThreadCPA,cpa.lock.LockCPA,cpa.functionpointer.FunctionPointerCPA</option>
    <option name="-setprop">cpa.usage.refinement.refinementChain=IdentifierIterator,PointIterator,UsageIterator,SinglePathIterator</option>
  </rundefinition>

</benchmark>
```



#### combat中的配置情况

```bash
# 1.base
输入的属性文件：
- config/lockator-linux-without-shraed.properties
     |
     ---- includes/lockator/lockator-core.properties
     ---- includes/lockator/lockStatistics-linux.properties
     		  |
     		  ---- lockStatistics-predicate.properties
     		  	   指定CPA等信息
     		  ---- linux.properties
属性设置：
cpa.predicate.refinement.predicateBasisStrategy=subgraph
cpa=cpa.arg.ARGCPA
CompositeCPA.cpas=cpa.location.LocationCPA,cpa.callstack.CallstackCPA,cpa.thread.ThreadCPA,cpa.lock.LockCPA,cpa.functionpointer.FunctionPointerCPA
cpa.usage.refinement.refinementChain=IdentifierIterator,PointIterator,UsageIterator,SinglePathIterator

# base中，获取后继时状态时直接调用每个封装状态的transferRelation函数
# 调用transferRelation之后，还有一个strengthen的过程
```

> 使用base的配置，容易出现误报

```bash
# 2.with-bam
输入的属性文件与base相同
配置唯一的不同在于没有指定
cpa=cpa.arg.ARGCPA
```



