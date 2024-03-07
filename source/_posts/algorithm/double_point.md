---
title: 双指针（一）
date: 2024-02-05 10:57:11
tags:
categories: 算法
mathjax: true
thumbnail:
---

双指针是一种简单而又灵活的技巧和思想，单独使用可以轻松解决一些特定问题，和其他算法结合也能发挥多样的用处。
双指针顾名思义，就是同时使用两个指针，在序列、链表结构上指向的是位置，在树、图结构中指向的是节点，通过或同向移动，或相向移动来维护、统计信息。
双指针本身的思想并不难，但是并不代表它很容易掌握，双指针的问题重点不在于你如何使用它，而是如何能“嗅到”该用它了。能用双指针解决的问题，数据往往都带有一些的特性质，比如单调性或者区间收敛，对这些数据的敏感度才是掌握双指针的关键，但不是每个人都是数学大神，对于这种情况“记录和总结”应该算是比较好的抢救方法了。

<!--more-->

## 滑动窗口
滑动窗口是双指针最经典的用法之一，通常对数组保存一对下标，然后根据实际情况交替推进两个端点直到得出答案。
### 例题：[POJ 3061 subsequence](http://poj.org/problem?id=3061)
给定长度为 n 的数列 `a_0,a_1,a_2...a_n-1` 以及整数 `S`，求出总和不少于 `S` 的连续子数组的长度的最小值，如果不存在则输出 `0`。
**解题思路：**
子区间求和，第一时间就是想到前缀和 $sum_{ij}=sum_j-sum_i$，因此对于每个 $sum_i$ 我们只需要求出满足 $sum_i+S$ 的最小的 $sum_j$ 就行了。由于 前缀和具有单调性质，因此可以考虑使用二分来解决，这样时间复杂度是 $O(nlogn)$。

我们先来考虑一下最暴力的解法：
```c++
int ans=0;
for(int i=0;i<n;++i){
    for(int j=i+1;j<=n;++j){
        if(sum[j]>=sum[i]+S){
            ans=max(ans,j-i);
            break;
        }
    }
}
```
这是一个 $O(n^{2})$ 的方法，我们观察上面的代码可以发现 `S` 是不变的，而 $sum[i]$ 是单调不减的，因此如果一个区间 `[j1,j2]` 在 `i` 的情况下不满足要求，那么在 `i + 1` 下也肯定不满足要求，因此我们可考虑使用两个指针在一个 for 循环上来维护 `i` 和 `j` 的位置，因为对于被 `i` 淘汰的 `j` 来说，`i + 1` 是不需要再回去遍历它们的。

```c++
#include <cstdio>
#include <algorithm>
using namespace std;
const int N=100010;
int n,s,a[N];

int main(){
    int T;
    for(scanf("%d",&T);T>0;--T){
        scanf("%d %d",&n,&s);
        int ans=n+1;       
        for(int i=0;i<n;++i) scanf("%d",&a[i]);
        for(int l=0,r=0,t=0;;){
            while(r<n&&t<s) t+=a[r++];
            if(t<s)break;
            ans=min(ans,r-l);
            t-=a[l++];
        }
        if(ans>n) ans=0;
        printf("%d\n",ans);
    }
}
```

### 例题：[leetcode 3. 无重复字符的最长子串](https://leetcode.cn/problems/longest-substring-without-repeating-characters/)
给定一个字符串 s ，请你找出其中不含有重复字符的 最长子串 的长度。
**思路分析：**
这是一种非常常见的滑动窗口面试题，通过统计区间的元素的个数来维持区间的某种性质，让其交替前进。

我们还是不妨先来考虑暴力枚举，我们枚举每一个结束字符的所有子区间，然后判断是否符合不包含重复字符的要求，然后取最大长度的那一个区间。
```c++
for(int j=0;j<s.size();++j){
    memset(cnt,0,sizefo(cnt));
    for(int i=j;i>=0;--i){
        if(cnt[s[i]]++==1) tot++;//如果当前字符之前已经存在第一个，那么重复字符个数 + 1
        if(tot==0){//表示没有重复字符
            ans=max(ans,j-i+1);
        }
    }
}
```
上面算法的时间复杂度是 $O(n^{2})$，我们可以发现有很多重复的统计，对于字符串 "abcabcbb"，当我们统计以第一个 c 为结尾的子区间的时候，会统计abc 的个数，当我们统计以第二个a为结尾的子区间的时候，abc 的个数还会被重新统计，但是其实我们可以重复利用前面的统计。我们可以使用两个指针 i 和 j 来维护一个区间，让区间在每次迭代后都满足重复个数等于 0 条件即可。

```c++
class Solution {
public:
    int lengthOfLongestSubstring(string s) {
        vector<int> cnt(256,0);
        int l,r,ans=0,tot=0;
        for(l=0,r=0;r<s.size();++r){
            if(cnt[s[r]]++==1) tot++;
            while(tot){
                if(--cnt[s[l++]]==1) tot--;
            }
            ans=max(ans,r-l+1);
        }
        return ans;
    }
};
```
[leetcode 567. 字符串的排列](https://leetcode.cn/problems/permutation-in-string/)

[leetcode 30. 串联所有单词的子串](https://leetcode.cn/problems/substring-with-concatenation-of-all-words/)

### 例题：[leetcode 220. 存在重复元素 III](https://leetcode.cn/problems/contains-duplicate-iii/)
给你一个整数数组 $nums$ 和两个整数 $k$ 和 $t$ 。请你判断是否存在 两个不同下标 $i$ 和 $j$，使得 $abs(nums[i] - nums[j]) <= t$ ，同时又满足 $abs(i - j) <= k$。

**思路分析：**

这种类型的滑动窗口长度是固定，所以推进是比较简单，但是往往会结合一些数据结构，比如堆、单调队列或者有序集合等。典型问题是的动态查询每个区间的第 n 大值，或者是否存在满足条件的元素。

就这题而言，我们可以枚举每个 $nums[j]$，因此问题就变成了在下标区间 $[j-k,j]$ 范围内是否存在一个数 $x$，其满足 $nums[j]-t<=x<=nums[j]+t$，可以使用一个有序集合（比如平衡树）来维护当前窗口除 $nums[j]$ 以外的所有数，我们通过查找第一个大于等于 $nums[j]-t$ 的数 $x$，判断 $x$ 是否满足上面的条件即可。

```c++
class Solution {
public:
    bool containsNearbyAlmostDuplicate(vector<int>& nums, int k, int t) {
        set<long long> st;
        int n=nums.size(),l,r;
        for(l=0,r=0;r<n;++r){
            auto it=st.lower_bound((long long)nums[r]-t);
            if(it!=st.end()&&*it<=(long long)nums[r]+t) return true;
            st.insert(nums[r]);
            if(r>=k) st.erase(nums[l++]);
        }
        return false;
    }
};
```

### 例题：[leetcode 480. 滑动窗口中位数](https://leetcode.cn/problems/sliding-window-median/)
给你一个数组 nums，有一个长度为 k 的窗口从最左端滑动到最右端。窗口中有 k 个数，每次窗口向右移动 1 位。你的任务是找出每次窗口移动后得到的新窗口中元素的中位数（如果 k 为偶数，中位数为中间两个数的平均值），并输出由它们组成的数组。

**思路分析：**

和上面的题目一样，都是长度是固定的滑动窗口问题。我们可以通过两个堆来维护第 k/2 大值，左边 [0,k/2) 为大顶堆，右边 [k/2,k] 为小顶堆，因此我们只需要通过两个堆的堆顶即可得到中位数。

1. 首先初始化一个长度为 k 的窗口，并且获取第一个中位数。
2. 窗口往右边滑动一个数，就把该数压入堆顶，如果右边的堆为空，或者该数比右边堆顶要大，则压入右边，否则左边。
3. 窗口左边出一个数，如果该数大于等于右边堆顶，则从右边堆顶删除，否则从左边删除。
4. 维护两边堆的个数，如果左边个数小于 k/2，则把右边的堆顶弹出压入左边，如果左边个数大于 k/2，则把左边的堆顶弹出压入右边。
5. 获取中位数
6. 循环第 2 - 5 步。

由于需要删除堆元素，因此需要自定义实现一个懒删除堆。

```c++
typedef long long LL;
class PQ {
    private: 
        priority_queue<LL> pq;
        unordered_map<LL,int> delayed;
        int delCnt=0;
    private:
        void prune(){
            while(pq.size()&&delayed.count(pq.top())){
                int c=--delayed[pq.top()];
                if(c==0) delayed.erase(pq.top());
                pq.pop();
                delCnt--;
            }
        }
    public:
        int size(){
            return pq.size() - delCnt;
        }

        LL top(){
            prune();
            LL res=pq.top();
            return res;
        }

        LL pop(){
            LL res=top();
            pq.pop();
            return res;
        }

        void push(LL x){
            pq.push(x);
        }

        void del(LL x){
            delayed[x]++;
            delCnt++;
        }
};
class Solution {
public:
    PQ L; //大顶堆
    PQ R; //通过存入相反数来模拟小顶堆
    double getMedian(int k){
        if(k&1) return -R.top();
        return ((double)L.top()-R.top())/2;
    }
    vector<double> medianSlidingWindow(vector<int>& nums, int k) {
        int l,r,n=nums.size();
        vector<double> ans(n-k+1,0);
        //初始化第一个长度为 k 的窗口
        for(l=0,r=0;r<k;++r) R.push(-(LL)nums[r]);
        for(int i=0;i<k/2;++i) L.push(-R.pop());
        int cur=0;
        ans[cur++]=getMedian(k);
        //不断向右滑动
        for(;r<n;++r,++l){
            int a=nums[l], b=nums[r];
            if(R.size()==0 ||b>=-R.top()) R.push(-(LL)b);
            else L.push(b);
            if(a>=-R.top()) R.del(-(LL)a);
            else L.del(a);
            while(L.size()<k/2) L.push(-R.pop());
            while(L.size()>k/2) R.push(-L.pop());
            ans[cur++]=getMedian(k);
        }
        return ans;
    }
};
```

### 例题：[leetcode 424. 替换后的最长重复字符](https://leetcode.cn/problems/longest-repeating-character-replacement/)
你一个字符串 s 和一个整数 k 。你可以选择字符串中的任一字符，并将其更改为任何其他大写英文字符。该操作最多可执行 k 次。在执行上述操作后，返回包含相同字母的最长子字符串的长度。

**思路分析：**

对于区间 $[l,r]$ 需要满足 $r-l+1-cnt<=k$，其中 $cnt$ 是不需要改变的字母数量，由于 $k$ 是常量，因此 $cnt$ 越大，$r-l+1$ 的值也就可以越大。因此在遍历的过程中我们只需要关注最大的 $max(cnt)$ 就行。当 $r$ 向右移动变成 $r+1$后，如果 $max(cnt)$ 不变，那么区间会变为 $[l+1,r+1]$，如果 $max(cnt)$ 增大了，那一次只会增大 $1$，这时候区间变为  $[l,r+1]$，最后 $r$ 滑到最右边的时候，最后的区间长度一定等于最优的长度。

>如果题目要求最长字串的内容，那么不能取最后的区间字串，需要取最大 cnt 的第一次出现的区间字串。

```c++
class Solution {
public:
    int characterReplacement(string s, int k) {
        vector<int> cnt(256,0);
        int l,r,mx=0,n=s.size();
        for(l=0,r=0;r<n;r++){
            mx=max(mx,++cnt[s[r]]); // r 向右移动一个字符
            if(r-l+1-mx>k) --cnt[s[l++]];//如果 mx 没有改变 l++
        }
        return r-l;
    }
};
```

[leetcode 2024. 考试的最大困扰度](https://leetcode.cn/problems/maximize-the-confusion-of-an-exam/)

### 例题：[leetcode 992. K 个不同整数的子数组](https://leetcode.cn/problems/subarrays-with-k-different-integers/)
给定一个正整数数组 $nums$ 和一个整数 $k$ ，返回 $nums$ 中`「好子数组」` 的数目。如果 $nums$ 的某个子数组中不同整数的个数恰好为 $k$，则称 $nums$ 的这个子数组为 `「好子数组 」`。

**参考思路：**

我们可以枚举每一个子数组的最后一个元素，下标设为 $r$，同时设$cnt(l,r)$ 为子数组$[l,r]$的不同整数的数目，对于任意一个 $r$，如果存在一个区间 $[l1,l2]$ 满足 $cnt(l,r)==k,(l1<=l<=l2)$，那么一定有 $cnt(l,r) >k,(l< l1)$ 和 $cnt(l,r) < k,(l>l2)$，因此满足 r 的所有左端点都在区间 [l1,l2] 这连续区间内，同时 [l1,l2] 的所有点也都可以作为 r 的左端点。但是那样我们需要求出每个 r 的 [l1,l2]，时间复杂度是 $O(n^{2})$。

由于我们只需要求子数组的个数，所以我们可以维护两个区间  $[l1,r]$ 和 $[l2+1,r]$，那么对于右端点 $r$ 来说满足要求的左端点个数就是 $l2+1-l1$。$l1$ 是第一个满足 $cnt(l1,r)==k$ 的左端点，而 $l2+1$，是第一个满足 $cnt(l2+1,r) < k$ 的左端点。

```c++
class Solution {
public:
    int subarraysWithKDistinct(vector<int>& nums, int k) {
        int n=nums.size();
        map<int,int> cnt1,cnt2;
        int l1=0,l2=0,r=0,t1=0,t2=0,ans=0,a=0;
        for(;r<n;++r){
            if(++cnt1[nums[r]]==1) t1++;
            if(++cnt2[nums[r]]==1) t2++;
            while(t1>k) {
                if(--cnt1[nums[l1++]]==0) t1--;
            }
            while(t2>k-1){
                if(--cnt2[nums[l2++]]==0) t2--;
            }
            ans+=l2-l1;
        }
        return ans;
    }
};
```

### 例题：[leetcode 167. 两数之和 II - 输入有序数组](https://leetcode.cn/problems/two-sum-ii-input-array-is-sorted/)
给你一个下标从 1 开始的非递减整数数组 numbers ，请你从数组中找出满足相加之和等于目标数 target 的两个数。你可以假设每个输入只对应一个答案。
```c++
class Solution {
public:
    vector<int> twoSum(vector<int>& numbers, int target) {
        int i,j,n=numbers.size();
        vector<int> ans;
        for(i=0,j=n-1;i<j;){
            if(numbers[i]+numbers[j]==target) {
                ans.push_back(i+1);
                ans.push_back(j+1);
                break;
            }
            if(i<j&&numbers[i]+numbers[j]>target) --j;
            if(i<j&&numbers[i]+numbers[j]<target) ++i;
        }
        return ans;
    }
};
```

### 例题：[leetcode 11. 盛最多水的容器](https://leetcode.cn/problems/container-with-most-water/)
给定一个长度为 n 的整数数组 height 。有 n 条垂线，第 i 条线的两个端点是 (i, 0) 和 (i, height[i]) 。找出其中的两条线，使得它们与 x 轴共同构成的容器可以容纳最多的水。返回容器可以储存的最大水量。

```c++
class Solution {
public:
    int maxArea(vector<int>& height) {
        int n=height.size();
        int ans=0;
        for(int i=0,j=n-1;i<j;){
            ans=max(ans,(j-i)*min(height[i],height[j]));
            if(height[i]<height[j]) ++i;
            else --j;
        }
        return ans;
    }
};
```

## 参考资料

《挑战程序设计竞赛》

《算法竞赛入门经典》

[《leetcode》](https://leetcode.cn/)

[《oi-wiki》](https://oi-wiki.org/)




