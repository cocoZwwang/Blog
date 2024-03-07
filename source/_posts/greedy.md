---
title: 贪心
date: 2024-02-05 01:27:24
tags:
categories: 算法
mathjax: true
thumbnail:
---
## 区间问题

### 区间调度问题

这种问题常常出现再工作或者会议安排上，其实就是求**不相交区间数**最大。

#### 例题：参与尽量多的工作

​    有 `n` 项工作，每项工作分别再时间 `si `开始，在时间 `ti`结束，对于每项工作，你都可以选择参与与否，但是如果选择参与，就必须完成该项工作，并且该工作时间段内不能参与任何其他工作。你的目标是参与尽量多的工作，返回该最大值。

<!--more-->

**限制条件：** 

- 1<=N<=100000;

- 1<=si<=ti<=1e9;

**例子：** 

> 输入：n=5; s={1,2,4,6,8}; t={3,5,7,9,0}    
>
> 输出：3 （选取 1、3、5）

**参考思路：**
    一般我们容易想到下面几个思路：

1. 优先选择最早开始的：很明显会错误，最早开始也可以最迟结束，这样只能选择一个工作

2. 优先选择**最早结束的**：是正确的，优先做尽量早结束的

3. 优先选择用时最少的：一个用时很短的工作 C，可以连接两个本不相交的工作 A、B，这样本来可以选择 2 个变成了只能选择一个

4. 优先选择最少重叠的：反例如下，第二行中间任务的重叠数最少，但如果优先选择此任务，最多只能完成 3 项，而实际全部选择第三行的任务，可以完成 4 项。

   >$~~~$$~~~$$~~~$————$~~~$$~~~$$~~~$$~~~$$~~~$$~~~$$~~~$$~~~$————
   >
   >$~~~$$~~~$$~~~$————$~~~$————$~~~$————
   >
   >————$~~~$————$~~~$————$~~~$————

```c++
//区间调度问题
typedef pair<int,int> pi;
pi itv[MAX_N];
int solve(int n, vector<int> s, vector<int> t){
    for(int i=0;i<n;++i){
        itv[i].second=s[i];
        itv[i].first=t[i];//结束时间作 first
    }
    sort(itv.begin(),itv.end());
    int ans=0;
    for(int i=0,t=0;i<n;++i){
        if(itv[i].second>t){
            ans++;
            t=itv[i].first;
        }
    }
    return ans;
}
```

### 区间覆盖问题

数轴上有 n 个闭区间 [ai,bi]，选择**尽量少**的区间，覆盖一条指定线段 [s,t]。突破口仍然是区间包含和区间排序，不过需要一次预处理。

- 每个区间在  [s,t]  外面的部分都应该切掉，因为它们是多余的。
- 预处理后，在相互包含的情况下，小区间显然是不应该考虑的。
- 把各个区间按照 `ai` 从小到大排序，如果第一个区间的起点不是 s，那么无解；否则选择起点为 s 的区间里面最长的一个，更新 s 为刚刚选择区间的 bi。
- 重复第一步，直到 s>=t，或者区间已经被用完。

#### 例题：[leetcode 1024 视频拼接](https://leetcode.cn/problems/video-stitching/)

​    你将会获得一系列视频片段，这些片段来自于一项持续时长为 time 秒的体育赛事。这些片段可能有所重叠，也可能长度不一。使用数组 clips 描述所有的视频片段，其中 clips[i] = [starti, endi] 表示：某个视频片段开始于 starti 并于 endi 结束。甚至可以对这些片段自由地再剪辑：例如，片段 [0, 7] 可以剪切成 [0, 1] + [1, 3] + [3, 7] 三部分。我们需要将这些片段进行再剪辑，并将剪辑后的内容拼接成覆盖整个运动过程的片段（[0, time]）。返回所需片段的最小数目，如果无法完成该任务，则返回 -1 。

```c++
class Solution {
    typedef pair<int,int> pi;
public:
    int videoStitching(vector<vector<int>>& clips, int time) {  
        vector<pi> ps;
        for(vector<int> c : clips){
            ps.push_back({max(0,c[0]),min(time,c[1])});
        }
        int n=ps.size();
        //区间按照坐标开始时间从小到达排序
        sort(ps.begin(),ps.end());
        int ans=0,pre=0;
        //判断条件别漏了 pre < time
        for(int i=0;i<n&&pre<time;++i){
            //如果没有区间能覆盖当前的最左边，则返回-1
            if(ps[i].first>pre) return -1;
            int j=i;
            int t=ps[i].second;
            //所有能覆盖最左边的区间中选择最长的一个
            while(i+1<n&&ps[i+1].first<=pre) t=max(t,ps[++i].second);
            pre=t;// 更新需要覆盖的最左边时间
            ans++;
        }
        return pre >=time ? ans : -1;
    }
};

```



> 区间问题（尤其覆盖问题）一定要注意：

> - 区间的边界是否可以交接，要小心处理 s[i].end 和 s[i+1].start。
> - 区间覆盖要记得 [s,t] 是否已经覆盖完，多余的区间要舍弃。
> - 区间覆盖要记得区间是否已经使用完，最后要判断当前覆盖的最大值是否 >= t。

[45. 跳跃游戏 II](https://leetcode.cn/problems/jump-game-ii/)  

### 区间选点问题

数轴上有 n 个闭区间  [ai,bi]。取尽量少的点，使得覆盖每个区间内至少有一个点。

- 和上面一样先考虑区间包含的情况，如果小区间被满足了，大区间一定被满足，所以在相互包含的情况下，大区间不应该考虑。
- 先按照 b 从小到大排序（b 相同 a 从大到小排序）。
- 取第一个区间的最后一个点，然后 ai 小于等于该点的区间都被跳过 ，直到 ai 大于刚刚选取的点，重复上一步。

#### 例题：[leetcode 452. 用最少数量的箭引爆气球](https://leetcode.cn/problems/minimum-number-of-arrows-to-burst-balloons/)

​    有一些球形气球贴在一堵用 `XY `平面表示的墙面上。墙面上的气球记录在整数数组 `points `，其中`points[i] = [xstart, xend]` 表示水平直径在 `xstart`和 `xend`之间的气球。你不知道气球的确切` y `坐标。一支弓箭可以沿着 `x` 轴从不同点 完全垂直 地射出。在坐标 `x` 处射出一支箭，若有一个气球的直径的开始和结束坐标为 `xstart`，`xend`， 且满足  `xstart ≤ x ≤ xend`，则该气球会被 引爆 。可以射出的弓箭的数量 没有限制 。 弓箭一旦被射出之后，可以无限地前进，返回引爆所有气球所必须射出的 最小 弓箭数 。

```c++
class Solution {
public:
    int findMinArrowShots(vector<vector<int>>& points) {
        sort(points.begin(),points.end(),
            [](const vector<int> &a,const vector<int> &b){
                // 按照 end 从小到大排序，如果 end 相同，start 大的排前面
                //这样区间包含的时候小区间一定排在前面
                return a[1]==b[1]?a[0]>b[0]:a[1]<b[1];
            });
        int n=points.size();
        int ans=1,t=points[0][1];
        for(int i=1;i<n;++i){
            if(points[i][0]>t){
                ans++;
                t=points[i][1];
            }
        }
        return ans;
    }
};
```



### 区间最大重叠数问题

我个人觉得这个应该不算贪心问题，更像一个单纯的模拟题，但是有时候我容易把它跟上面几个区间问题搞混了，因此在这里也记录一下。该问题就是求整个区间内最大的重叠数，打个比方就是一个大盘子里面放了好多饼，这些饼会相互部分重叠，也可能不重叠，求最厚的地方是多少。

#### 例题：[leetcode 253 会议室 II](https://leetcode.cn/problems/meeting-rooms-ii/)

    给你一个会议时间安排的数组 intervals ，每个会议时间都会包括开始和结束的时间 intervals[i] = [starti, endi] ，返回 所需会议室的最小数量。

**思路分析：**最早开始的会议肯定需要 1 个会议室，如果第 2 早开始的会议在前 1 个会议结束前开始，那么会议室 +1，否则使用空闲的。那么后面每个会议开始的时候都可以查看前面是否有空闲的会议室。我们可以使用计数来模拟上面的行为，我们把所有开始和结束的时间排序，碰到开始时间会议室 +1，碰到结束时间会议室 -1（这和后面开始的 +1 抵消，相当于使用空闲）。计数过程中的最大值，就是所需的最少会议室。也可以使用小根堆来模拟，那样模拟行为会更加明显。

```c++
class Solution {
    typedef pair<int,int> pi;
public:
    int minMeetingRooms(vector<vector<int>>& intervals) {
        vector<pi> times;
        for(vector<int>& t : intervals){
            times.push_back({t[0],1});// 1 表示开始
            times.push_back({t[1],0});//如果某个结束时间和开始相同，结束排前面。
        }
        sort(times.begin(),times.end());
        int ans=0,cnt=0;
        for(auto &t : times){
            if(t.second) cnt++;
            else cnt--;
            ans=max(ans,cnt);
        }
        return ans;
    }
};
```



### 其他例题

#### 例题：[POJ 3069 Saruman's Army]()

​    直线上有 n 个点，坐标为 xi，选择若干个点添加标记，每个标记的点的覆盖范围是 R，要求所有点都能被标记点覆盖。求满足上述条件情况下，选择尽量少的点添加标记。

**思路分析：**我们从**最左边的点**开始考虑，这个点向右距离为 R 的范围内必须要有点（包括它自身）带有标记。哪给那个点带标记比较好呢？从贪心的角度考虑，最右的点是最好。因此我们从左边开始，选择其右边 R 范围内最远的点作为标记点，然把标记点右边距离其大于 R 的第一个点，作为新的**最左边的点**，依次类推。 

```c++
# include<cstdio>
# include<string>
# include<algorithm>
using namespace std;

int r,n,x[1010];

// #define LOCAL
int main(){
    #ifdef LOCAL
        freopen("input.txt","r",stdin);
        freopen("output.txt","w",stdout);
    #endif
   while(true){
     scanf("%d %d",&r,&n);
     if(r==-1&&n==-1) break;
     for(int i=0;i<n;++i) scanf("%d",&x[i]);
     sort(x,x+n);
     int ans=0;
     for(int i=0;i<n;++i){
       int s=x[i];
       int j=i;
       while(j+1<n&&s+r>=x[j+1]) ++j;
       int t=x[j];
       while(j+1<n&&t+r>=x[j+1]) ++j;
       i=j;
       ans++;
     }
    printf("%d\n",ans);
  }
  return 0;
}
```



## 字典序问题

#### 例题：note "POJ 3617 Best Cow Line

​    给定一个长度为 N 的字符串 S，要构造一个长度同样为 N 的字符 T，T 初始为空，你可以反复进行以下任意操作：

- 从 S 的头部删除一个字符，放到 T 的尾部。
- 从 S 的尾部删除一个字符，放到 T 的尾部

**目标：**T  **字典序尽量小**。

**限制：**1<=N<=1000

**思路分析：**从字典序的性质来看，无论 T 的末尾有多大，只要前面够小就行，因此我们可以使用下面的贪心算法：我们不断从 S 的开始和末尾取较小的一个。如果它们相等呢？因为我们贪心地想尽量早用到较小的字母，所以我们需要比较下一个，如果还相等，则下下一个，直到有一边较小的，那么就取较小的那一边的第一个。

因此我们可以这样设计算法：

- 我们可以把 S 反转后得到一个新的字符串 S'。
- 比较 S 和 S' 的大小，如果 S 较小，则删除 S 的第一个字母添加到 T 的末尾；反之，则删除 S' 的第一个字母添加到 T 的末尾。
- 依次类推直到 T 的长度等于原来字符串的长度。

```c++
#include <cstdio>
#include <string>
using namespace std;
const int MAX_N=2010;
int N;
char s[MAX_N];

// #define LOCAL
int main(){
    #ifdef LOCAL
        freopen("input.txt","r",stdin);
        freopen("output.txt","w",stdout);
    #endif
  scanf("%d",&N);
  for(int i =0;i<N;++i){
    scanf(" %c",&s[i]);
  }
  int a=0; int b=N-1; int cnt=0;
  while(a<=b){
    bool left=true;
    for(int i=0;i+a<=b-i;++i){
      if(s[a+i]==s[b-i]) continue;
      left=s[a+i]<s[b-i];
      break;
   }
   if(left) putchar(s[a++]);
   else putchar(s[b--]);
   if(++cnt==80){
     putchar('\n');
     cnt=0;
   }
  }
  if(cnt<80) putchar('\n');
  return 0;
}
```



## 哈夫曼编码问题

哈夫曼编码的两个关键：

- 前缀码树：每个编码都是叶子路径编码，只有叶子才是最终的编码。
- 总价值最小的前缀码树：价值最小的两个节点最为兄弟节点，并且是深度最大的节点。

#### 例题：[POJ 3253  **Fence Repair**](http://poj.org/problem?id=3253)

​    有一块木板要切割成 $N$ 块，每块的长度为 $L_i$ ,木板的初始长度为恰好是 $L_i$ 的总和。每次切割的开销是这块木板的长度，求最小的切割开销。
​    例如长度为 21 木板需要切割成 5，8，8，则先把 21 切割成 8 和 13，开销是 21，再把 13 切割成 5 和 8，开销是 13，因此总开销是 34。

**思路分析：**把切割的过程看成一颗二叉树，根节点就是初始木板，需要最终切割的木板就是叶子，每次切割的开销为当前节点的长度，因此总的开销就是各个**叶子节点木板的长度*其深度**的和，因此我们就是要找一个让上面和最小的树，这不就和哈夫曼编码如何构造一颗最优编码树一样么？

```c++
# include<cstdio>
# include<string>
using namespace std;
typedef long long LL;
const int N=20010;
int n,len[N];

LL solve(){
  LL ans=0;
  while(n>1){
    int mi1=0,mi2=1;
    if(len[mi1]>len[mi2]) swap(mi1,mi2);
    for(int i=2;i<n;++i){
      if(len[i]<=len[mi1]){
        mi2=mi1; mi1=i;
      }else if(len[i]<len[mi2]) mi2=i;
    }
   int t=len[mi1] + len[mi2];
   ans+=t;
   if(mi1==n-1) swap(mi1,mi2);
   len[mi1]=t;
   len[mi2]=len[n-1];
   --n;
  }
  return ans;
}


// #define LOCAL
int main(){
    #ifdef LOCAL
        freopen("input.txt","r",stdin);
        freopen("output.txt","w",stdout);
    #endif
   scanf("%d",&n);
   for(int i=0;i<n;++i) scanf("%d",&len[i]);
   LL ans=solve();
   printf("%lld\n",ans);
   return 0;
}

```



## 邻项交换法

#### 例题：[POJ 3045 Cow Acrobats](http://poj.org/problem?id=3045)

​    N 头牛叠罗汉，每头牛都有一个体重值 wi 和一个理论值 si，一头牛叠罗汉时候承受的风险等于其上面所有牛的体重之和（不包括自己）减去自身的力量值。给出每头牛的 wi 和 si，求如何安排叠罗汉的顺序，让所有牛中最大的风险系数最小。

**思路分析：**我们假设存在两头相邻的牛 $cow_i$ 和 $cow_{i+1}$，这两头牛是否交换顺序不会影响它们上面和它们下面的牛的风险系数，它们只会影响它们本身。设 T 是它们上面所有牛的重量，它们不交换顺序和交换顺序的最大风险系数分别是是：

$max(T-s_i,T+w_i-s_{i+1})$ 和 $max(T-s_{i+1},T+w_{i+1}-s_i)$

我们把 T 去掉就有：

$max(-s_i,w_i-s_{i+1})$ 和 $max(-s_{i+1},w_{i+1}-s_i)$

假如是 i 排在前面更优，那么前面的公式就会更小，因此我们可以根据这个来排序，然后上到下计算每头牛的风险系数，最大值就是答案。

```c++
#include <cstdio>
#include <cstring> 
#include <iostream>
#include <algorithm>
#include <set>
using namespace std;
using namespace std;
const int N=50010;
int n;
struct cow{
    int s,w;
}cows[N];
//排序对比
int cmp (const cow &a, const cow& b){
    return max(a.w-b.s,-a.s) < max(b.w-a.s,-b.s);
}

//贪心 邻项交换法
// #define LOCAL
int main(){
    #ifdef LOCAL
        freopen("input.txt","r",stdin);
        freopen("output.txt","w",stdout);
    #endif
    scanf("%d",&n);
    for(int i=0;i<n;++i){
        scanf("%d %d",&cows[i].w,&cows[i].s);
    }
    sort(cows,cows+n,cmp);
    int ans=-1e9,sw=0;
    for(int i=0;i<n;++i){
        int risk=sw-cows[i].s;
        sw+=cows[i].w;
        ans=max(ans,risk);
    }
    printf("%d\n",ans);
    return 0;
}

```



[P1080 NOIP2012 国王游戏](https://oi-wiki.org/basic/greedy/)

## 后悔法

思路就是无论当前是否是最优都接受，然后进行比较，如果选择之后不是最优了，则反悔，舍去掉这个选项，否则，正式接受。如此往复。

#### 例题：[leetcode 871 最低加油次数](https://leetcode.cn/problems/minimum-number-of-refueling-stops/)

​    汽车从起点出发驶向目的地，该目的地位于出发位置东面 target 英里处。沿途有 n 个加油站，$station[i]=\{x,f\}$ 表示第 i 个加油站距离出发点 x 英里，可以加 f 升汽油。汽车最初的油量是 startFuel，每行驶 1 英里消耗 1 升汽油，可以认为油箱的容量是无限的，求到达终点所需加油的最小次数，如果不能到达终点则返回 -1。
??? note "思路分析"
​    由于是求最小的加油次数，因此先考虑不加油，如果能一直开到终点则答案为 0 ，反之如果我们在到达某个加油站前就已经用光了汽油，那么我们可以“后悔”没加油，从已经路过的加油站中选择加油量最大的加，如果还不够则继续选择次大的加，直到总油量能到达下一站。然后我们再按照前面的后悔法一直开，等到没油的时候再后悔。如此往复，直到终点。但是如果中间出现即使把前面路过的加油站的油全加了也无法到达下一站，则返回 -1。

其实这题也是区间覆盖问题，区间左边是加油站，右边是加油站 + 提供的油量，使用最小的区间覆盖开始到目的地。后悔法是一种处理问题的技巧，它有时会用来解决区间覆盖问题，但不是只解决此类问题。

```c++
class Solution {
    typedef long long LL;
public:
    int minRefuelStops(int target, int startFuel, vector<vector<int>>& stations) {
        stations.push_back({target,0});
        int n=stations.size();
        LL s=startFuel;
        priority_queue<int> pq;
        int ans=0;
        for(int i=0;i<n;++i){
            while(s<stations[i][0]&&pq.size()) {
                s+=pq.top();
                pq.pop();
                ans++;
            }
            if(s<stations[i][0]) return -1;
            pq.push(stations[i][1]);
        }
        return ans;
    }
};

```



[USACO09OPEN 工作调度 Work Scheduling](https://oi-wiki.org/basic/greedy/)

[P1209 [USACO1.3]修理牛棚 Barn Repair](https://www.luogu.com.cn/problem/P1209)

[leetcode 630. 课程表 III](https://leetcode.cn/problems/course-schedule-iii/)


## 参考资料

《算法竞赛入门经典 第2版》

《挑战程序设计竞赛》

[《leetcode》](https://leetcode.cn/problemset/all/)

[《oi-wiki》](https://oi-wiki.org/)