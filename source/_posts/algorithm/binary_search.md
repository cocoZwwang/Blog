---
title: 二分
date: 2024-02-04 22:38:26
categories: 算法
mathjax: true
---
## 查找值

这是二分搜索最简单的用法，在**有序**的数组中查找某个值是否存在或者最接近某个值的值。

C++ STL 实现了二分查找值的算法，可以直接使用
```c++
    //返回第一个大于或者等于 x 的下标，如果不存在则返回 n
    int i=lower_bound(a,a+n,x) - a;
    //返回第一个大于或者等于 x 的迭代器，如果不存在则返回尾迭代器。
    auto it=lower_bound(a.begin(),a.end(),x);

    //返回第一个大于 x 的下标，如果不存在则返回 n
    int i=upper_bound(a,a+n,x) - a;
    //返回第一个大于或者等于 x 的迭代器，如果不存在则返回尾迭代器。
    auto it=upper_bound(a.begin(),a.end(),x);
    
    //注意： 如果是 set 和 map 这种关联式容器，使用上面的两个方法的时间复杂度是 O(n)，需要使用容器自带的方法，时间复杂度为 O(logn)。
    auto it=s.lower_bound(x);
    
    //快速找出有序数组 a 中 x 的数量。
    int cnt=upper_bound(a,a+n,x) - lower_bound(a,a+n,x);
    int cnt=upper_bound(a.begin(),a.end(),x) - lower_bound(a.begin(),a.end(),x);
    
    //有序数组 a 中 小于 x 的数量
    int cnt=lower_bound(a,a+n,x) - a;
    int cnt=lower_bound(a.begin(),a.end(),x) - a.begin();
    
    //有序数组 a 中 小于等于 x 的数量
    int cnt=upper_bound(a,a+n,x) - a;
    int cnt=upper_bound(a.begin(),a.end(),x) - a.begin();
```

## 假定判断

假定判断就是先假定一个解并且判断是否可行，在二分中使用也是非常的频繁，例如求解**最大化**或者**最小化**问题。

[例题：POJ 1064 Cable master](http://poj.org/problem?id=1064)

```c++
#include <cstdio>
#include <algorithm>
#include <cstring>
#include <iostream>
#include <math.h>
using namespace std;
const int N=10010;
int n,k;
double L[N];

int calc(double len){
    int res=0;
    for(int i=0;i<n;++i) res+=(int)(L[i]/len);
    return res;
}
//标签：二分 求最大值 实数二分
// #define LOCAL
int main(){
    #ifdef LOCAL
        freopen("input.txt","r",stdin);
        freopen("output.txt","w",stdout);
    #endif
    scanf("%d %d",&n,&k);
    for(int i=0;i<n;++i){
        scanf("%lf",&L[i]);
    }
    double l=0,r=100010;
    //100 次循环精度可以达到 10^-30，基本上是没有问题的
    for(int i=0;i<100;i++){
       double md=(l+r)/2;
       if(calc(md)>=k) l=md;
       else r=md;
    }
    printf("%.2f\n",floor(r*100)/100);
    return 0;
}
```



## 最大化最小值
[例题：POJ 2456  Aggressive cows](http://poj.org/problem?id=2456)

```c++
#include <cstdio>
#include <cstring> 
#include <algorithm>
#include <iostream>
using namespace std;
const int N=100010;
int n, c,x[N];

bool check(int k,int cnt){
    cnt--;
    for(int i=1,j=0;i<n&&cnt;i++){
        if(x[i]-x[j]>=k){
            j=i;
            cnt--;
        }
    } 
    return cnt==0;
}
//标签：二分 最大值最小
// #define LOCAL
int main(){
    #ifdef LOCAL
        freopen("input.txt","r",stdin);
        freopen("output.txt","w",stdout);
    #endif
    scanf("%d %d",&n,&c);
    for(int i=0;i<n;++i) scanf("%d",&x[i]);
    sort(x,x+n);
    int l=0,r=x[n-1]-x[0];
    while(l<=r){
        int md=l+(r-l)/2;
        if(check(md,c)) l=md+1;
        else r=md-1;
    }
    printf("%d\n",r);
}
```



**最大化最小值**或者**最小化最大值** 经常也会带有一定的贪心算法在里面，通常借助排序或者优先队列来建立判断条件。
## 最大化平均值
**例题：**有 n 个物品的重量和价值分别是 wi 和 vi。从中选出 k 个物品使得单位重量的价值最大。

**限制条件：** 1<=k<=n<1000；1<=wi,vi<=1000000

**样例：**

> 输入：n=3；k=2；(w,v)={(2,2),(5,3),(2,1)}
>
> 输出：0.75 (选择 0 号和 2 号物品，(2+1)/(2+2)=0.75)

解题思路：
    一般最先想到的方法是按照单位价值进行排序，从大到小贪心地进行选择，但是这种方法对于上面的例子的结果是 0.714，所以这种方式不可行。

我们可以使用二分解决这个问题，假定判断条件：

$$
C(x)=可以通过选择使得单位价值不小于 x
$$
那么原问题就变成了求满足 C(x) 的最大的 x。假设我们选择的集合是 s，因此它们的单位价值是：

$$
\sum_{k \in s} vi \div \sum_{k \in s} wi \geq x
$$

把上面不等式进行变形：

$$
\sum_{k \in s} (vi-x \times wi) \geq 0
$$

因此可以对 
$$
(vi-x \times wi)
$$
的值进行排序，然后贪心地进行选取：

$$
C(x)=(vi-x \times wi)从大到小前 k 个和不小于0
$$

```c++
#include <algorithm>
using namespace std;
const int MAX_N = 10010;
int n, k;
int w[MAX_N], v[MAX_N];
double y[MAX_N], INF;

//判断是否满足条件
bool C(double x){
    for (int i = 0; i < n; ++i){
        y[i] = v[i] - w[i] * x;
    }
    sort(y, y + n);

    //计算最大的 K 个 yi 的和
    double sum = 0;
    for (int i = 0; i < k; ++i){
        sum += y[n - 1 - i];
    }
    return sum >= 0;
}

void solve(){
    double l = 0, r = INF;
    for (int i = 0; i < 100; ++i){
        double md = (l + r) / 2;
        if (C(md))
            l = md;
        else
            r = md;
    }
    printf("%.2f\n", l);

```



## 查找第 K 个值

如果是有序序列（比如：数组），我们可以直接读取第 k 个值，但是有时候我们并不能高效地定位到第 k 个值，或者第 K 个值是动态的，比如二维矩阵、动态查询等，这时候我们可以尝试考虑二分查找。

利用二分来查找第 k 个值，一般基于这么一个判断：设定一个值 x，如果 x **小于**第 k 个值，那么**小于等于** x 的个数一定**小于** k；反过来如果 x **大于等于**第 k 个值，那么**小于等于** x 的个数一定**不小于** k，这其实就是一个假定判断，因此这种方法的核心就是计数。


> ps ：在有序的序列中，upper_bound 和 lower_bound 能非常方便的计算小于某个值的个数。

**例题：**[leetcode 668. 乘法表中第k小的数](https://leetcode.cn/problems/kth-smallest-number-in-multiplication-table/)
    几乎每一个人都用 乘法表。但是你能在乘法表中快速找到第 k 小的数字吗？乘法表是大小为 m x n 的一个整数矩阵，其中 mat[i][j] == i * j（下标从 1 开始）。给你三个整数 m、n 和 k，请你在大小为 m x n 的乘法表中，找出并返回第 k 小的数字。

**参考思路：**就是使用上面的判断方法来进行二分，关键是计数,这里很简单，因为
$$
mat[i][j]=i \times j
$$
因此每一行小于等于 x 的个数就是 
$$
min(n,x/i)
$$
m x n 的矩阵可以在 O(m) 的时间复杂度内进行每次的计数。

```c++
class Solution {
public:
    int findKthNumber(int m, int n, int k) {
        if(m>n) swap(m,n);
        int l=1;
        int r=9e8;
        while(l<=r){
            int mid=l + (r-l)/2;
            if(calc(m,n,mid)<k) l=mid+1;
            else r=mid-1;
        }
        return l;
    }

    int calc(int m, int n,int mid){
        int res=0;
        for(int i=1;i<=m;++i){
            res+=n*i<=mid?n:mid/i;
        }
        return res;
    }
};
```



## 最小（大）化第 K 值

**例题：**[POJ 2010 Moo University - Financial Aid](http://poj.org/problem?id=2010)"
    农场新建了一所奶牛大学，想从一共 $C$ 头牛中招收 $N$ 头牛作为学生（$N <= C$，而且 $N$ 是奇数），每头牛都有一个如下前的考试分数 $ci$ 和一个需要补助的学费 $fi$。但是目前大学总的补助经费只有 $F$，因此不能随意招收任意的奶牛。现在你是招生办主任，你要在不超过补助经费 $F$ 的情况下，从 $C$ 头牛中选 $N$ 头录取，并且分数中位数最大。

**思路分析：**
    由于` N `是奇数，因此就是让第 `N/2 + 1` 高的分数最大，怎么办？我们可以先考虑最暴力的解法，求出所有的符合条件的 `N`头牛的不同组合，然后对比这些组合中第 `N/2 + 1` 高的分数，那个最大。但是组合数太大了，而且如果我们把每个组合的分数排序然后取第 `N/2 + 1` 个分数，那总的时间复杂度需要 
$$
\displaystyle \binom{C}{N} \times O(NlogN)
$$
​    

如果我们像上面那样使用二分来取第 `K` 大值呢，好像时间复杂度还是 `O(NlogN)`，但是它可以使用一些贪心算法，我们没必要比对每个组合。我们回忆一下上面求第 `K` 大值的方法，我们根据当前小于等于 `x` 的个数 `count` 来决定 `x` 是该增大还是减少，假如 `count` 小于 `K`，那么 `x` 就可以增大。那么回到当前，如果这些组合里面至少存在一个组合的 `count` 比 `K` 小，那么 `x` 就可以增大了，换句话说只要存在有 `count` 比 `K` 小，`x` 就可以增大，那么我们是不是可以只和最小的 `count` 比就行了。那么我们如何让比 `x` 小的个数尽量少呢？在经费不超过 `F` 的情况下，尽量选分数比 `x` 大就行。但是需要额外先判断是否存在解，只有一定存在解才进入二分的逻辑，可以避免特殊边界的判断。

```c++
#include <cstdio>
#include <iostream>
#include <algorithm>
#include <cstring>
#include <queue>
using namespace std;
typedef long long LL;
const int MAX_C=100010;
int N,C,F,L[MAX_C],R[MAX_C];
struct cow{
    int c,f;
}cows[MAX_C];

int cmp(const cow& a, const cow& b){
    return a.c<b.c;
}
// 标签：二分 最小化第 K 大的值 
int calc(int x){
    for(int i=0;i<C;++i){
        L[i]=cows[i].f;
    }
    int i,j,l,r;
    for(i=0,l=0,r=0;i<C;++i){
       if(cows[i].c<=x) L[l++]=cows[i].f; 
       else R[r++]=cows[i].f;
    }
    int res=0;
    sort(L,L+l);//分数小于等于 x 的奶牛的补助放到一个数组，称为左边
    sort(R,R+r);//分数比 x 大的奶牛的补助放到一个数组，称为右边
    LL s=0,cnt=0;
    for(i=0;cnt<N&&i<r;++i) s+=R[i],++cnt;//优先右边选，并且从小往大选，这样能选尽量多。
    j=i;
    for(i=0;i<l&&cnt<N;++i) s+=L[i],++cnt;//如果右边的奶牛总算不够 N 头，从左边补够N头。
    for(;i<l&&s>F;++i) s+=L[i]-R[--j];//如果总的补助大于 F，那么把右边最大替换掉，再用左边较小的替换。
    res=i;//左边一共选择了多少头，就是有多个分数小于等于 x。
    return res;
}

// #define LOCAL
int main(){
    #ifdef LOCAL
        freopen("input.txt","r",stdin);
        freopen("output.txt","w",stdout);
    #endif
    scanf("%d %d %d",&N,&C,&F);
    for(int i=0;i<C;++i){
       scanf("%d %d",&cows[i].c,&cows[i].f);
       L[i]=cows[i].f;
    }
    sort(L,L+C);
    LL s=0; int ans=-1;
    for(int i=0;i<N;++i) s+=L[i];
    //判断是否无解，如果无解不需要二分
    if(s<=F){
        sort(cows,cows+C,cmp);
        int l=0,r=C-1,k=N/2+1;
        while(l<=r){
            int md=(l+r)/2;
            if(calc(cows[md].c)<k) l=md+1;
            else r=md-1;
        }
        ans=cows[l].c;
    }
    printf("%d\n",ans);
    return 0;
}
```



**例题：**[POJ 3662 Telephone Lines](http://poj.org/problem?id=3662)
    FJ 想在电话公司和自己的农场之间建立电话线路，它们之间有 n 根电线杆，1 和 n 号电线杆分别在电话公司和农场内，这些电线杆之间有的可以相互连接(a,b,l 表示a 和 b 连接的费用是 l)，有的不行。电话公司愿意支付 k 根最贵的电话线，剩下的电话线中最贵的一根的费用就是 FJ 需要支付的最终费用。求 FJ 可以建立电话线路需要的最小费用。

**解题思路：**这是一张无向图，如果 `1` 和 `n` 之间非连通，则返回 `-1`，如果连通需要的最少边数小于等于 `k` 则费用是`0`，如果边数大于 `k` ，则第 `k+1` 贵的电线就是 FJ 要支付的费用，因此问题就是最小化第 `k+1`**大**的值。

连通性和连通的最少边数可以使用 BFS 求出，这个非常简单，问题是如何才能最小化第 `K+1` 大的值。不妨先想办法找出每条路径第 `k+1` 大的值，然后取它们中的最小值，总体上的思路和上面 POJ 2010 类似，使用二分 + 贪心。我们依然使用 `x` 来判断，如果大于等于 `x` 的边数 `count` 少于 `k+1`，那么说明 `x` 还可以更小，因此我们只需要和最小的 `count` 比就行。我们设大于等于 `x` 的边权为 `1`，其他为 `0`，那么最小的 `count` 就是最短路径，可以使用 dijkstra 算法。

```c++
#include <cstdio>
#include <iostream>
#include <cstring>
#include <string>
#include <algorithm>
#include <queue>
#include <climits>
using namespace std;
typedef pair<int,int> pi;
const int P=20100; // 要记得向前星无向边的容量是边数的两倍
const int N=1010;
int n,p,k,d[N],e[P];
priority_queue<pi,vector<pi>,greater<pi>> pq; //小根堆
struct Edge{
    int to, w,next;
}edges[P];
int id,head[P];

void add(int u, int v, int w){
    edges[++id].to=v;
    edges[id].next=head[u];
    edges[id].w=w;
    head[u]=id;
}

int djk(int x){
    memset(d,-1,sizeof(d));
    while(pq.size()) pq.pop(); //这里一定要注意清空队列
    d[1]=0;
    pq.push(make_pair(0,1));
    while(pq.size()){
        pi t=pq.top(); pq.pop();
        int w=t.first, u=t.second;
        if(u==n) break;
        if(d[u]!=-1&&d[u]<w) continue;
        for(int i=head[u];i;i=edges[i].next){
            int v=edges[i].to;
            if(d[v]==-1||(d[v]>d[u]+(edges[i].w>=x))){
                d[v]=d[u] + (edges[i].w>=x);
                pq.push(make_pair(d[v],v));
            }
        }
    }
    return d[n];
}

//#define LOCAL
int main(){
    #ifdef LOCAL
        freopen("input.txt","r",stdin);
        freopen("output.txt","w",stdout);
    #endif
    scanf("%d %d %d",&n,&p,&k);
    int u,v,w;
    for(int i=0;i<p;++i){
       scanf("%d %d %d",&u,&v,&w);
       add(u,v,w);
       add(v,u,w);
       e[i]=w;
    }
    int mik=djk(0);
    if(mik<0) printf("%d\n",-1);
    else if(mik<=k) printf("%d\n",0);
    else {
        sort(e,e+p);
        int l=1,r=p-1;
        while(l<=r){
            int md=(l+r)/2;
            int ret=djk(e[md]);
            if(ret<k+1) r=md-1;
            else l=md+1;
        }
        printf("%d\n",e[r]);
    }
}

```

通过上面的两条例题可以看出，最小化第 K 大值，是通过`二分查找第 K 值` + `贪心` 来实现的，`二分查找第 K 值` 中我们通过计算小于等于 x 的个数来判断 x 是在第 K 值的左边还是右边，而这里则使用最优的方案来计算小于等于 x 的个数，因此最后得出的第 K 值也是最优的。

## 其他
### 折半舍去
我们在判断一条题目是否可以使用二分的时候，最常用的一个方法就是判断其是否具有单调性，但其实只要能保证每次迭代的区间一定存在解，那么通过区间收敛就一定能获取解，单调性能比较容易看出并且实现这一点，但并不代表只有单调性才能使用二分。

**例题：**[leetcode 852. 山脉数组的峰顶索引](https://leetcode.cn/problems/peak-index-in-a-mountain-array/)
    给你由整数组成的山脉数组 arr ，返回任何满足 `arr[0] < arr[1] < ... arr[i - 1] < arr[i] > arr[i + 1] > ... > arr[arr.length - 1] 的下标 i。0<i<arr.length-1`;

**思路分析：**如果区间 `[l,r] `存在顶点，那么对于区间的 `x`， 如果有 `arr[x+1]>arr[x]`，那么顶点一定在 `[x+1,r]`，反过来如果 `arr[x]>arr[x+1]`，那么顶点一定在 `[l,x]`。

```c++
class Solution {
    public int peakIndexInMountainArray(int[] arr) {
        int n=arr.length;
        int l=0, r=n-1;
        while(l<r){
            int md=(l+r)/2;
            if(arr[md+1]>arr[md]) l=md+1;
            else r=md;
        }
        return l;
    }
}
```



**例题：**[Uva 1607 与非门电路](https://onlinejudge.org/index.php?option=com_onlinejudge&Itemid=8&category=825&page=show_problem&problem=4482)

**参考思路：** 因为 `n` 个输入接收同一个 `x`，因此整个电路的功能无非就是 `4` 种：常数 `0`；常数 `1`；`x`，非 `x`。

我们可以先把所有的 `x` 全部设置为 `0`，再设置为 `1`，如果两者输出一致，那说明结果是常数，那么可以把所有的输入全部设置成 `0`，或者全部设置成 `1`。

如果输出不同，那么我们可以只需要讨论其中一种，因为它们逻辑是一样的。我们设 `x=0`时候，输出 `0`；`x=1` 时候，输出 `1`。现在我们把第一个输入改成 `1`，其他还是 `0`，如果输出是 `1`，那么就得到一个解` x00...00`；如果输出还是 `0 `，再把第二个也改成 `1`，如果输出 `1`，则又找到一个解 `1x00...0`，如果还是 `0`，再继续尝试`11100...00`，如此等等。由于全为 `1` 的时候，则一定会输出 `1`，这表示一定存在解。那为什么不直接使用 `111...1x` 呢？ 因为虽然全为 `1` 时输出一定为 `1`，但是 `11...10 `输出不一定为`0`，也有可能为 `1`，那么 `111...1x `输出就变成了常数。

我们设 
$$
f(i)=输入为 i 个 1 时的输出
$$
那么如果存在 `f(i)=0 `和`f(i+1)=1 `，则 `i `和 `i+1 `就是我们要找的临界点，题目中可能会存在多个这样的临界点，但是我们只需要找到其中一个即可。我们设区间`[l,r]`满足`f(l)=0` 和 `f(r)=1`，那么如果区间中的 `i `有 `f(i)=0`，那么区间可以缩小为`[i,r]`，反之如果`f(i)=1`，那么区间可以缩小为`[l,i]`，最终收敛为 `(i,i+1]`。

```c++
#include <bits/stdc++.h>
using namespace std;
const int N=100010;
const int M=200010;
struct gate{
    int a,b,o;
}gates[M];
int n,m;

int calc(int cnt){
    for(int i=1;i<=m;++i){
        int a=gates[i].a;
        int b=gates[i].b;
        int ia=a<0?(-a<=cnt):gates[a].o;
        int ib=b<0?(-b<=cnt):gates[b].o;
        gates[i].o=~(ia&ib);
        gates[i].o&=1;
    }
    return gates[m].o;
}

// #define LOCAL
int main(){
    #ifdef LOCAL
        freopen("input.txt","r",stdin);
        freopen("output.txt","w",stdout);
    #endif
    int T;
    scanf("%d",&T);
    while(T-->0){
        scanf("%d %d",&n,&m);
        for(int i=1;i<=m;++i){
            scanf("%d %d",&gates[i].a,&gates[i].b);
        }
        int u=calc(0);
        int v=calc(n);
        if(u==v){
            for(int i=0;i<n;++i) printf("%c",'1');
            printf("\n");
        }else{
            int l=0,r=n;
            while(l<r){
                int md=(l+r)/2;
                if(calc(md)==u) l=md+1;
                else r=md;
            } 
            char ch;
            for(int i=1;i<=n;++i){
                ch=i<r?'1':(i==r?'x':'0');
                printf("%c",ch);
            }
            printf("\n");
        }
    }
    return 0;
}

```



## 参考资料
《算法竞赛入门经典-第 2 版》

《挑战程序设计竞赛》

[《oi-wiki》](https://oi-wiki.org/)

[《leetcode 网站》](https://leetcode.cn/problemset/all/)






