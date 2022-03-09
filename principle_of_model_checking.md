> #### ch1. model checking

validation:检验“模型+性质”对于描述实际的问题是否足够，是否充分满足了我们对系统的预期要求

model checking的优缺点：



> #### ch2. modeling concurrent systems

使用迁移系统建模的关键因素之一：**不确定性**

```bash
# 模型和自动机如何求交？
系统 ---> model A (TS ?)      ----
								 |
性质 ---> LTL B (automaton)   ----|
								 |
						  ?  <---|
```

