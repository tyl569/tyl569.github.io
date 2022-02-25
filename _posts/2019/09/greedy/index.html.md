---
layout: post
title: 贪心算法
date: 2019-09-30
tags: ["PHP","算法","算法","贪心算法"]
---

### 描述

所谓的贪心算法，是在对问题求解的时候，总是做出在当前看来是最好的选择。也就是说，<font color="red">不从整体最优上进行考虑，所做的仅仅是在某种意义上的局部最优解。</font>

贪心算法没有固定的算法框架，算法设计的关键是贪心策略的选择。贪心算法不是对所有问题都能得到最优解，选择的贪心策略必须具备无后效性，即其他过程，不会影响以前的状态，完全是局部的。

### 基本思路

*   建立数学模型来描述问题
*   把求解的问题分成若干个子问题
*   对每一个子问题求解，得到子问题的局部最优解
*   把子问题的局部最优解合成原来界问题的一个解

### 实例

#### 两地调度（LeetCode: 1029）

公司计划面试 2N 人。第 i 人飞往 A 市的费用为 costs[i][0]，飞往 B 市的费用为 costs[i][1]。

返回将每个人都飞到某座城市的最低费用，要求每个城市都有 N 人抵达

示例：

输入：[[10,20],[30,200],[400,50],[30,20]]
输出：110
解释：
第一个人去 A 市，费用为 10。
第二个人去 A 市，费用为 30。
第三个人去 B 市，费用为 50。
第四个人去 B 市，费用为 20。

最低总费用为 10 + 30 + 50 + 20 = 110，每个城市都有一半的人在面试。

#### 算法思路

假设，第一个人去A的成本是a1,去B的成本是b1,那么他去A或者去B的成本差距就是a1-b1，这个成本的差距可证可负，如果成本差距是正数，那么说明这个人去A的价格要比去B的价格贵。

同理，第二个人去A的成本是a2，去B的成本B2，那么他去A和B的成本差距就是a2-b2

...

**<font color="red">那么所有人去A和B的成本差的总和就是(a1-b1)+(a2-b2)....+(a2n-b2n)，根据题目的要求，如果安排的成本是最小，那么把成本差从小到大进行排序，那么直接让前n个人去A，另外n个人去B，就能得到满足题干的解了。</font>**

<table>
<thead>
<tr>
<th>员工</th>
<th>A</th>
<th>B</th>
<th>成本差</th>
</tr>
</thead>
<tbody>
<tr>
<td>1</td>
<td>10</td>
<td>b1</td>
<td>a1-b1</td>
</tr>
<tr>
<td>2</td>
<td>a2</td>
<td>b2</td>
<td>a2-b2</td>
</tr>
<tr>
<td>3</td>
<td>a3</td>
<td>b3</td>
<td>a3-b3</td>
</tr>
<tr>
<td>4</td>
<td>a4</td>
<td>b4</td>
<td>a4-b4</td>
</tr>
<tr>
<td>5</td>
<td>a5</td>
<td>b5</td>
<td>a5-b5</td>
</tr>
<tr>
<td>2n</td>
<td>a2n</td>
<td>b2n</td>
<td>a2n-b2n</td>
</tr>
</tbody>
</table>

将题干引入模型：

<table>
<thead>
<tr>
<th>员工</th>
<th>A</th>
<th>B</th>
<th>成本差</th>
</tr>
</thead>
<tbody>
<tr>
<td>1</td>
<td>10</td>
<td>20</td>
<td>-10</td>
</tr>
<tr>
<td>2</td>
<td>30</td>
<td>200</td>
<td>-170</td>
</tr>
<tr>
<td>3</td>
<td>400</td>
<td>50</td>
<td>350</td>
</tr>
<tr>
<td>4</td>
<td>30</td>
<td>20</td>
<td>10</td>
</tr>
</tbody>
</table>

成本差分别是
-10，-170，350，10
排序之后的结果：
-170，-10，10，350
那么直接让用户2和用户1去A，让用户4和用户3去B。
最低费用是 30+10+50+20=110 

#### 算法实现

    class Solution {

        /**
         * @param Integer[][] $costs
         * @return Integer
         */
        function twoCitySchedCost($costs) {
            $additional = [];
            $diff = [];
            foreach ($costs as &$cost) {
                $cost['diff'] = $cost[0]-$cost[1];
                $diff[] = $cost[0]-$cost[1];
                unset ($cost);
            }
            array_multisort($diff, SORT_ASC,$costs );
            $total = 0;
            $n = count($costs) / 2;
            for ($i=0; $i< $n; $i++) {
                $total += $costs[$i][0] + $costs[$i+$n][1];
            }
            return $total;
        }
    }