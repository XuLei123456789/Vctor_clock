# 浅析时钟向量算法
## 算法的目的

在使用分布式数据库的时候，不同节点中数据的一致性一向是一个经典且难以解决的问题，而这个问题的根源是难以实现一个全局统一的时钟。下面就描述了这种问题的一种情况：
![picture1](https://github.com/XuLei123456789/Vctor_clock/blob/master/picture1.png)

如上图所示：A，B，C表示分布式系统中的三个数据库，纵轴表示时间。在T<sub>A1</sub>时刻A做出了更改`Key=Value1`，这次更改在T<sub>C2</sub>时刻传输到了C；在T<sub>B1</sub>时刻B做出了更改`Key=Value2`，这次更改在T<sub>C1</sub>时刻传输到了C。那么问题来了：C数据库中的`Key`应该等于`Value1`还是`Value2`呢？

自然地，我们可以想到：`Key`的取值应该和最新的更改保持一致。但是，由于很难实现一个所有节点都一致的全局时钟，所以**不同节点各自的时钟实际上并不具有可比性**，即T<sub>A1</sub> < T<sub>B1</sub>也不能说明B对`Key`的更改是在A之后发生的。

接下来要介绍的`向量时钟算法`就能够**部分**解决上面的问题。

## 算法的内容
在一个有N个节点的分布式数据库中，用一个N维的向量来表征时间，其中的某一个维表示一个节点的时间，这个时间向量按照以下规则进行处理：

- 所有节点的初始时间向量都是0
- 每一次经历一个时间间隔，都要在各自的时间维度上加1
- 每次发送数据，都要将这个向量时间作为时间戳和数据一起发出去
- 每次节点收到了时间向量，都要比较该时间向量和自身时间向量，并取两者中每一维中的最大值，作为自身新的时间向量
- 当收到有冲突的更改时，比较这两次更改的时间向量：若存在偏序关系，则取偏序关系中时间向量较大的对应的值，并以此作为本节点新的时间向量；若不存在偏序关系，则不能合并

其中的偏序关系是指：`若A向量中的每一维都大于等于B向量，那么就说A,B向量之间存在偏序关系，否则不存在偏序关系` 。

举个例子：
A，B，C三个节点的初始时间向量都是`(0,0,0)`，该向量的一，二，三维分别对应A，B，C三个节点各自的时间。
![picture2](https://github.com/XuLei123456789/Vctor_clock/blob/master/picture2.png)

- A作出更改`Key=Value1`，时间向量变为`(1,0,0)`
- A的更改传输到了B处，B处`Key=Value1`， 且时间向量变为`(1,0,0)`
- B作出更改`Key=Value2`， 将时间向量中自己对应的那一维加1变为`(1,1,0)`
- B和A的更改都同步到了C处。C比较两者的时间向量`(1,0,0)`和`(1,1,0)`，发现存在偏序关系，于是C的时间向量更新为`(1,1,0)`且`Key=Value2`

## 算法的本质
虽然我们已经学会了怎么使用时钟向量算法，但是似乎还和算法的本质隔着一层雾：我们其实并没有解决“不同节点之间统一的时钟”这一个客观的问题，但是通过向量时钟算法，我们却可以确定一些更改的先后顺序，而这些是在之前无法确定。那多出来的信息是从哪里获得的？

结合前面给出的两张图中的两个例子，对比传统的方法和时间向量算法的差异：

其实客观上是可以得知B对Key的更改是在A之后的，因为B是在收到A的更改之后才进行的下一步更改。传统的方法丢失了这部分信息，而向量时钟算法将这个信息保存了起来，用于后面对更改的先后顺序的判定。用数学表达就是：
因为T<sub>B0</sub> > T<sub>A1</sub>, T<sub>B1</sub> > T<sub>B0</sub>，所以T<sub>B1</sub> > T<sub>A1</sub>；而传统的方法丢失了“T<sub>B1</sub> > T<sub>B0</sub>”这条信息

分析完之后发现：

既然向量时钟的唯一目的是传输“T<sub>B1</sub> > T<sub>B0</sub>”这条信息，那么其他任何方法，只要能够将这类信息包含进去，也拥有和向量时钟算法相同的效果。比如`Git`中冲突合并的思想和向量时钟算法的本质其实是一样的：`Git`中不同的本地仓库拥有不同的时间维度，每一次`commit`对应一个时间维度上值的增加，可以快速合并的仓库对应的时间向量是具有偏序关系的。


总结以下：

- `给不同的节点设置不同的时间维度`体现了`不同节点之间没有一个统一的时钟，因此不同节点之间的时间就不具有可比性`
- `每个节点的更新会在自己的时间维度上加1`体现了`同一个节点上的时间是可以比较的`
- `时钟向量A，B具有偏序关系且A>B`体现了`A是在在同一节点上，在B的基础上增加的得到的；同一节点上发生的事件可以判断先后顺序，那么可以得知A是在B之后的`
- `时钟向量A，B不具有偏序关系`体现了`A，B之间不存在能够将某一个向量的尾端顺着同一个节点连到另一个向量中间的事件`

所以向量时钟算法的实质是：

- 将逻辑上可以合并的冲突成功合并
- 逻辑上无法合并的冲突依旧冲突



## 拓展：和狭义相对论时空观念的比较
上面算法中的这种`不同节点中的时间不具有不具有可比性`和`用一个多维的时间来代替一维的时间`的时空观，和狭义相对论中的时空观非常类似。

在狭义相对论中，同一事件从不同的参考系中观察是不一样的，而两次事件可以建立因果关系的前提是：两个事件之间可以用等于或小于光速的速度传递信息（或者说一事件位于另一事件的光锥内部）。

| 在狭义相对论中的描述 | 在向量时钟算法中的描述 |
| ------------- |:-------------:|
| 不同的惯性参考系有不同的时间 | 不同的节点有不同的时钟向量维度 |
| 两件事件具有因果关系（事件2位于事件1的光锥内部）| 两次更改可确定先后顺序（两个向量具有偏序，类似光锥内部） |
|两件事件不具有因果关系（事件2位于事件1的光锥外部） | 两件事件不具有因果关系（事件2位于事件1的光锥外部）|
更详细的关于时钟向量算法和狭义相对论时空观的比较可以阅读参考文献3
区别：

狭义相对论可以用洛伦兹变换将不同的参考系中的时间进行变换（利用到不同参考系之间的相对速度V），但是向量时钟算法不可以（其实逻辑上也可以，但是实现后没有现实意义），所以只能用多维的时间来表征不同节点中的时间却没法相互转化。

## 查考文献
-  Colin J. Fidge (February 1988). ["Timestamps in Message-Passing Systems That Preserve the Partial Ordering"](http://zoo.cs.yale.edu/classes/cs426/2012/lab/bib/fidge88timestamps.pdf). In K. Raymond (Ed.). Proc. of the 11th Australian Computer Science Conference (ACSC'88). pp. 56–66. Retrieved 2009-02-13.
- Almeida, Paulo; Baquero, Carlos; Fonte, Victor (2008), ["Interval Tree Clocks: A Logical Clock for Dynamic Systems", in Baker, Theodore P.; Bui, Alain; Tixeuil, Sébastien, Principles of Distributed Systems](http://gsd.di.uminho.pt/members/cbm/ps/itc2008.pdf), Lecture Notes in Computer Science, 5401, Springer-Verlag, Lecture Notes in Computer Science, pp. 259–274, doi:10.1007/978-3-540-92221-6, ISBN 978-3-540-92220-9
- [时钟向量算法和狭义相对论时空观的比较](https://www.zhihu.com/question/30084741/answer/71115362)
