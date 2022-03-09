## 1. specification

规范自动机的相关语法：doc/specificationAutomata.md

自动机状态与CFA和自定义的规范自动机有关，或者说抽象状态是两者结合得到的（通过transferRelation和程序控制流来联合得到？）



```c
# 数据竞争的范例测试文件：
#include <threads.h>
#include <stdlib.h>
#include <assert.h>

extern void abort(void);

void reach_error() { assert(0); }


void ldv_assert(int expression) { if (!expression) { ERROR: {reach_error();abort();}}; return; }


int a = 0;

int Function( void* ignore )
{
    a = 1;

    return 0;
}

int main( void )
{
    thrd_t id;
    thrd_create( &id , Function , NULL );

    ldv_assert(a == 1);
    int b = a;

    thrd_join( id , NULL );
}
```

在测试时使用预处理选项-preprocess



## 2.properties File

```bash
# 有关输出的properties设置
report.export = true
report.file = ****.html  -> 可以任意指定

```





### 3.3 RefinementBlockFactory







