## 2021.11.23

#### 模型

系统的建模使用迁移系统（transition system），对于一个特定的迁移系统$M = \left\langle S,S_0,\rightarrow, L  \right\rangle$，使用状态的集合S作为字符表$\Sigma$，M中每一条无限路径$\pi$是集合$\Sigma^{\omega}$中的一个文字，M所有路径的集合便是被迁移系统M所接受的语言$L(M)$.

#### LTL公式的语言

$\phi$是一个LTL公式，S是与$\phi$有着相同原子命题集合的模型的状态的集合，$\phi$的语言$L(\phi)$定义为:
$$
L(\phi) = \{\pi \in S^{\omega} | \pi \models^0  \phi\}
$$
使用原子命题集合的幂集作为字符表而不是使用状态的集合S也是可以的

#### 模型检测

模型检测的问题在于回答$ M\models \phi$是否成立，等价地，$\forall \pi \in Paths(M). \pi \models^0 \phi$是否成立.

使用语言来表示，即为$L(M) \subseteq L(\phi)$，或者等价地，$L(M) \cap \overline{L(\phi)} = \emptyset$.

> $L(M)$使用了状态迁移系统来定义，但$L(\phi)$不能使用迁移系统来表示，而是使用$Buchi$自动机来表示：
>
> S：状态集合
>
> $\Sigma$：字符表
>
> $\rightarrow \subseteq S \times \Sigma \times S$：迁移关系
>
> $S_0 \subseteq S$：初始状态集合
>
> $A \subseteq S$：接受状态
>
> 一个文字被$Buchi$自动机接受当且仅当该自动机的接受状态被无限多次访问

实际中，模型检测的做法是：

* $\overline{L(\phi)} = L(\neg \phi)$
* 假设公式$\phi$对应$Buchi$自动机为$A_{\phi}$，使得$L(\phi) = L(A_\phi)$
* 让$M \otimes A$表示迁移系统M和自动机A的组合，则我们有：$L(M \otimes A) = L(M) \cap L(A)$
* 因此，检测$M \models \phi$就变成了检测$ L(M \otimes A_{\neg \phi}) = \emptyset$



#### 一个简单的例子

对LTL公式$\phi =G \ p$进行模型检测

首先为$\neg \phi = \neg G\ p = F\ \neg p$构建自动机$A_{F\ \neg p}$：

![3.png](https://i.loli.net/2021/11/23/mYTfeRKBl3ksc8N.png)

模型M为：

![0.png](https://i.loli.net/2021/11/23/2jY7kUzITHQxEdN.png)

当在模型M中定义$p\ :=\ st = 0 | st = 1$时，对应的模型M为：

![1.png](https://i.loli.net/2021/11/23/VRPBXmFusNeSAad.png)

对于该模型的一条路径：$0222222...$，对应于自动机$A_{F\ \neg p}$中的路径$01111111...$，该路径是自动机$A_{F\ \neg p}$的一个接收文字，因此当$p\ :=\ st = 0 | st = 1$时，$L(M \otimes A_{F\ \neg p}) \neq \emptyset$，即$M \models \phi$不成立。直观上，路径$02222...$中$G\ p$是不成立的。

当在模型M中定义$p\ :=\ TRUE$时，对应的模型M为：

![2.png](https://i.loli.net/2021/11/23/bRhWV3LyM1BKCYw.png)

此时M中的任何路径，在自动机$A_{F\ \neg p}$中对应的路径都为$011111...$，该路径不是自动机$A_{F\ \neg p}$的接受文字，因此$L(M \otimes A_{F\ \neg p}) \neq \emptyset$，$M \models \phi$成立。直观上，M中的所有路径，$G\ p$都是成立的。





