## 2021.11.18

* #### lockator中是如何对环境（Projectin）中进行近似的

  * PredicateApplyOperator.java中

    project产生投影的结果，组成投影的两部分分别进行prepare：prepareEdge（edge），prepareFormula（guard）

    * prepareEdge

      1.首先只考虑函数返回边和C表达式边

      2.然后进一步只考虑表达式为赋值表达式的边

      3.如果表达式不涉及全局变量，则prepareEdge的结果为空边

      4.获取left和right，为right生成未定义函数fExp

      ```java
      right = 0;
      fExp = prepareUndefFunctionFor(right);
      //fExp可能是一个生成不确定int的函数，returnType = int，name = int__VERIFIER__nondet__signed__int()
      ```

      5.使用fExp进行新的赋值，生成新的赋值表达式

      6.使用新的赋值表达式生成新的边，fakeEdge

      7.谓词抽象中，还要进一步生成路径公式，如果pFormula为true，则prepareEdge的结果为空边

    * prepareFormula

      prepareFormula的结果是公式，例如：`guard： (= 0 (|*(struct_net_device)*:global| |__ADDRESS_OF_ipddp_init::dev__ENV@|))`
      
      不过生成PredicateProjectedState时，guard通常会变为true

在产生的投影中，其封装状态中，edge和guard只在PredicateProjectedState中有：

```java
//例如投影
ARG State (Id: 3343, Parents: [], Children: [], Covering: []) []
(ProjectedLocationStateWithEdge: N6573
 ProjectedCallstackState: Function null called from node null, stack depth 1 [25f0c5e7], stack [null]
 ThreadState: ldv_thread_5:{ldv_thread_5=CREATED_THREAD}
 LockState: Without locks
 PredicateProjectedState: org.sosy_lab.cpachecker.cpa.predicate.PredicateProjectedState@5882b202
)
//封装状态PredicateProjectedState中
edge:[cf_arg_3__ENV->arg0 = __VERIFIER_nondet_struct_pci_driver_p_();]
guard:"true"
```

所以，guard和edge都采用了**上近似**

> 禁用predicateCPA
>
> 在race版本的lockator中，如果禁用了predicateCAP，则原本会产生投影的任务不会产生投影，而且验证结果也不正确



> 2021.11.15

| 配置                                                         | 结果                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `-config config/lockator-linux-without-shared.properties -setprop cpa.predicate.refinement.predicateBasisStrategy=subgraph unsafe2.i` | 这里根据ANT.xml文件中的both配置得到<br />能够得出验证结果FALSE，<br />不过状态中并没有自动机状态 |
| `-config config/lockator-linux-without-shared.properties -setprop cpa.predicate.refinement.predicateBasisStrategy=subgraph -spec ldv-ben/data-race.prp unsafe2.i` | 能够得出验证结果FALSE，<br />不过状态中并没有自动机状态      |
| `-config config/MyConfigure/both/lockator-linux-without-shared.properties -setprop cpa.predicate.refinement.predicateBasisStrategy=subgraph -spec ldv-ben/data-race.prp -setprop configuration.dumpFile=MyConfigure.properties -setprop report.exprot=true unsafe2.i` | OUT OF MEMORY                                                |
|                                                              |                                                              |
|                                                              |                                                              |

#### .spc文件

```bash
# 以sv-comp-reachbility.spc为例 
CONTROL AUTOMATON SVCOMP

INITIAL STATE Init;

STATE USEFIRST Init :
  MATCH {__VERIFIER_error($?)} || MATCH {reach_error($?)} || MATCH FUNCTIONCALL "reach_error"
      -> ERROR("unreach-call: $rawstatement called in $location");
  MATCH {__assert_fail($?)} || MATCH {abort($?)} || MATCH {exit($?)} -> STOP;

END AUTOMATON

# 结合benchmark中的.i文件（当中定义了__VERIFIER_error()）函数，应该是检查对__VERIFIER_error()函数的调用
# 之前加了data-race.prp文件之后，也能运行成功，但是程序状态中没有自动机状态，推测data-race.prp文件可能没有作用（对于验证而言）

##### 在combat版本中
# 如果指定assertion.spc（当中定义了__VERIFIER_error()函数），不过这里有点问题就是Assertion.spc被定义为失效，理论上应该不能生效，存疑
# 则可以得出验证结果false
./scripts/cpa.sh -config  config/lockator-linux-without-shared.properties -spec config/specification/Assertion.spc  unsafe.i 

# 如果不指定assertion.spc，则也能得出验证结果
# 此时应该是使用了default.spc，其中包含了Assertion.spc（不过这里有点问题就是Assertion.spc被定义为失效，理论上应该不能生效，存疑）
./scripts/cpa.sh -config  config/lockator-linux-without-shared.properties unsafe.i                                           

# 如果明显指定其他的.spc文件，如Errorlabel.spc
# 则得出的结果为UNKNOWN
$ ./scripts/cpa.sh -config  config/lockator-linux-without-shared.properties -spec config/specification/ErrorLabel.spc  unsafe.i

# 使用race_02_verifier_error.i进行测试
$ ./scripts/cpa.sh -config  config/lockator-linux-without-shared.properties  -spec config/specification/Assertion.spc  race_02_verifier_error.i 
# 输出内容中包含以下内容：
WARNING: Function __VERIFIER_error() is ignored by this specification. If you want to check for reachability of __VERIFIER_error, pass '-spec sv-comp-reachability' as parameter. (AssertionAutomaton:AutomatonTransition.executeActions, INFO)
# 说明Assertion.spc中的__VERIFIER_error没有生效
# 验证结果为true（不正确）

# 同样使用race_02_verifier_error.i进行测试
$ ./scripts/cpa.sh -config  config/lockator-linux-without-shared.properties  -spec config/specification/sv-comp-reachability.spc race_02_verifier_error.i
# 结果显示为UNKNOW
# 可能是__VERIFIER_error的用法还不明确，同时也可能是利用其他标签来判错
```



## 2021.11.26

```bash
# 对于race_02_verifier_error.i，在__VERIFIER_error()中明确指定reach_error()函数时，可以得到正确的结果，即验证结果为False
# 其他情况验证结果均为true

# ？：使用svcomp-21的配置文件对unsafe.i和unsafe2.i进行验证，设置spec文件为svcomp-reachability.spc，结果均为true
##### 验证safe.i时，使用default.spc会报错，unsafe2.i使用default.spc不会报错，但是得出的结果不正确
```

benchmark测试，data-race.prp对于结果无影响，似乎只影响结果的判定，即没有data-race.prp也能得出正确的验证结果。将benchmark中的配置复制，对unsafe.i的验证能够得出正确结果
