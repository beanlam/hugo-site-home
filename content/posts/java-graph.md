---
title: "图与其Java实现"
date: 2015-04-17T18:07:51+08:00
categories : ["java"]
---


# 图的基本概念
图是什么，图是一种数据结构，一种非线性结构，所谓的非线性结构，浅显地理解的话，就是图的存储不是像链表这样的线性存储结构，而是由两个集合所组成的一种数据结构。

一个图中有两类东西，一种是结点，一种结点之间的连线。要用一种数据结构来表示的话，首先我们需要一个集合来存储所有的点，我们用V这个集合来表示（vertex），还需要另一个集合来存储所有的边，我们用E来表示(Edge)，那么一个图就可以表示为：

> **G = (V，E）**

有的图的边是有方向的，有的是没有方向的。

（A,B）表示A结点与B结点之间无方向的边，<A,B>则表示方向为从A到B的一条边，当然，如果是<B,A>，则方向相反。因此从边的方向我们就可以把图分为**有向图**和**无向图**两种。



## 其它概念

一个图中的元素有很多，例如：

> 完全图，邻接节点，结点的度，路径，权，路径长度，子图，连通图和强连通图，生成树，简单路径和回路...

本文只说说容易混淆的概念。

### 完全图，连通图，与强连通图
完全图可分为有向完全图和无向完全图两种，如果一个图的任意两个结点之间有且只有一条边，则称此图为无向完全图，若任意两个结点之间有且只有方向相反的两条边，则称为有向完全图。

那么连通图与完全图有什么区别呢？连通图是指在无向图中，若图中任意一对结点之间都有路径可达，则称这个无向图是连通图，而强连通图则是对应于有向图来说的，其特点与连通图是一样的。只不过是有向的，所以加了"强"。

连通图与完全图的区别就是，完全图要求任意两点之间有边，而连通图则是要求有路径。边和路径是有区别的。

### 邻接结点
一个结点的邻接节点，对于无向图来说，就是与这个结点相连的结点，至少有一个。

对于有向图来说，由于边是有方向的，所以一个结点的邻接节点是指以这个结点为开头，所指向的那些结点。

### 结点的度
度是针对结点来说的， 又分为出度和入度，看到“出度入度”，我们不难想到这是与边和边的方向有关的。
对于无向图来说，没有出度入度之分，一个结点的度就是经过这个结点的边的数目(或者是与这个结点相关联的边的数目)，对于有向图来说，出度就是指以这个结点为起始的边的条数（箭头向外），入度则是以这个点为终点的边的条数（箭头向内）。

**出 = 箭头向外，入 = 箭头向内**

### 权

权是指一条边所附带的数据信息，比如说一个结点到另一个结点的距离，或者花费的时间等等都可以用权来表示。带权的图也称为网格或网。

### 子图
跟一个集合有子集一样，图也有子图。可以类比理解。

# 存储结构
要存储一个图，我们知道图既有结点，又有边，对于有权图来说，每条边上还带有权值。常用的图的存储结构主要有以下二种：
*  邻接矩阵
*  邻接表

# 邻接矩阵
我们知道，要表示结点，我们可以用一个一维数组来表示，然而对于结点和结点之间的关系，则无法简单地用一维数组来表示了，我们可以用二维数组来表示，也就是一个矩阵形式的表示方法。

我们假设A是这个二维数组，那么A中的一个元素aij不仅体现出了结点vi和结点vj的关系，而且aij的值正可以表示权值的大小。

以下是一个**无向图**的邻接矩阵表示示例：

![无向图](/assets/java-graph/wuxiangtu.png)

从上图我们可以看到，无向图的邻接矩阵是**对称矩阵**，也**一定**是对称矩阵。且其左上角到右下角的对角线上值为零（对角线上表示的是相同的结点）

**有向图**的邻接矩阵是怎样的呢？

![有向图](/assets/java-graph/youxiangtu.png)

# 邻接表

我们知道，图的邻接矩阵存储方法用的是一个n*n的矩阵，当这个矩阵是稠密的矩阵（比如说当图是完全图的时候），那么当然选择用邻接矩阵存储方法。
可是如果这个矩阵是一个稀疏的矩阵呢，这个时候邻接表存储结构就是一种更节省空间的存储结构了。
对于上文中的无向图，我们可以用邻接表来表示，如下：

![无向图邻接表](/assets/java-graph/wuxiangtu_linjiebiao.png)

每一个结点后面所接的结点都是它的邻接结点。

## 邻接矩阵与邻接表的比较
当图中结点数目较小且边较多时，采用邻接矩阵效率更高。
当节点数目远大且边的数目远小于相同结点的完全图的边数时，采用邻接表存储结构更有效率。

# 邻接矩阵的Java实现

## 邻接矩阵模型类
邻接矩阵模型类的类名为AMWGraph.java，能够通过该类构造一个邻接矩阵表示的图，且提供插入结点，插入边，取得某一结点的第一个邻接结点和下一个邻接结点。

```java

import java.util.ArrayList;
import java.util.LinkedList;
/**
 * @description 邻接矩阵模型类
 * @author beanlam
 * @time 2015.4.17 
 */
public class AMWGraph {
    private ArrayList vertexList;//存储点的链表
    private int[][] edges;//邻接矩阵，用来存储边
    private int numOfEdges;//边的数目
    
    public AMWGraph(int n) {
        //初始化矩阵，一维数组，和边的数目
        edges=new int[n][n];
        vertexList=new ArrayList(n);
        numOfEdges=0;
    }
    
    //得到结点的个数
    public int getNumOfVertex() {
        return vertexList.size();
    }
    
    //得到边的数目
    public int getNumOfEdges() {
        return numOfEdges;
    }
    
    //返回结点i的数据
    public Object getValueByIndex(int i) {
        return vertexList.get(i);
    }
    
    //返回v1,v2的权值
    public int getWeight(int v1,int v2) {
        return edges[v1][v2];
    }
    
    //插入结点
    public void insertVertex(Object vertex) {
        vertexList.add(vertexList.size(),vertex);
    }
    
    //插入结点
    public void insertEdge(int v1,int v2,int weight) {
        edges[v1][v2]=weight;
        numOfEdges++;
    }
    
    //删除结点
    public void deleteEdge(int v1,int v2) {
        edges[v1][v2]=0;
        numOfEdges--;
    }
    
    //得到第一个邻接结点的下标
    public int getFirstNeighbor(int index) {
        for(int j=0;j<vertexList.size();j++) {
            if (edges[index][j]>0) {
                return j;
            }
        }
        return -1;
    }
    
    //根据前一个邻接结点的下标来取得下一个邻接结点
    public int getNextNeighbor(int v1,int v2) {
        for (int j=v2+1;j<vertexList.size();j++) {
            if (edges[v1][j]>0) {
                return j;
            }
        }
        return -1;
    }
}
```

## 邻接矩阵模型类的测试
接下来根据下面一个有向图来设置测试该模型类

![有向图测试](/assets/java-graph/youxiangtu_example.png)

TestAMWGraph.java测试程序如下所示：

```java

/**
 * @description AMWGraph类的测试类
 * @author beanlam
 * @time 2015.4.17
 */
public class TestAMWGraph {
    public static void main(String args[]) {
        int n=4,e=4;//分别代表结点个数和边的数目
        String labels[]={"V1","V1","V3","V4"};//结点的标识
        AMWGraph graph=new AMWGraph(n);
        for(String label:labels) {
            graph.insertVertex(label);//插入结点
        }
        //插入四条边
        graph.insertEdge(0, 1, 2);
        graph.insertEdge(0, 2, 5);
        graph.insertEdge(2, 3, 8);
        graph.insertEdge(3, 0, 7);
        
        System.out.println("结点个数是："+graph.getNumOfVertex());
        System.out.println("边的个数是："+graph.getNumOfEdges());
        
        graph.deleteEdge(0, 1);//删除<V1,V2>边
        System.out.println("删除<V1,V2>边后...");
        System.out.println("结点个数是："+graph.getNumOfVertex());
        System.out.println("边的个数是："+graph.getNumOfEdges());
    }
}
```

控制台输出结果如下图所示：

![控制台输出](/assets/java-graph/console1.png)

# 遍历
图的遍历，所谓遍历，即是对结点的访问。一个图有那么多个结点，如何遍历这些结点，需要特定策略，一般有两种访问策略：

* 深度优先遍历
* 广度优先遍历

## 深度优先
深度优先遍历，从初始访问结点出发，我们知道初始访问结点可能有多个邻接结点，深度优先遍历的策略就是首先访问第一个邻接结点，然后再以这个被访问的邻接结点作为初始结点，访问它的第一个邻接结点。总结起来可以这样说：每次都在访问完当前结点后首先访问当前结点的第一个邻接结点。

我们从这里可以看到，这样的访问策略是优先往纵向挖掘深入，而不是对一个结点的所有邻接结点进行横向访问。

具体算法表述如下：

1. 访问初始结点v，并标记结点v为已访问。
2. 查找结点v的第一个邻接结点w。
3. 若w存在，则继续执行4，否则算法结束。
4. 若w未被访问，对w进行深度优先遍历递归（即把w当做另一个v，然后进行步骤123）。
5. 查找结点v的w邻接结点的下一个邻接结点，转到步骤3。

例如下图，其深度优先遍历顺序为 `1->2->4->8->5->3->6->7`


![深度优先](/assets/java-graph/depth-first.png)

## 广度优先
类似于一个分层搜索的过程，广度优先遍历需要使用一个队列以保持访问过的结点的顺序，以便按这个顺序来访问这些结点的邻接结点。

具体算法表述如下：

1. 访问初始结点v并标记结点v为已访问。
2. 结点v入队列
3. 当队列非空时，继续执行，否则算法结束。
4. 出队列，取得队头结点u。
5. 查找结点u的第一个邻接结点w。
6. 若结点u的邻接结点w不存在，则转到步骤3；否则循环执行以下三个步骤：
    1). 若结点w尚未被访问，则访问结点w并标记为已访问。
    2). 结点w入队列
    3). 查找结点u的继w邻接结点后的下一个邻接结点w，转到步骤6。

如下图，其广度优先算法的遍历顺序为：1->2->3->4->5->6->7->8


![广度优先](/assets/java-graph/width-first.png)


## Java实现
上文中已经给出了邻接矩阵图模型类 `AMWGraph.java`，在原先类的基础上增加了两个遍历的函数，分别是 `depthFirstSearch()` 和 `broadFirstSearch()` 分别代表深度优先和广度优先遍历。

```java
import java.util.ArrayList;
import java.util.LinkedList;
/**
 * @description 邻接矩阵模型类
 * @author beanlam
 * @time 2015.4.17 
 */
public class AMWGraph {
    private ArrayList vertexList;//存储点的链表
    private int[][] edges;//邻接矩阵，用来存储边
    private int numOfEdges;//边的数目

    public AMWGraph(int n) {
        //初始化矩阵，一维数组，和边的数目
        edges=new int[n][n];
        vertexList=new ArrayList(n);
        numOfEdges=0;
    }

    //得到结点的个数
    public int getNumOfVertex() {
        return vertexList.size();
    }

    //得到边的数目
    public int getNumOfEdges() {
        return numOfEdges;
    }

    //返回结点i的数据
    public Object getValueByIndex(int i) {
        return vertexList.get(i);
    }

    //返回v1,v2的权值
    public int getWeight(int v1,int v2) {
        return edges[v1][v2];
    }

    //插入结点
    public void insertVertex(Object vertex) {
        vertexList.add(vertexList.size(),vertex);
    }

    //插入结点
    public void insertEdge(int v1,int v2,int weight) {
        edges[v1][v2]=weight;
        numOfEdges++;
    }

    //删除结点
    public void deleteEdge(int v1,int v2) {
        edges[v1][v2]=0;
        numOfEdges--;
    }

    //得到第一个邻接结点的下标
    public int getFirstNeighbor(int index) {
        for(int j=0;j<vertexList.size();j++) {
            if (edges[index][j]>0) {
                return j;
            }
        }
        return -1;
    }

    //根据前一个邻接结点的下标来取得下一个邻接结点
    public int getNextNeighbor(int v1,int v2) {
        for (int j=v2+1;j<vertexList.size();j++) {
            if (edges[v1][j]>0) {
                return j;
            }
        }
        return -1;
    }
    
    //私有函数，深度优先遍历
    private void depthFirstSearch(boolean[] isVisited,int  i) {
        //首先访问该结点，在控制台打印出来
        System.out.print(getValueByIndex(i)+"  ");
        //置该结点为已访问
        isVisited[i]=true;
        
        int w=getFirstNeighbor(i);//
        while (w!=-1) {
            if (!isVisited[w]) {
                depthFirstSearch(isVisited,w);
            }
            w=getNextNeighbor(i, w);
        }
    }
    
    //对外公开函数，深度优先遍历，与其同名私有函数属于方法重载
    public void depthFirstSearch() {
        for(int i=0;i<getNumOfVertex();i++) {
            //因为对于非连通图来说，并不是通过一个结点就一定可以遍历所有结点的。
            if (!isVisited[i]) {
                depthFirstSearch(isVisited,i);
            }
        }
    }
    
    //私有函数，广度优先遍历
    private void broadFirstSearch(boolean[] isVisited,int i) {
        int u,w;
        LinkedList queue=new LinkedList();
        
        //访问结点i
        System.out.print(getValueByIndex(i)+"  ");
        isVisited[i]=true;
        //结点入队列
        queue.addlast(i);
        while (!queue.isEmpty()) {
            u=((Integer)queue.removeFirst()).intValue();
            w=getFirstNeighbor(u);
            while(w!=-1) {
                if(!isVisited[w]) {
                        //访问该结点
                        System.out.print(getValueByIndex(w)+"  ");
                        //标记已被访问
                        isVisited[w]=true;
                        //入队列
                        queue.addLast(w);
                }
                //寻找下一个邻接结点
                w=getNextNeighbor(u, w);
            }
        }
    }
    
    //对外公开函数，广度优先遍历
    public void broadFirstSearch() {
        for(int i=0;i<getNumOfVertex();i++) {
            if(!isVisited[i]) {
                broadFirstSearch(isVisited, i);
            }
        }
    }
}
```

上面的public声明的depthFirstSearch()和broadFirstSearch()函数，是为了应对当该图是非连通图的情况，如果是非连通图，那么只通过一个结点是无法完全遍历所有结点的。

下面根据上面用来举例的图来构造测试类：

```java
public class TestSearch {

    public static void main(String args[]) {
        int n=8,e=9;//分别代表结点个数和边的数目
        String labels[]={"1","2","3","4","5","6","7","8"};//结点的标识
        AMWGraph graph=new AMWGraph(n);
        for(String label:labels) {
            graph.insertVertex(label);//插入结点
        }
        //插入九条边
        graph.insertEdge(0, 1, 1);
        graph.insertEdge(0, 2, 1);
        graph.insertEdge(1, 3, 1);
        graph.insertEdge(1, 4, 1);
        graph.insertEdge(3, 7, 1);
        graph.insertEdge(4, 7, 1);
        graph.insertEdge(2, 5, 1);
        graph.insertEdge(2, 6, 1);
        graph.insertEdge(5, 6, 1);
        graph.insertEdge(1, 0, 1);
        graph.insertEdge(2, 0, 1);
        graph.insertEdge(3, 1, 1);
        graph.insertEdge(4, 1, 1);
        graph.insertEdge(7, 3, 1);
        graph.insertEdge(7, 4, 1);
        graph.insertEdge(6, 2, 1);
        graph.insertEdge(5, 2, 1);
        graph.insertEdge(6, 5, 1);
        
        System.out.println("深度优先搜索序列为：");
        graph.depthFirstSearch();
        System.out.println();
        System.out.println("广度优先搜索序列为：");
        graph.broadFirstSearch();
    }
}
```

运行后控制台输出如下:

![控制台输出](/assets/java-graph/console2.png)