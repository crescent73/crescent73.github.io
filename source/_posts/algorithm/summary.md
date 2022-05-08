---
title: 算法总结
date: 2022-05-04 16:16:41
categories:
- algorithm
tags:
- algorithm
- summary
---

# 算法刷题总结
刷了一些算法题，感觉有些题有套路，所以在这里做下笔记，总结一下！  
最近做的周赛比较多，所以，先总结周赛里的一些套路。


## 二分
二分应该算是很简单的题了，一般，我看一个题能不能拿二分做，就看判断条件。如果判断是不是满足条件特别好算，然后有最大最小值，可以直接遍历求解，那就拿二分做。  
个人感觉二分的难点在于边界条件的判定？需要比较细心。  
PS: 也有一下hard题披着马甲，虽然说是二分，但是根本想不到。  
python模板：
``` python
# 初始化左右边界
left, right = min_value, max_value
while left <= right:
    mid = (left + right) // 2 # 计算中间值
    if 判断条件:
        left = mid + 1
    else:
        right = mid - 1
# 后续操作

```
代表题目： 
+ [同时运行 N 台电脑的最长时间](https://leetcode-cn.com/problems/maximum-running-time-of-n-computers/)
+ [完成旅途的最少时间](https://leetcode-cn.com/problems/minimum-time-to-complete-trips/)


## hash表
hash表有个特点，就是判断某个元素是否在表中的事件复杂度是O(1)！  
代表题目： 
+ [统计追加字母可以获得的单词数](https://leetcode-cn.com/problems/count-words-obtained-after-adding-a-letter/)


## 贪心
贪心题个人感觉就是思路题，一般都是先排序再找一个贪心的策略。这种策略需要证明是每一步最优的（当然做题的时候不会证明）。听起来挺简单，但是实际做的时候，感觉很多题贪心的思路很难想。  
代表题目：
+ [构造限制重复的字符串](https://leetcode-cn.com/problems/construct-string-with-repeat-limit/)
+ [拿出最少数目的魔法豆](https://leetcode-cn.com/problems/removing-minimum-number-of-magic-beans/)
+ [使数组变成交替数组的最少操作数](https://leetcode-cn.com/problems/minimum-operations-to-make-the-array-alternating/)
+ [得到目标值的最少行动次数](https://leetcode-cn.com/problems/minimum-moves-to-reach-target-score/)


## 二叉树
二叉树感觉这儿有很多trick！  
首先是建二叉树，一种建树的方式是，拿一个set存储所有的树节点，然后建节点之间关系的时候，直接从set里拿。这样，一次遍历以后，树的结构已经建好了，然后需要二次遍历找到树的根节点（一般就找没有父节点的点）。  
代表题目：
+ [根据描述创建二叉树](https://leetcode-cn.com/problems/create-binary-tree-from-descriptions/)

二叉树的遍历：一般是dfs和bfs。目前刷的题比较少，等多刷几道题再补上！


## 滑动窗口
滑动窗口的题感觉，比较巧妙？前提是，你要能想到，能通过窗口，判断数据是不是满足题目条件！  
有个trick，如果每次移动窗口，都对窗口求和或者其他操作的话，肯定超时。所以，需要手动维护窗口的状态。比如窗口右移一位，对于窗口中的元素来说，是左边少了一个数，右边多了一个数，所以，可以多加两个变量记录，直接加减变量值。  
代表题目：
+ [最少交换次数来组合所有的 1 II](https://leetcode-cn.com/problems/minimum-swaps-to-group-all-1s-together-ii/)


## dp
dp感觉是个重重点，但是一直没怎么弄懂。目前的感觉就是，走到后面的步的时候，需要前面步的状态，所以，需要考虑好转移方程（后面的步怎么从前面的步转移过来），还有初始化条件。
代表题目：
+ [解决智力问题](https://leetcode-cn.com/problems/solving-questions-with-brainpower/)


## 状态压缩
状态压缩感觉也是个重点，思路是穷举，一般用在数据量不大的情况下。具体来说就是，穷举每一种可能的情况，用01状态表示，然后对于每一种情况，判断是否满足条件，如果满足条件，进行进一步的判断。  
这个算法一开始对我来说，最难的是怎么构造出2**n种状态表示方法，但是这块，感觉是能拿模板直接做的。  
python模板：
``` python
# n表示有n个状态
for i in range(2**n):
    tmp = i
    bit_tmp = []
    while tmp > 0:
        bit_tmp.append(tmp&1)# 1 的二进制是 0000001，所以 &1 就是取最后一位
        tmp>>=1 # >> 位运算符，原来所有位右移，最低位丢弃，最高位使用符号位填充
    bit_tmp.extend([0]*(n-len(bit_tmp))) # 空位补0，这里应该从右往左看
    # 正式判断
```
代表题目：
+ [射箭比赛中的最大得分](https://leetcode-cn.com/problems/maximum-points-in-an-archery-competition/)
