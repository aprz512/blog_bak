## 最短路径之Dijkstra算法

看一个动图：

![img](https://upload.wikimedia.org/wikipedia/commons/thumb/e/e4/DijkstraDemo.gif/220px-DijkstraDemo.gif)



**算法原理**：

**1)** 初始时，S只包含起点s；U包含除s外的其他顶点，且U中顶点的距离为"起点s到该顶点的距离"[例如，U中顶点v的距离为(s,v)的长度，然后s和v不相邻，则v的距离为∞]。 

**(2)** 从U中选出"距离最短的顶点k"，并将顶点k加入到S中；同时，从U中移除顶点k。 

**(3)** 更新U中各个顶点到起点s的距离。之所以更新U中顶点的距离，是由于上一步中确定了k是求出最短路径的顶点，从而可以利用k来更新其它顶点的距离；例如，(s,v)的距离可能大于(s,k)+(k,v)的距离。 

**(4)** 重复步骤(2)和(3)，直到遍历完所有顶点。 



### 代码

```c++
/*
 * Dijkstra最短路径。
 * 即，统计图中"顶点vs"到其它各个顶点的最短路径。
 *
 * 参数说明：
 *       vs -- 起始顶点(start vertex)。即计算"顶点vs"到其它顶点的最短路径。
 *     prev -- 前驱顶点数组。即，prev[i]的值是"顶点vs"到"顶点i"的最短路径所经历的全部顶点中，位于"顶点i"之前的那个顶点。
 *     dist -- 长度数组。即，dist[i]是"顶点vs"到"顶点i"的最短路径的长度。
 */
void MatrixUDG::dijkstra(int vs, int prev[], int dist[])
{
    int i,j,k;
    int min;
    int tmp;
    int flag[MAX];

    // 初始化
    for (i = 0; i < mVexNum; i++)
    {
        flag[i] = 0;
        prev[i] = 0;
        dist[i] = mMatrix[vs][i];
    }

    // vs 是起点
    flag[vs] = 1;
    dist[vs] = 0;

    for (i = 1; i < mVexNum; i++)
    {
        // dist[] 储存的是各个顶点距离起点的距离
        // 寻找里面的最小值
        min = INF;
        for (j = 0; j < mVexNum; j++)
        {
            if (flag[j]==0 && dist[j]<min)
            {
                min = dist[j];
                k = j;
            }
        }
        
        // 认为 k 是离起点最近的，那么 k 之后就不再参与遍历
        flag[k] = 1;

        // 利用 k 作为中转点，更新其他点到起点的距离
        for (j = 0; j < mVexNum; j++)
        {
            tmp = (mMatrix[k][j]==INF ? INF : (min + mMatrix[k][j]));
            if (flag[j] == 0 && (tmp  < dist[j]) )
            {
                // 发现j点经过k点到起点更短
                // 将新的最短路径更新到 dist 中
                dist[j] = tmp;
                // prev 是记录的最短路径
                prev[j] = k;
            }
        }
    }

    // 打印dijkstra最短路径的结果
    cout << "dijkstra(" << mVexs[vs] << "): " << endl;
    for (i = 0; i < mVexNum; i++)
        cout << "  shortest(" << mVexs[vs] << ", " << mVexs[i] << ")=" << dist[i] << endl;
}
```

dist[i] 储存的是 i 到起点的最短路径值，没有记录详细的路径节点，前驱节点在 prev[i] 中储存着。

dist 会不断借助中转点进行更新。



不看算法，只看算法思想，需要用到邻接矩阵，所以复杂度为 O(n * n)；



### 参考文档

https://www.cnblogs.com/msymm/p/9769915.html