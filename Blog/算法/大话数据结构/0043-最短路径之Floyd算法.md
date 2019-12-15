## Flyod算法

Flyod是个非常牛逼的人，拥有一个开挂的人生。

人物简介在此：[罗伯特·弗洛伊德](https://baike.baidu.com/item/罗伯特·弗洛伊德/4903135?fr=aladdin) 。



### 算法介绍

Flyod算法是一个非常优美的算法，因为它就只有几行，如下：

```java
for(k=0;k<n;k++)
   for(i=0;i<n;i++)
       for(j=0;j<n;j++)
           A[i][j]=min(A[i][j],A[i][k]+A[k][j]);
```

关于这个算法，它还有一个笑话：

> 为什么 Dijkstra 不能提出 floyd 算法？因为他的名字是 ijk 而不是 kij。

要理解这个笑话，你需要认真读一下这个算法，在这个算法的3层循环中，三个变量是有一定顺序的：k 必须要在 i 与 j 的外面，i 与 j 没有顺序。



### 动态规划

我们下面来说一下这个算法的由来。Floyd 算法是基于**动态规划**而来的，不了解动态规划的可以先去补一下相关知识。

我们假设，![[公式]](https://www.zhihu.com/equation?tex=d%5Ek%28i%2Cj%29)为 i, j之间的最短路，它使用 ![[公式]](https://www.zhihu.com/equation?tex=%5C%7B1%2C2%2C%5Ccdots%2Ck%5C%7D)作为中间节点，那么根据动态规划方程，可以知道:

![[公式]](https://www.zhihu.com/equation?tex=d%5Ek%28i%2Cj%29+%3D+min%28d%5E%7Bk-1%7D%28i%2C+j%29%2C+d%5E%7Bk-1%7D%28i%2C+k%29+%2B+d%5E%7Bk+-+1%7D%28k%2C+j%29%29)。

本来，是有四种情况的，如下：

![[公式]](https://www.zhihu.com/equation?tex=d%5E%7Bk%7D%28i%2C+j%29+%3D+min%28d%5E%7Bk+-+1%7D%28i%2C+j%29%2C+d%5E%7Bk+-+1%7D%28i%2C+k%29+%2B+d%5E%7Bk+-+1%7D%28k%2C+j%29%29+)
![[公式]](https://www.zhihu.com/equation?tex=d%5E%7Bk%7D%28i%2C+j%29+%3D+min%28d%5E%7Bk+-+1%7D%28i%2C+j%29%2C+d%5E%7Bk+-+1%7D%28i%2C+k%29+%2B+d%5E%7Bk%7D%28k%2C+j%29%29+)
![[公式]](https://www.zhihu.com/equation?tex=d%5E%7Bk%7D%28i%2C+j%29+%3D+min%28d%5E%7Bk+-+1%7D%28i%2C+j%29%2C+d%5E%7Bk%7D%28i%2C+j%29+%2B+d%5E%7Bk+-+1%7D%28k%2C+j%29%29)
![[公式]](https://www.zhihu.com/equation?tex=d%5E%7Bk%7D%28i%2C+j%29+%3D+min%28d%5E%7Bk+-+1%7D%28i%2C+j%29%2C+d%5E%7Bk%7D%28i%2C+j%29+%2B+d%5E%7Bk%7D%28k%2C+j%29%29)

但是只有第一种形式是正确的，其余3中都是错误的，为啥呢？因为 k 不可能是![[公式]](https://www.zhihu.com/equation?tex=d%5Ek%28i%2Ck%29)或者![[公式]](https://www.zhihu.com/equation?tex=d%5Ek%28k%2Cj%29)的中间节点。我们拿 ![[公式]](https://www.zhihu.com/equation?tex=d%5Ek%28i%2Ck%29)举例，如果 k 是  ![[公式]](https://www.zhihu.com/equation?tex=d%5Ek%28i%2Ck%29)  的中间节点。那就说明从 i 到 k 的最短路径包含了 k，那岂不是 k 经过了两次，说明形成了环，而环是不可能作为最短路径的。



### 自己的理解

好，动态规划的公式，我们写出来了，接下来，我们再来理解一下这个公式的真正意义。

由公式可知，我们可以根据 K-1 的关系式得出 K 得关系式。这就意味着，**只要我们慢慢的加入顶点，就可以得出最小路径。**

这句话的意思是这样的：

想象一下，我们已经有了一个已求得各点最短路径的图。这个时候，我们往图里面添加一个顶点 X，我们就只需要求出各点经过 X 的路径，然后与原来的路径做比较即可。



那么，现在，应该就能理解 k 为什么在最外层了！

如果 k 在最里层，那么在遍历 k 之前，**没有保证各个点之间已经是最小路径**，自然，经过 k 点时，也不是最短路径。



这个算法，最神奇的地方在于：先求出各个点经过 V0 点的最短路径之后，再求出各个点经过 V1 点的最短路径时，这个最小距离已经包含了经过 V0 的最小距离。

所以，外层循环每遍历一次，都相当于更新了一次各个点之间的最短路径。也许，本来最短路径只经过 V0 点，现在变成了 V1 点，或者 V0 + V1 点。