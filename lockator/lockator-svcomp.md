## 1.配置&验证内容差异

svcomp21-lockator中的CPA配置与lockator-linux-withou-shared_single中的配置不相同

```java
/* svcomp21 */
cpa = cpa.threadmodular.ThreadModularCPA
ThreadModularCPA.cpa =  cpa.arg.ARGCPA
ARGCPA.cpa = cpa.composite.CompositeCPA
CompositeCPA.cpas = cpa.location.LocationCPA, cpa.callstack.CallstackCPA, cpa.thread.ThreadCPA, cpa.lock.LockCPA, cpa.predicate.PredicateCPA

/* lockator-linux-without-shared */
cpa = cpa.bam.BAMCPA
BAMCPA.cpa = cpa.arg.ARGCPA
ARGCPA.cpa = cpa.usage.UsageCPA
UsageCPA.cpa = cpa.composite.CompositeCPA
CompositeCPA.cpas = cpa.location.LocationCPA, cpa.callstack.CallstackCPA, cpa.thread.ThreadCPA, cpa.lock.LockCPA, cpa.predicate.BAMPredicateCPA, cpa.functionpointer.FunctionPointerCPA
```

svcomp的配置中没有设置Usage，应该不能通过Pair的方式来决定是否发现了错误，应该是使用相应的标签来检查是否发现了target状态

<font color=red>直接用svcomp的配置，跑不了ldv-benchmark中的内容，svcomp是可达任务，和race检测不一样</font>

<font color=red>只能跑sv-benchmark中的内容</font>，例如sv-benchmark中的[ldv-linux-3.14](https://gitlab.com/sosy-lab/benchmarking/sv-benchmarks/-/tree/main/c/ldv-linux-3.14)（P.A.在其2021的文章中提到）

race检测使用的benchmark则是ldv-benchmark中的[linux-4.2.6-races](https://gitlab.com/sosy-lab/software/ldv-benchmarks/-/tree/main/linux-4.2.6-races)

```java
// ldv-linux-3.14中的待检测任务需要经过一定的处理，例如：
Insert assert(0) into the body of reach_error
	As discussed in the (virtual) community meeting on 2020-09-18,
	unreach-call may be turned into plain assertion checking. As first step,
	make the body of reach_error() `assert(0);`. For preprocessed files, the
	result of assert(0) expansion is what we get on Ubuntu 18.04 with glibc
	and GCC 7.5.0.
// 应该是将原本非reach的任务变成可达任务
// 所以使用svcomp的配置能够顺利检测
// 而Linux-4.2.6-races中的任务是没有这样的处理的
```





