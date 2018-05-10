---
title: leetcode经典题目
date: 2018/3/16 08:28:25
category:
- 算法和数据结构
- 算法
tag:
- leetcode
comments: true  
---

# 数学

## 数组

### Leetcode 370. Range Addition

给定一个全为0的array的长度，以及一系列操作，每个操作会指明要操作的开始索引和结束索引，以及要加上的值，求出所给操作执行完之后的array情况，具体样例如下：

```
length = 5,
updates = 
[
[1,  3,  2],
[2,  4,  3],
[0,  2, -2]
]
return [-2, 0, 3, 5, 3]
```

思路：只需要记录start和end即可，对每个operation中的start位置的值加上val，end位置值减去val；第二：最后求解整个list的值的时候，是从前到后逐个相加得到的

``` python
class Solution:
    def getModifiedArray(self, length, updates):
        # Write your code here
        result = [0 for idx in range(length)]
        for item in updates:
            result[item[0]] += item[2]
            if item[1]+1 < length:
                result[item[1]+1] -= item[2]
        tmp = 0
        for idx in range(length):
            tmp += result[idx]
            result[idx] = tmp
        
        return result

```


>
>
> 扩展联想：
>
> 重点在于复用前一个值，如何复用？
>

其他题目：

 - 求数组中任意一个连续范围的数字和

 - 二维如何计算，

   - 方法一：扩展1D到2D，可以设置4个角的数据，然后自顶向下，自左向右累加即可，注意计算行时先计算本行，再加上一行

     4*4矩阵，updates=[[1,1],[3,3],1], [[2,2],[3,3],1]如下：

     0 0 0 0        	 1 0 0 -1       	 1  0 0 -1             1 1 1 0            
     0 0 0 0    	 0 0 0 0    	  0  2 0 -2     	    1 3 3 0                   
     0 0 0 0    ==>   0 0 0 0    ==>  0  0 0 0      ==> 1 3 3 0                   
     0 0 0 0             -1 0 0 1           -1 -2 0 3             0 0 0 0  

     ``` python
         def getModifiedArray(self, m, n, updates):
             # Write your code here
             result = [[0 for idx in range(m)] for idx in range(n)]
             for item in updates:
                 result[item[0][0]][item[0][1]] += item[2]
                 if item[1][0] + 1 < m:
                     result[item[1][0] + 1][item[0][1]] -= item[2]
                 if item[1][1] + 1 < n:
                     result[item[0][0]][item[1][1] + 1] -= item[2]
                 if item[1][0] + 1 < m and item[1][1] + 1 < n:
                     result[item[1][0] + 1][item[1][1] + 1] += item[2]
             for i in range(m):
                 tmp = 0
                 for j in range(n):
                     tmp += result[i][j]
                     result[i][j] = tmp
                 if i > 0:
                     for j in range(n):
                         result[i][j] += result[i - 1][j]
             return result
     ```

   - 方法二：

     一行一行处理，简单方便

     ``` python
         def getModifiedArray2(self, m, n, updates):
             # Write your code here
             result = []
             for i in range(m):
                 up2 = []
                 for item in updates:
                     if i >= item[0][0] and i <= item[1][0]:
                         up2.append([item[0][1], item[1][1], item[2]])
                 if len(up2)>0:
                     result.append(self.getModifiedArray(n, up2))
                 else:
                     result.append([0]*n)
             return result
     ```

     ​



### Leetcode 370. 

