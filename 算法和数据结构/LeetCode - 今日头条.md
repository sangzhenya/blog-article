---
title: "LeetCode - 今日头条"
tags: ["算法"]
categories: ["算法"]
date: "2019-01-13T09:00:00+08:00"
---



[toc]

### 字符串（7）

#### 无重复字符的最长子串

给定一个字符串，请你找出其中不含有重复字符的 **最长子串** 的长度。

**示例 1:**

```
输入: "abcabcbb"
输出: 3 
解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。
```

**示例 2:**

```
输入: "bbbbb"
输出: 1
解释: 因为无重复字符的最长子串是 "b"，所以其长度为 1。
```

**示例 3:**

```
输入: "pwwkew"
输出: 3
解释: 因为无重复字符的最长子串是 "wke"，所以其长度为 3。
     请注意，你的答案必须是 子串 的长度，"pwke" 是一个子序列，不是子串。
```

---

```java
class Solution {
    public int lengthOfLongestSubstring(String s) {
        int[] imap = new int[128];
        int res = 0;
        int lastIndex = 0;
        for (int i = 0; i < s.length(); i++) {
            // 上次出现的位置，记录最后一次出现重复的值的位置
            lastIndex = Math.max(imap[s.charAt(i)], lastIndex);
            // 更新结果
            res = Math.max(i - lastIndex + 1, res);
            // 更新上次出现的位置
            imap[s.charAt(i)] = i + 1;
        }
        return res;
    }
}
```

#### 最长公共前缀

编写一个函数来查找字符串数组中的最长公共前缀。

如果不存在公共前缀，返回空字符串 `""`。

**示例 1:**

```
输入: ["flower","flow","flight"]
输出: "fl"
```

**示例 2:**

```
输入: ["dog","racecar","car"]
输出: ""
解释: 输入不存在公共前缀。
```

**说明:**

所有输入只包含小写字母 `a-z` 。

---

```java
class Solution {
    public String longestCommonPrefix(String[] strs) {
        if (strs.length == 0) {
            return "";
        }
        String prefix = strs[0];
        for (int i = 1; i < strs.length; i++) {
            while(strs[i].indexOf(prefix) != 0) {
                prefix = prefix.substring(0, prefix.length() - 1);
                if (prefix.equals("")) {
                    return "";
                }
            }
        }
        return prefix;
    }
}
```

#### 字符串的排列

给定两个字符串 **s1** 和 **s2**，写一个函数来判断 **s2** 是否包含 **s1** 的排列。

换句话说，第一个字符串的排列之一是第二个字符串的子串。

**示例1:**

```
输入: s1 = "ab" s2 = "eidbaooo"
输出: True
解释: s2 包含 s1 的排列之一 ("ba").
```

 

**示例2:**

```
输入: s1= "ab" s2 = "eidboaoo"
输出: False
```

 

**注意：**

1. 输入的字符串只包含小写字母
2. 两个字符串的长度都在 [1, 10,000] 之间

---

```java
class Solution {
    // 主要思路是使用数组记录 string 中 char 的元素个数是否相等
    public boolean checkInclusion(String s1, String s2) {
        char[] a1 = new char[26];
        // 收集 s1 中的 char 
        for (char c : s1.toCharArray()) {
            a1[c - 'a'] += 1;
        }
        char[] a2 = new char[26];
        int l1 = s1.length();
        // 收集 s2 中的 char 保证仅收集和 s1 长度相等的部分，且同时判断两个数组是否相等
        for (int i = 0; i < s2.length(); i++) {
            if (i - l1 >= 0) {
                a2[s2.charAt(i - l1) - 'a'] -= 1;
            }
            a2[s2.charAt(i) - 'a'] += 1;
            if (Arrays.equals(a1, a2)) {
                return true;
            }
        }
        return false;
    }
}
```

#### 字符串相乘

给定两个以字符串形式表示的非负整数 `num1` 和 `num2`，返回 `num1` 和 `num2` 的乘积，它们的乘积也表示为字符串形式。

**示例 1:**

```
输入: num1 = "2", num2 = "3"
输出: "6"
```

**示例 2:**

```
输入: num1 = "123", num2 = "456"
输出: "56088"
```

**说明：**

1. `num1` 和 `num2` 的长度小于110。
2. `num1` 和 `num2` 只包含数字 `0-9`。
3. `num1` 和 `num2` 均不以零开头，除非是数字 0 本身。
4. **不能使用任何标准库的大数类型（比如 BigInteger）**或**直接将输入转换为整数来处理**。

```java
class Solution {
    public String multiply(String num1, String num2) {
        if (num1.equals("0") || num2.equals("0")) {
            return "0";
        }
        int l1 = num1.length();
        int l2 = num2.length();
        int[] res = new int[l1 + l2];
        for (int i = l1 - 1; i >= 0; i--) {
            int v1 = num1.charAt(i) - '0';
            for (int j = l2 - 1; j >= 0; j--) {
                int v2 = num2.charAt(j) - '0';
                // 计算当前两数相乘 加上 上一次两数相乘的进位数（当前位的数字）
                int sum = v1 * v2 + res[i + j + 1];
                // 结果取余作为当前位的数字
                res[i + j + 1] = sum % 10;
                // 将进位结果放到下一位上
                res[i + j] += sum / 10;
            }
        }
        
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < res.length; i++) {
            if (i == 0 && res[i] == 0) {
                continue;
            }
            sb.append(res[i]);
        }
        return sb.toString();
    }
}
```

#### 翻转字符串里的单词

给定一个字符串，逐个翻转字符串中的每个单词。

 

**示例 1：**

```
输入: "the sky is blue"
输出: "blue is sky the"
```

**示例 2：**

```
输入: "  hello world!  "
输出: "world! hello"
解释: 输入字符串可以在前面或者后面包含多余的空格，但是反转后的字符不能包括。
```

**示例 3：**

```
输入: "a good   example"
输出: "example good a"
解释: 如果两个单词间有多余的空格，将反转后单词间的空格减少到只含一个。
```

 

**说明：**

- 无空格字符构成一个单词。
- 输入字符串可以在前面或者后面包含多余的空格，但是反转后的字符不能包括。
- 如果两个单词间有多余的空格，将反转后单词间的空格减少到只含一个。

```java
class Solution {
    public String reverseWords(String s) {
        String[] raws = s.split(" ");
        StringBuilder res = new StringBuilder();
        for (int i = raws.length - 1; i >= 0; i--) {
            String ts = raws[i].trim();
            if (!ts.equals("")) {
                if (res.length() != 0) {
                    res.append(" ");
                }
                res.append(ts);
            }
        }
        return res.toString();
    }
}
```

#### 简化路径

以 Unix 风格给出一个文件的**绝对路径**，你需要简化它。或者换句话说，将其转换为规范路径。

在 Unix 风格的文件系统中，一个点（`.`）表示当前目录本身；此外，两个点 （`..`） 表示将目录切换到上一级（指向父目录）；两者都可以是复杂相对路径的组成部分。更多信息请参阅：[Linux / Unix中的绝对路径 vs 相对路径](https://blog.csdn.net/u011327334/article/details/50355600)

请注意，返回的规范路径必须始终以斜杠 `/` 开头，并且两个目录名之间必须只有一个斜杠 `/`。最后一个目录名（如果存在）**不能**以 `/` 结尾。此外，规范路径必须是表示绝对路径的**最短**字符串。

 

**示例 1：**

```
输入："/home/"
输出："/home"
解释：注意，最后一个目录名后面没有斜杠。
```

**示例 2：**

```
输入："/../"
输出："/"
解释：从根目录向上一级是不可行的，因为根是你可以到达的最高级。
```

**示例 3：**

```
输入："/home//foo/"
输出："/home/foo"
解释：在规范路径中，多个连续斜杠需要用一个斜杠替换。
```

**示例 4：**

```
输入："/a/./b/../../c/"
输出："/c"
```

**示例 5：**

```
输入："/a/../../b/../c//.//"
输出："/c"
```

**示例 6：**

```
输入："/a//b////c/d//././/.."
输出："/a/b/c"
```

```java
class Solution {
    public String simplifyPath(String path) {
        Stack<String> stack = new Stack<>();
        String[] raws = path.split("/");
        for (String rs : raws) {
            if (rs.equals("..")) {
                if (!stack.isEmpty()) {
                    stack.pop();
                }
            } else if (!rs.equals(".") && !rs.equals("")) {
                stack.push(rs);
            }
        }
        if (stack.isEmpty()) {
            return "/";
        }
        StringBuilder res = new StringBuilder();
        for (int i = 0; i < stack.size(); i++) {
            res.append("/").append(stack.get(i));
        }
        return res.toString();
    }
}
```



#### 复原IP地址

给定一个只包含数字的字符串，复原它并返回所有可能的 IP 地址格式。

有效的 IP 地址正好由四个整数（每个整数位于 0 到 255 之间组成），整数之间用 `'.' `分隔。

 

**示例:**

```
输入: "25525511135"
输出: ["255.255.11.135", "255.255.111.35"]
```

```java
class Solution {
    public List<String> restoreIpAddresses(String s) {
        List<String> res = new ArrayList<>();
        bcip(res, new StringBuilder(), s, 0);
        return res;
    }
    
    private void bcip(List<String> res, StringBuilder current, String s, int k) {
        // 最长只能有 4 段
        if (k == 4 || s.length() == 0) {
            if (k == 4 && s.length() == 0) {
                res.add(current.toString());
            }
            return;
        }
        for (int i = 0; i <= 2 && i < s.length(); i++) {
            // 不能以 0 开头
            if (i != 0 && s.charAt(0) == '0') {
                continue;
            }
            // 获取当前段，parse 成数字应该小于等于 255
            String part = s.substring(0, i + 1);
            if (Integer.parseInt(part) <= 255) {
                // 如果不是第一个则需要添加 》
                if (current.length() != 0) {
                    part = "." + part;
                }
                // 将当前段添加到候选中
                current.append(part);
                // 递归添加
                bcip(res, current, s.substring(i + 1), k + 1);
                // 删除当前段继续执行
                current.delete(current.length() - part.length(), current.length());
            }
        }
    }
}
```



### 数组与排序（10）

#### 三数之和

给你一个包含 *n* 个整数的数组 `nums`，判断 `nums` 中是否存在三个元素 *a，b，c ，*使得 *a + b + c =* 0 ？请你找出所有满足条件且不重复的三元组。

**注意：**答案中不可以包含重复的三元组。

 

**示例：**

```
给定数组 nums = [-1, 0, 1, 2, -1, -4]，

满足要求的三元组集合为：
[
  [-1, 0, 1],
  [-1, -1, 2]
]
```

```java
class Solution {
    public List<List<Integer>> threeSum(int[] nums) {
        List<List<Integer>> result = new ArrayList<>();
        // 去重遍历
        Arrays.sort(nums);
        // 边界值
        if (nums == null || nums.length == 0 || nums[0] > 0|| nums[nums.length - 1] < 0) {
            return result;
        }
        for (int i = 0; i < nums.length - 2; i++) {
            // 去重
            if (i > 0 && nums[i] == nums[i - 1]) {
                continue;
            }
            // 第二和第三个值
            int left = i + 1;
            int right = nums.length - 1;
            while (left < right) {
                // 计算 sum
                int sum = nums[i] + nums[left] + nums[right];
                if (sum == 0) {
                    // 获取结果
                    result.add(Arrays.asList(nums[i], nums[left], nums[right]));
                    // 去重
                    while(left < right && nums[left] == nums[left + 1]) {
                        left += 1;
                    }
                    // 去重
                    while(left < right && nums[right] == nums[right - 1]) {
                        right -= 1;
                    }
                }
								// 抱住每次都会增加或者减少
                if (sum > 0) {
                    right -= 1;
                } else {
                    left += 1;
                }
            }
        }
        return result;
    }
}
```

#### 岛屿的最大面积

给定一个包含了一些 `0` 和 `1` 的非空二维数组 `grid` 。

一个 **岛屿** 是由一些相邻的 `1` (代表土地) 构成的组合，这里的「相邻」要求两个 `1` 必须在水平或者竖直方向上相邻。你可以假设 `grid` 的四个边缘都被 `0`（代表水）包围着。

找到给定的二维数组中最大的岛屿面积。(如果没有岛屿，则返回面积为 `0` 。)

 

**示例 1:**

```
[[0,0,1,0,0,0,0,1,0,0,0,0,0],
 [0,0,0,0,0,0,0,1,1,1,0,0,0],
 [0,1,1,0,1,0,0,0,0,0,0,0,0],
 [0,1,0,0,1,1,0,0,1,0,1,0,0],
 [0,1,0,0,1,1,0,0,1,1,1,0,0],
 [0,0,0,0,0,0,0,0,0,0,1,0,0],
 [0,0,0,0,0,0,0,1,1,1,0,0,0],
 [0,0,0,0,0,0,0,1,1,0,0,0,0]]
```

对于上面这个给定矩阵应返回 `6`。注意答案不应该是 `11` ，因为岛屿只能包含水平或垂直的四个方向的 `1` 。

**示例 2:**

```
[[0,0,0,0,0,0,0,0]]
```

对于上面这个给定的矩阵, 返回 `0`。

 

**注意:** 给定的矩阵`grid` 的长度和宽度都不超过 50。

```java
class Solution {
    private int[][] direction = new int[][]{{1, 0}, {0, 1}, {-1, 0}, {0, -1}};
    int cols = 0;
    int rows = 0;
    public int maxAreaOfIsland(int[][] grid) {
        if (grid == null || grid.length == 0) {
            return 0;
        }
        rows = grid.length;
        cols = grid[0].length;
        int maxa = 0;
        for (int i = 0; i < rows; i++) {
            for (int j = 0; j < cols; j++) {
                if (grid[i][j] == 1) {
                    maxa = Math.max(maxa, dfsa(grid, i, j));
                }
            }
        }
        return maxa;
    }
    
    private int dfsa(int[][] grid, int row, int col) {
        if (row < 0 || row >= rows || col < 0 || col >= cols || grid[row][col] == 0) {
            return 0;
        }
        grid[row][col] = 0;
        int area = 1;
        for (int[] di : direction) {
            area += dfsa(grid, row + di[0], col + di[1]);
        }
        return area;
    }
}
```

#### 搜索旋转排序数组

假设按照升序排序的数组在预先未知的某个点上进行了旋转。

( 例如，数组 `[0,1,2,4,5,6,7]` 可能变为 `[4,5,6,7,0,1,2]` )。

搜索一个给定的目标值，如果数组中存在这个目标值，则返回它的索引，否则返回 `-1` 。

你可以假设数组中不存在重复的元素。

你的算法时间复杂度必须是 *O*(log *n*) 级别。

**示例 1:**

```
输入: nums = [4,5,6,7,0,1,2], target = 0
输出: 4
```

**示例 2:**

```
输入: nums = [4,5,6,7,0,1,2], target = 3
输出: -1
```

```java
class Solution {
    public int search(int[] nums, int target) {
        int lo = 0;
        int hi = nums.length - 1;
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            if (nums[mid] == target) {
                return mid;
            }
            // 左边连续
            if (nums[lo] <= nums[mid]) {
                // 在 lo 和 mid 中间则设置 hi 为 mid 继续查找
                if (target <= nums[mid] && target >= nums[lo]) {
                    hi = mid - 1;
                } else {    // 不在 lo 和 mid 中间，则设置 lo 为 mid 继续查找
                    lo = mid + 1;
                }
            } else { // 右边连续
                // 在 mid 和 hi 中间则设置 lo 为 mid 继续查找
                if (target >= nums[mid] && target <= nums[hi]) {
                    lo = mid + 1;
                } else {    // 不在 mid 和 hi 中间，则设置 hi 为 mid 继续查找
                    hi = mid - 1;
                }
            }
        }
        return -1;
    }
}
```

#### 最长连续递增序列

给定一个未经排序的整数数组，找到最长且**连续**的的递增序列，并返回该序列的长度。

 

**示例 1:**

```
输入: [1,3,5,4,7]
输出: 3
解释: 最长连续递增序列是 [1,3,5], 长度为3。
尽管 [1,3,5,7] 也是升序的子序列, 但它不是连续的，因为5和7在原数组里被4隔开。 
```

**示例 2:**

```
输入: [2,2,2,2,2]
输出: 1
解释: 最长连续递增序列是 [2], 长度为1。
```

 

**注意：**数组长度不会超过10000。

```java
class Solution {
    public int findLengthOfLCIS(int[] nums) {
        if (nums == null || nums.length == 0) {
            return 0;
        }
        int mcnt = 1;
        int cnt = 1;
        for (int i = 1; i < nums.length; i++) {
            if (nums[i] > nums[i - 1]) {
                cnt += 1;
                mcnt = Math.max(cnt, mcnt);
            } else {
                cnt = 1;
            }
        }
        return mcnt;
    }
}
```

#### 数组中的第K个最大元素

在未排序的数组中找到第 **k** 个最大的元素。请注意，你需要找的是数组排序后的第 k 个最大的元素，而不是第 k 个不同的元素。

**示例 1:**

```
输入: [3,2,1,5,6,4] 和 k = 2
输出: 5
```

**示例 2:**

```
输入: [3,2,3,1,2,4,5,5,6] 和 k = 4
输出: 4
```

**说明:**

你可以假设 k 总是有效的，且 1 ≤ k ≤ 数组的长度。

```java
class Solution {
    public int findKthLargest(int[] nums, int k) {
        // 翻转求值
        k = nums.length - k;
        int lo = 0;
        int hi = nums.length - 1;
        while (lo < hi) {
            // 计算轴的位置（即轴左边的均比轴小，轴右边的均比轴大）
            int t = partition(nums, lo, hi);
            if (t == k) {
                break;
            } else if (t > k) {
                hi = t - 1;
            } else {
                lo = t + 1;
            }
        }
        return nums[k];
    }
    
    private int partition(int[] nums, int left, int right) {
        // 最左边的值作为轴
        int p = nums[left];
        // 左/右边初始值 -- 后续计算的时候会前置操作，所以 hi 此处 + 1
        int lo = left;
        int hi = right + 1;
        while (lo < hi) {
            // 左边小于轴则一直移动左边的下标
            while (lo < right && nums[++lo] < p) {};
            // 右边大于轴则一直移动右边的下标
            while (hi > left && nums[--hi] > p) {};
            // 左边大于右边则结束循环
            if (lo >= hi) {
                break;
            }
            // 交换左边大于轴和右边小于轴的元素
            swap(nums, lo, hi);
        }
        // 将轴放到正确的位置，此时轴的位置还是初始的元素，hi 处是小于 轴的一个值
        swap(nums, left, hi);
        // 位置
        return hi;
    }
    
    private void swap(int[] nums, int i, int j) {
        int tmp = nums[i];
        nums[i] = nums[j];
        nums[j] = tmp;
    }
}
```

#### 最长连续序列

给定一个未排序的整数数组，找出最长连续序列的长度。

要求算法的时间复杂度为 *O(n)*。

**示例:**

```
输入: [100, 4, 200, 1, 3, 2]
输出: 4
解释: 最长连续序列是 [1, 2, 3, 4]。它的长度为 4。
```

```java
class Solution {
    public int longestConsecutive(int[] nums) {
        // 从某个值出发的最⼤连续数
        Map<Integer, Integer> numCntMap = new HashMap<>();
        // 默认全部设置为 1
        for (int num : nums) {
            numCntMap.put(num, 1);
        }
        // 遍历各个元素，逐个去查找连续的值
        for (int num : nums) {
            numCntMap.put(num, forward(numCntMap, num));
        }
        int res = 0;
        for (int cnt : numCntMap.values()) {
            res = Math.max(res, cnt);
        }
        return res;
    }
    
    private int forward(Map<Integer, Integer> numCntMap, int num) {
        // 如果不包含当前的元素，表明查找结束返回 0
        if (!numCntMap.containsKey(num)) {
            return 0;
        }
        // 如果 cnt ⼤于 1 则说明该点已经查找过了，直接返回 cnt
        int cnt = numCntMap.get(num);
        if (cnt > 1) {
            return cnt;
        }
        // 继续查找下⼀个元素的后续可能有多个值
        cnt += forward(numCntMap, num + 1);
        // 将当前值后续的有多少个连续值的结果放到 map 中
        numCntMap.put(num, cnt);
        return cnt;
    }
}
```

#### 第k个排列

给出集合 `[1,2,3,…,*n*]`，其所有元素共有 *n*! 种排列。

按大小顺序列出所有排列情况，并一一标记，当 *n* = 3 时, 所有排列如下：

1. `"123"`
2. `"132"`
3. `"213"`
4. `"231"`
5. `"312"`
6. `"321"`

给定 *n* 和 *k*，返回第 *k* 个排列。

**说明：**

- 给定 *n* 的范围是 [1, 9]。
- 给定 *k* 的范围是[1,  *n*!]。

**示例 1:**

```
输入: n = 3, k = 3
输出: "213"
```

**示例 2:**

```
输入: n = 4, k = 9
输出: "2314"
```

```java
class Solution {
    // https://www.youtube.com/watch?v=xdvPD1IiyUM
    public String getPermutation(int n, int k) {
        char[] res = new char[n];
        ArrayList<Integer> nums = new ArrayList<>();
        int[] fact = new int[n];
        fact[0] = 1;
        // 构建阶乘数组 fact
        for (int i = 1; i < n; i++) {
            fact[i] = fact[i - 1] * i;
        }
        // 获取所有可用元素
        for (int i = 1; i <= n; i++) {
            nums.add(i);
        }
        // 获取 index 位置
        k -= 1;
        for (int i = 0; i < n; i++) {
            // n - 1 - i 表示确定当前位之后还有几位需要确定
            // fact[n - 1 - i] 剩余几位的排列数
            // k / fact[n - 1 - i] 获取当前对应的下标
            res[i] = Character.forDigit(nums.remove(k / fact[n - 1 - i]), 10);
            // 计算还需要多少位
            k = k % fact[n - 1 - i];
        }
        return new String(res);
    }
}
```

班上有 **N** 名学生。其中有些人是朋友，有些则不是。他们的友谊具有是传递性。如果已知 A 是 B 的朋友，B 是 C 的朋友，那么我们可以认为 A 也是 C 的朋友。所谓的朋友圈，是指所有朋友的集合。

给定一个 **N \* N** 的矩阵 **M**，表示班级中学生之间的朋友关系。如果M[i][j] = 1，表示已知第 i 个和 j 个学生**互为**朋友关系，否则为不知道。你必须输出所有学生中的已知的朋友圈总数。

**示例 1:**

```
输入: 
[[1,1,0],
 [1,1,0],
 [0,0,1]]
输出: 2 
说明：已知学生0和学生1互为朋友，他们在一个朋友圈。
第2个学生自己在一个朋友圈。所以返回2。
```

**示例 2:**

```
输入: 
[[1,1,0],
 [1,1,1],
 [0,1,1]]
输出: 1
说明：已知学生0和学生1互为朋友，学生1和学生2互为朋友，所以学生0和学生2也是朋友，所以他们三个在一个朋友圈，返回1。
```

**注意：**

1. N 在[1,200]的范围内。
2. 对于所有学生，有M[i][i] = 1。
3. 如果有M[i][j] = 1，则有M[j][i] = 1。

```java
class Solution {
    public int findCircleNum(int[][] M) {
        int m = M.length;
        boolean[] visited = new boolean[m];
        int cnt = 0;
        for (int i = 0; i < m; i ++) {
            if (!visited[i]) {
                dfsf(M, i, visited);
                cnt += 1;
            }
        }
        return cnt;
    }
    
    private void dfsf(int[][] ffs, int i, boolean[] visited) {
        visited[i] = true;
        for (int j = 0; j < ffs.length; j++) {
            if (ffs[i][j] == 1 && !visited[j]) {
                dfsf(ffs, j, visited);
            }
        }
    }
}
```

#### 合并区间

给出一个区间的集合，请合并所有重叠的区间。

**示例 1:**

```
输入: [[1,3],[2,6],[8,10],[15,18]]
输出: [[1,6],[8,10],[15,18]]
解释: 区间 [1,3] 和 [2,6] 重叠, 将它们合并为 [1,6].
```

**示例 2:**

```
输入: [[1,4],[4,5]]
输出: [[1,5]]
解释: 区间 [1,4] 和 [4,5] 可被视为重叠区间。
```

```java
class Solution {
    public int[][] merge(int[][] intervals) {
      	// 合并区间相关的都是先排序
        Arrays.sort(intervals, Comparator.comparingInt(o -> o[0]));
        int length = intervals.length;
        List<int[]> ans = new ArrayList<>();
      	// 逐个比较
        for (int i = 0; i < length; i++) {
            int[] sa = new int[2];
            sa[0] = intervals[i][0];
            sa[1] = intervals[i][1];
          	// 拿 1 和 后面 0 比较，逐个合并区间
            while (i + 1 < length && sa[1] >= intervals[i+1][0]) {
                sa[1] = Math.max(sa[1], intervals[i + 1][1]);
                i += 1;
            }
            ans.add(sa);
        }
        return ans.toArray(new int[0][]);
    }
}
```



#### 接雨水

给定 *n* 个非负整数表示每个宽度为 1 的柱子的高度图，计算按此排列的柱子，下雨之后能接多少雨水。

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/10/22/rainwatertrap.png)

上面是由数组 [0,1,0,2,1,0,1,3,2,1,2,1] 表示的高度图，在这种情况下，可以接 6 个单位的雨水（蓝色部分表示雨水）。 **感谢 Marcos** 贡献此图。

**示例:**

```
输入: [0,1,0,2,1,0,1,3,2,1,2,1]
输出: 6
```

```java
class Solution {
    // https://www.bilibili.com/video/av83646139?from=search&seid=9170732822822571265
    public int trap(int[] height) {
        int n = height.length;
        // 0 - i 最高的值
        int[] lo = new int[n];
        // i - n - 1 的最大值
        int[] hi = new int[n];
        for (int i = 0; i < n; i++) {
            lo[i] = i == 0 ? height[i] : Math.max(height[i], lo[i - 1]);
        }
        for (int i = n - 1; i >= 0; i--) {
            hi[i] = i == (n - 1) ? height[i] : Math.max(height[i], hi[i + 1]);
        }
        int ans = 0;
        for (int i = 0; i < n; i++) {
            // 取左边最大的和右边最大的二者中最小的一个减去当前的即当前能存储水的量
            ans += Math.min(lo[i], hi[i]) - height[i];
        }
        return ans;
    }
}
```



### 链表或树（9）

#### 合并两个有序链表

将两个升序链表合并为一个新的 **升序** 链表并返回。新链表是通过拼接给定的两个链表的所有节点组成的。 

 

**示例：**

```
输入：1->2->4, 1->3->4
输出：1->1->2->3->4->4
```

```java
class Solution {
    public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        if (l1 == null) return l2;
        if (l2 == null) return l1;
        if (l1.val < l2.val) {
            l1.next = mergeTwoLists(l1.next, l2);
            return l1;
        } else {
            l2.next = mergeTwoLists(l1, l2.next);
            return l2;
        }
    }
}
```

#### 反转链表

反转一个单链表。

**示例:**

```
输入: 1->2->3->4->5->NULL
输出: 5->4->3->2->1->NULL
```

**进阶:**
你可以迭代或递归地反转链表。你能否用两种方法解决这道题？

```java
class Solution {
    public ListNode reverseList(ListNode head) {
        if (head == null || head.next == null) {
            return head;
        }
        ListNode next = head.next;
        ListNode newHeader = reverseList(next);
        next.next = head;
        head.next = null;
        return newHeader;
    }
}
```

#### 两数相加

给出两个 **非空** 的链表用来表示两个非负的整数。其中，它们各自的位数是按照 **逆序** 的方式存储的，并且它们的每个节点只能存储 **一位** 数字。

如果，我们将这两个数相加起来，则会返回一个新的链表来表示它们的和。

您可以假设除了数字 0 之外，这两个数都不会以 0 开头。

**示例：**

```
输入：(2 -> 4 -> 3) + (5 -> 6 -> 4)
输出：7 -> 0 -> 8
原因：342 + 465 = 807
```

```java
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        ListNode dummyHeader = new ListNode(-1);
        ListNode tmpNode = dummyHeader;
        int sum = 0;
        int carry = 0;
        while (l1 != null || l2 != null) {
            int v1 = l1 == null ? 0 : l1.val;
            int v2 = l2 == null ? 0 : l2.val;
            sum = v1 + v2 + carry;
            tmpNode.next = new ListNode(sum % 10);
            tmpNode = tmpNode.next;
            carry = sum / 10;
            if (l1 != null) l1 = l1.next;
            if (l2 != null) l2 = l2.next;
        }
        if (carry > 0) {
            tmpNode.next = new ListNode(carry);
        }
        return dummyHeader.next;
    }
}
```

#### 排序链表

在 *O*(*n* log *n*) 时间复杂度和常数级空间复杂度下，对链表进行排序。

**示例 1:**

```
输入: 4->2->1->3
输出: 1->2->3->4
```

**示例 2:**

```
输入: -1->5->3->4->0
输出: -1->0->3->4->5
```

```java
class Solution {
    public ListNode sortList(ListNode head) {
        if (head == null || head.next == null) {
            return head;
        }
        // 设置 dummy 头结点
        ListNode dummy = new ListNode(-1);
        dummy.next = head;
        int length = getLength(head);
        
        // 每次排序的长度
        int itrv = 1;
        while (itrv < length) {
            ListNode pre = dummy;
            ListNode h = dummy.next;
            // 归并排序，第一次链表中每 2 个比较一次，第二次每 4 个比较一次，第三次 8 一次类推
            while (h != null) {
                // 找到第一组需要排序节点 1，2，4
                int i = itrv;
                ListNode h1 = h;
                for (; h != null && i > 0; i--) {
                    h = h.next;
                }
                // 大于 0 表示就一组
                if (i > 0) {
                    break;
                }
                // 找到第二组要排序的节点，1，2，4
                ListNode h2 = h;
                i = itrv;
                for (; h != null && i > 0; i--) {
                    h = h.next;
                }
                
                // 第一组的长度
                int c1 = itrv;
                // 第二组的长度
                int c2 = itrv - i;
                
                // 合并两组节点
                while (c1 > 0 && c2 > 0) {
                    if (h1.val < h2.val) {
                        pre.next = h1;
                        h1 = h1.next;
                        c1 -= 1;
                    } else {
                        pre.next = h2;
                        h2 = h2.next;
                        c2 -= 1;
                    }
                    pre = pre.next;
                }
                // 走完已排序好的节点
                pre.next = c1 > 0 ? h1 : h2;
                while (c1 > 0 || c2 > 0) {
                    pre = pre.next;
                    c1 -= 1;
                    c2 -= 1;
                }
                // 连接未排序的节点
                pre.next = h;
            }
            itrv *= 2;
        }
        return dummy.next;
    }
    
    private int getLength(ListNode head) {
        ListNode cur = head;
        int length = 0;
        while(cur != null) {
            length += 1;
            cur = cur.next;
        }
        return length;
    }
}
```

#### 环形链表 II

给定一个链表，返回链表开始入环的第一个节点。 如果链表无环，则返回 `null`。

为了表示给定链表中的环，我们使用整数 `pos` 来表示链表尾连接到链表中的位置（索引从 0 开始）。 如果 `pos` 是 `-1`，则在该链表中没有环。

**说明：**不允许修改给定的链表。

 

**示例 1：**

```
输入：head = [3,2,0,-4], pos = 1
输出：tail connects to node index 1
解释：链表中有一个环，其尾部连接到第二个节点。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist.png)

**示例 2：**

```
输入：head = [1,2], pos = 0
输出：tail connects to node index 0
解释：链表中有一个环，其尾部连接到第一个节点。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist_test2.png)

**示例 3：**

```
输入：head = [1], pos = -1
输出：no cycle
解释：链表中没有环。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist_test3.png)

 

**进阶：**
你是否可以不用额外空间解决此题？

```java
public class Solution {
    public ListNode detectCycle(ListNode head) {
        if (head == null || head.next == null) {
            return null;
        }
        ListNode midNode = getCycNode(head);
        if (midNode == null) {
            return midNode;
        }
        ListNode resNode = head;
        while(resNode != midNode) {
            resNode = resNode.next;
            midNode = midNode.next;
        }
        return resNode;
    }
    
    private ListNode getCycNode(ListNode head) {
        ListNode fast = head;
        ListNode slow = head;
        while(fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
            if (slow == fast) {
                return slow;
            }
        }
        return null;
    }
}
```

相交链表

编写一个程序，找到两个单链表相交的起始节点。

如下面的两个链表**：**

[![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/14/160_statement.png)](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/14/160_statement.png)

在节点 c1 开始相交。

 

**示例 1：**

[![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/14/160_example_1.png)](https://assets.leetcode.com/uploads/2018/12/13/160_example_1.png)

```
输入：intersectVal = 8, listA = [4,1,8,4,5], listB = [5,0,1,8,4,5], skipA = 2, skipB = 3
输出：Reference of the node with value = 8
输入解释：相交节点的值为 8 （注意，如果两个链表相交则不能为 0）。从各自的表头开始算起，链表 A 为 [4,1,8,4,5]，链表 B 为 [5,0,1,8,4,5]。在 A 中，相交节点前有 2 个节点；在 B 中，相交节点前有 3 个节点。
```

 

**示例 2：**

[![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/14/160_example_2.png)](https://assets.leetcode.com/uploads/2018/12/13/160_example_2.png)

```
输入：intersectVal = 2, listA = [0,9,1,2,4], listB = [3,2,4], skipA = 3, skipB = 1
输出：Reference of the node with value = 2
输入解释：相交节点的值为 2 （注意，如果两个链表相交则不能为 0）。从各自的表头开始算起，链表 A 为 [0,9,1,2,4]，链表 B 为 [3,2,4]。在 A 中，相交节点前有 3 个节点；在 B 中，相交节点前有 1 个节点。
```

 

**示例 3：**

[![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/14/160_example_3.png)](https://assets.leetcode.com/uploads/2018/12/13/160_example_3.png)

```
输入：intersectVal = 0, listA = [2,6,4], listB = [1,5], skipA = 3, skipB = 2
输出：null
输入解释：从各自的表头开始算起，链表 A 为 [2,6,4]，链表 B 为 [1,5]。由于这两个链表不相交，所以 intersectVal 必须为 0，而 skipA 和 skipB 可以是任意值。
解释：这两个链表不相交，因此返回 null。
```

 

**注意：**

- 如果两个链表没有交点，返回 `null`.
- 在返回结果后，两个链表仍须保持原有的结构。
- 可假定整个链表结构中没有循环。
- 程序尽量满足 O(*n*) 时间复杂度，且仅用 O(*1*) 内存。

```java
public class Solution {
    public ListNode getIntersectionNode(ListNode headA, ListNode headB) {
        if (headA == null | headB == null) {
            return null;
        }
        ListNode p1 = headA;
        ListNode p2 = headB;
        while (p1 != p2) {
            p1 = p1 == null ? headB : p1.next;
            p2 = p2 == null ? headA : p2.next;
        }
        return p1;
    }
}
```

#### 合并K个排序链表

合并 *k* 个排序链表，返回合并后的排序链表。请分析和描述算法的复杂度。

**示例:**

```
输入:
[
  1->4->5,
  1->3->4,
  2->6
]
输出: 1->1->2->3->4->4->5->6
```

```java
class Solution {
   public ListNode mergeKLists(ListNode[] lists) {
        return merge(lists, 0, lists.length - 1);
    }

    private ListNode merge(ListNode[] lists, int lo, int hi) {
        if (lo > hi) {
            return null;
        }
        if (lo == hi) {
            return lists[lo];
        }
        int mid = lo + (hi - lo) / 2;
        ListNode l1 = merge(lists, lo, mid);
        ListNode l2 = merge(lists, mid + 1, hi);
        return mergeTwoLists(l1, l2);
    }

    private ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        if (l1 == null) return l2;
        if (l2 == null) return l1;
        if (l1.val < l2.val) {
            l1.next = mergeTwoLists(l1.next, l2);
            return l1;
        } else {
            l2.next = mergeTwoLists(l1, l2.next);
            return l2;
        }
    }
}
```

#### 二叉树的最近公共祖先

给定一个二叉树, 找到该树中两个指定节点的最近公共祖先。

[百度百科](https://baike.baidu.com/item/最近公共祖先/8918834?fr=aladdin)中最近公共祖先的定义为：“对于有根树 T 的两个结点 p、q，最近公共祖先表示为一个结点 x，满足 x 是 p、q 的祖先且 x 的深度尽可能大（**一个节点也可以是它自己的祖先**）。”

例如，给定如下二叉树: root = [3,5,1,6,2,0,8,null,null,7,4]

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/15/binarytree.png)

 

**示例 1:**

```
输入: root = [3,5,1,6,2,0,8,null,null,7,4], p = 5, q = 1
输出: 3
解释: 节点 5 和节点 1 的最近公共祖先是节点 3。
```

**示例 2:**

```
输入: root = [3,5,1,6,2,0,8,null,null,7,4], p = 5, q = 4
输出: 5
解释: 节点 5 和节点 4 的最近公共祖先是节点 5。因为根据定义最近公共祖先节点可以为节点本身。
```

 

**说明:**

- 所有节点的值都是唯一的。
- p、q 为不同节点且均存在于给定的二叉树中。



```java
class Solution {
    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        if (root == null || root == p || root == q) {
            return root;
        }
        TreeNode left = lowestCommonAncestor(root.left, p, q);
        TreeNode right = lowestCommonAncestor(root.right, p, q);
        return left == null ? right : right == null ? left : root;
    }
}
```

#### 二叉树的锯齿形层次遍历

给定一个二叉树，返回其节点值的锯齿形层次遍历。（即先从左往右，再从右往左进行下一层遍历，以此类推，层与层之间交替进行）。

例如：
给定二叉树 `[3,9,20,null,null,15,7]`,

```
    3
   / \
  9  20
    /  \
   15   7
```

返回锯齿形层次遍历如下：

```
[
  [3],
  [20,9],
  [15,7]
]
```

```java
class Solution {
    public List<List<Integer>> zigzagLevelOrder(TreeNode root) {
        List<List<Integer>> res = new ArrayList<>();
        if (root == null) {
            return res;
        }
        Queue<TreeNode> queue = new LinkedList<>();
        queue.add(root);
        boolean isRe = false;
        while(!queue.isEmpty()) {
            List<Integer> singleRes = new ArrayList<>();
            int size = queue.size();
            while(size-- > 0) {
                TreeNode node = queue.poll();
                if (node == null) {
                    continue;
                }
                singleRes.add(node.val);
                queue.add(node.left);
                queue.add(node.right);
            }
            if (singleRes.size() > 0) {
                if (isRe) {
                    Collections.reverse(singleRes);
                }
                res.add(singleRes);
                isRe = !isRe;
            }
        }
        return res;
    }
}
```



### 动态与贪心（6）

#### 买卖股票的最佳时机

给定一个数组，它的第 *i* 个元素是一支给定股票第 *i* 天的价格。

如果你最多只允许完成一笔交易（即买入和卖出一支股票一次），设计一个算法来计算你所能获取的最大利润。

注意：你不能在买入股票前卖出股票。

 

**示例 1:**

```
输入: [7,1,5,3,6,4]
输出: 5
解释: 在第 2 天（股票价格 = 1）的时候买入，在第 5 天（股票价格 = 6）的时候卖出，最大利润 = 6-1 = 5 。
     注意利润不能是 7-1 = 6, 因为卖出价格需要大于买入价格；同时，你不能在买入前卖出股票。
```

**示例 2:**

```
输入: [7,6,4,3,1]
输出: 0
解释: 在这种情况下, 没有交易完成, 所以最大利润为 0。
```

```java
class Solution {
    public int maxProfit(int[] prices) {
        if (prices.length < 2) {
            return 0;
        }
        int soFarMin = prices[0];
        int res = 0;
        for (int i = 1; i < prices.length; i++) {
            if (soFarMin > prices[i]) {
                soFarMin = prices[i];
            }
            res = Math.max(res, prices[i] - soFarMin);
        }
        return res;
    }
}
```



#### 买卖股票的最佳时机 II

给定一个数组，它的第 *i* 个元素是一支给定股票第 *i* 天的价格。

设计一个算法来计算你所能获取的最大利润。你可以尽可能地完成更多的交易（多次买卖一支股票）。

**注意：**你不能同时参与多笔交易（你必须在再次购买前出售掉之前的股票）。

 

**示例 1:**

```
输入: [7,1,5,3,6,4]
输出: 7
解释: 在第 2 天（股票价格 = 1）的时候买入，在第 3 天（股票价格 = 5）的时候卖出, 这笔交易所能获得利润 = 5-1 = 4 。
     随后，在第 4 天（股票价格 = 3）的时候买入，在第 5 天（股票价格 = 6）的时候卖出, 这笔交易所能获得利润 = 6-3 = 3 。
```

**示例 2:**

```
输入: [1,2,3,4,5]
输出: 4
解释: 在第 1 天（股票价格 = 1）的时候买入，在第 5 天 （股票价格 = 5）的时候卖出, 这笔交易所能获得利润 = 5-1 = 4 。
     注意你不能在第 1 天和第 2 天接连购买股票，之后再将它们卖出。
     因为这样属于同时参与了多笔交易，你必须在再次购买前出售掉之前的股票。
```

**示例 3:**

```
输入: [7,6,4,3,1]
输出: 0
解释: 在这种情况下, 没有交易完成, 所以最大利润为 0。
```

 

**提示：**

- `1 <= prices.length <= 3 * 10 ^ 4`
- `0 <= prices[i] <= 10 ^ 4`

```java
class Solution {
    public int maxProfit(int[] prices) {
        int max = 0;
        for (int i = 1; i < prices.length; i++) {
            if (prices[i] > prices[i - 1]) {
                max += (prices[i] - prices[i - 1]);
            }
        }
        return max;
    }
}
```



#### 最大正方形

在一个由 0 和 1 组成的二维矩阵内，找到只包含 1 的最大正方形，并返回其面积。

**示例:**

```
输入: 

1 0 1 0 0
1 0 1 1 1
1 1 1 1 1
1 0 0 1 0

输出: 4
```

![image-20200724001000463](https://i.loli.net/2020/07/24/amhqbi7QyDZX98l.png)

```java
class Solution {
    public int maximalSquare(char[][] matrix) {
        if (matrix == null || matrix.length == 0) {
            return 0;
        }
        int rows = matrix.length;
        int cols = matrix[0].length;
        
        int[] dp = new int[cols + 1];
        int mg = 0;
        int pre = 0;
        for (int i = 1; i <= rows; i++) {
            for (int j = 1; j <= cols; j++) {
	              // 保存上一次 j ，最后放到 pre 中
                int tmp = dp[j];
                if (matrix[i - 1][j - 1] == '1') {
                    dp[j] = Math.min(Math.min(dp[j - 1], pre), dp[j]) + 1;
                    mg = Math.max(dp[j], mg);
                } else {
                    dp[j] = 0;
                }
                pre = tmp;
            }
        }
        return mg * mg;
    }
}
```



####  最大子序和

给定一个整数数组 `nums` ，找到一个具有最大和的连续子数组（子数组最少包含一个元素），返回其最大和。

**示例:**

```
输入: [-2,1,-3,4,-1,2,1,-5,4],
输出: 6
解释: 连续子数组 [4,-1,2,1] 的和最大，为 6。
```

**进阶:**

如果你已经实现复杂度为 O(*n*) 的解法，尝试使用更为精妙的分治法求解。

```java
class Solution {
    public int maxSubArray(int[] nums) {
        int sum = 0;
        int gSum = Integer.MIN_VALUE;
        for (int num : nums) {
            sum = (sum <= 0) ? num : sum + num;
            gSum = Math.max(sum, gSum);
        }
        return gSum;
    }
}
```



#### 三角形最小路径和

给定一个三角形，找出自顶向下的最小路径和。每一步只能移动到下一行中相邻的结点上。

**相邻的结点** 在这里指的是 `下标` 与 `上一层结点下标` 相同或者等于 `上一层结点下标 + 1` 的两个结点。

 

例如，给定三角形：

```
[
     [2],
    [3,4],
   [6,5,7],
  [4,1,8,3]
]
```

自顶向下的最小路径和为 `11`（即，**2** + **3** + **5** + **1** = 11）。

 

**说明：**

如果你可以只使用 *O*(*n*) 的额外空间（*n* 为三角形的总行数）来解决这个问题，那么你的算法会很加分。

```java
class Solution {
    public int minimumTotal(List<List<Integer>> triangle) {
        if (triangle == null || triangle.size() == 0) {
            return 0;
        }
	      // 自底向上
        int[] dp = new int[triangle.size() + 1];
        for (int i = triangle.size() - 1; i >= 0; i--) {
            List<Integer> curRow = triangle.get(i);
            for (int j = 0; j < curRow.size(); j++) {
                dp[j] = Math.min(dp[j], dp[j + 1]) + curRow.get(j);
            }
        }
        return dp[0];
    }
}
```



#### 俄罗斯套娃信封问题

给定一些标记了宽度和高度的信封，宽度和高度以整数对形式 `(w, h)` 出现。当另一个信封的宽度和高度都比这个信封大的时候，这个信封就可以放进另一个信封里，如同俄罗斯套娃一样。

请计算最多能有多少个信封能组成一组“俄罗斯套娃”信封（即可以把一个信封放到另一个信封里面）。

**说明:**
不允许旋转信封。

**示例:**

```
输入: envelopes = [[5,4],[6,4],[6,7],[2,3]]
输出: 3 
解释: 最多信封的个数为 3, 组合为: [2,3] => [5,4] => [6,7]。
```

```java
class Solution {
    public int lol(int[] nums) {
        int[] dp = new int[nums.length];
        int len = 0;
        for (int num : nums) {
          	// 如果找到则返回下标，如果找不到则返回值是 -(插入点 - 1)
            int i = Arrays.binarySearch(dp, 0, len, num);
            if (i < 0) {
              	// 找到插入点
                i = - (i + 1);
            }
          	// 插入数据
            dp[i] = num;
          	// 只有在找到比自己大的时候才进行扩容。
            if (i == len) {
                len += 1;
            }
        }
        return len;
    }
    
	  // 排序后 + 最长递增子序列
    public int maxEnvelopes(int[][] envelopes) {
	      // 因为相同宽度的信封无法放入所以，0 正序，1 倒序
      	// [1, 2] 无法放到 [1, 3] 中
        Arrays.sort(envelopes, (ar1, ar2) -> {
            public int compare(int[] ar1, int[] ar2) {
                if (ar1[0] == ar2[0]) {
                    return ar2[1] - ar1[1];
                } else {
                    return ar1[0] - ar2[0];
                }
            } 
        });
        int[] secondDim = new int[envelopes.length];
        for (int i = 0; i < envelopes.length; i++) {
            secondDim[i] = envelopes[i][1];
        }
	      // 求 1 的最长递增子序列
        return lol(secondDim);
    }
}
```



### 数据结构（3）

#### 最小栈

设计一个支持 `push` ，`pop` ，`top` 操作，并能在常数时间内检索到最小元素的栈。

- `push(x)` —— 将元素 x 推入栈中。
- `pop()` —— 删除栈顶的元素。
- `top()` —— 获取栈顶元素。
- `getMin()` —— 检索栈中的最小元素。

 

**示例:**

```
输入：
["MinStack","push","push","push","getMin","pop","top","getMin"]
[[],[-2],[0],[-3],[],[],[],[]]

输出：
[null,null,null,null,-3,null,0,-2]

解释：
MinStack minStack = new MinStack();
minStack.push(-2);
minStack.push(0);
minStack.push(-3);
minStack.getMin();   --> 返回 -3.
minStack.pop();
minStack.top();      --> 返回 0.
minStack.getMin();   --> 返回 -2.
```

 

**提示：**

- `pop`、`top` 和 `getMin` 操作总是在 **非空栈** 上调用。



```java
class MinStack {
    private Stack<Integer> dataStack;
    private Stack<Integer> minStack;
    public MinStack() {
        dataStack = new Stack<>();
        minStack = new Stack<>();
    }
    
    public void push(int x) {
        dataStack.push(x);
        minStack.push(minStack.isEmpty() ? x : Math.min(x, minStack.peek()));
    }
    
    public void pop() {
        dataStack.pop();
        minStack.pop();
    }
    
    public int top() {
        return dataStack.peek();
    }
    
    public int getMin() {
        return minStack.peek();
    }
}
```

#### LRU缓存机制

运用你所掌握的数据结构，设计和实现一个 [LRU (最近最少使用) 缓存机制](https://baike.baidu.com/item/LRU)。它应该支持以下操作： 获取数据 `get` 和 写入数据 `put` 。

获取数据 `get(key)` - 如果关键字 (key) 存在于缓存中，则获取关键字的值（总是正数），否则返回 -1。
写入数据 `put(key, value)` - 如果关键字已经存在，则变更其数据值；如果关键字不存在，则插入该组「关键字/值」。当缓存容量达到上限时，它应该在写入新数据之前删除最久未使用的数据值，从而为新的数据值留出空间。

 

**进阶:**

你是否可以在 **O(1)** 时间复杂度内完成这两种操作？

 

**示例:**

```
LRUCache cache = new LRUCache( 2 /* 缓存容量 */ );

cache.put(1, 1);
cache.put(2, 2);
cache.get(1);       // 返回  1
cache.put(3, 3);    // 该操作会使得关键字 2 作废
cache.get(2);       // 返回 -1 (未找到)
cache.put(4, 4);    // 该操作会使得关键字 1 作废
cache.get(1);       // 返回 -1 (未找到)
cache.get(3);       // 返回  3
cache.get(4);       // 返回  4
```

```java
class LRUCache extends LinkedHashMap<Integer, Integer> {
    private int capacity;

    public LRUCache(int capacity) {
        // initialCapacity – the initial capacity
        // loadFactor – the load factor
        // accessOrder – the ordering mode - true for access-order, false for insertion-order
        super(capacity, 0.75F, true);
        this.capacity = capacity;
    }
    
    public int get(int key) {
        return super.getOrDefault(key, -1);
    }
    
    public void put(int key, int value) {
        super.put(key, value);
    }
    
    // Returns true if this map should remove its eldest entry.
    protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
        return size() > capacity;
    }
}
```

```java
import java.util.Hashtable;
class LRUCache {
    private Hashtable<Integer, DLinkedNode> cache = new Hashtable<>();
    private int size;
    private int capacity;
    private DLinkedNode head, tail;
    
    class DLinkedNode {
        int key;
        int value;
        DLinkedNode pre;
        DLinkedNode next;
    }
    
    private void addNode(DLinkedNode node) {
        node.pre = head;
        node.next = head.next;
        
        head.next.pre = node;
        head.next = node;
    }
    
    private void removeNode(DLinkedNode node) {
        DLinkedNode pre = node.pre;
        DLinkedNode next = node.next;
        
        pre.next = next;
        next.pre = pre;
    }
    
    private void moveToHead(DLinkedNode node) {
        removeNode(node);
        addNode(node);
    }

    private DLinkedNode popTail() {
        DLinkedNode res = tail.pre;
        removeNode(res);
        return res;
    }
    
    public LRUCache(int capacity) {
        this.size = 0;
        this.capacity = capacity;
        
        head = new DLinkedNode();
        tail = new DLinkedNode();
        
        head.next = tail;
        tail.pre = head;
    }
    
    public int get(int key) {
        DLinkedNode node = cache.get(key);
        if (node == null) {
            return -1;
        }
        moveToHead(node);
        return node.value;
    }
    
    public void put(int key, int value) {
        DLinkedNode node = cache.get(key);
        if (node == null) {
            DLinkedNode newNode = new DLinkedNode();
            newNode.key = key;
            newNode.value = value;
            cache.put(key, newNode);
            addNode(newNode);
            ++size;
            if (size > capacity) {
                DLinkedNode tail = popTail();
                cache.remove(tail.key);
                --size;
            }
        } else {
            node.value = value;
            moveToHead(node);
        }
    }
}
```



#### 全 O(1) 的数据结构

请你实现一个数据结构支持以下操作：

1. `Inc(key)` - 插入一个新的值为 1 的 key。或者使一个存在的 key 增加一，保证 key 不为空字符串。
2. `Dec(key)` - 如果这个 key 的值是 1，那么把他从数据结构中移除掉。否则使一个存在的 key 值减一。如果这个 key 不存在，这个函数不做任何事情。key 保证不为空字符串。
3. `GetMaxKey()` - 返回 key 中值最大的任意一个。如果没有元素存在，返回一个空字符串`""` 。
4. `GetMinKey()` - 返回 key 中值最小的任意一个。如果没有元素存在，返回一个空字符串`""`。

 

**挑战：**

你能够以 O(1) 的时间复杂度实现所有操作吗？

```java

```



### 扩展练习（3）

#### x 的平方根

实现 `int sqrt(int x)` 函数。

计算并返回 *x* 的平方根，其中 *x* 是非负整数。

由于返回类型是整数，结果只保留整数的部分，小数部分将被舍去。

**示例 1:**

```
输入: 4
输出: 2
```

**示例 2:**

```
输入: 8
输出: 2
说明: 8 的平方根是 2.82842..., 
     由于返回类型是整数，小数部分将被舍去。
```

```java
class Solution {
    public int mySqrt(int x) {
        if (x <= 1) {
            return x;
        }
        int lo = 1;
        int hi = x;
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            int sqrt = x / mid;
            if (sqrt == mid) {
                return mid;
            } else if (sqrt < mid) {
                hi = mid - 1;
            } else {
                lo = mid + 1;
            }
        }
        return hi;
    }
}
```



#### UTF-8 编码验证

UTF-8 中的一个字符可能的长度为 **1 到 4 字节**，遵循以下的规则：

1. 对于 1 字节的字符，字节的第一位设为0，后面7位为这个符号的unicode码。
2. 对于 n 字节的字符 (n > 1)，第一个字节的前 n 位都设为1，第 n+1 位设为0，后面字节的前两位一律设为10。剩下的没有提及的二进制位，全部为这个符号的unicode码。

这是 UTF-8 编码的工作方式：

```
   Char. number range  |        UTF-8 octet sequence
      (hexadecimal)    |              (binary)
   --------------------+---------------------------------------------
   0000 0000-0000 007F | 0xxxxxxx
   0000 0080-0000 07FF | 110xxxxx 10xxxxxx
   0000 0800-0000 FFFF | 1110xxxx 10xxxxxx 10xxxxxx
   0001 0000-0010 FFFF | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
```

给定一个表示数据的整数数组，返回它是否为有效的 utf-8 编码。

**注意:**
输入是整数数组。只有每个整数的**最低 8 个有效位**用来存储数据。这意味着每个整数只表示 1 字节的数据。

**示例 1:**

```
data = [197, 130, 1], 表示 8 位的序列: 11000101 10000010 00000001.

返回 true 。
这是有效的 utf-8 编码，为一个2字节字符，跟着一个1字节字符。
```

**示例 2:**

```
data = [235, 140, 4], 表示 8 位的序列: 11101011 10001100 00000100.

返回 false 。
前 3 位都是 1 ，第 4 位为 0 表示它是一个3字节字符。
下一个字节是开头为 10 的延续字节，这是正确的。
但第二个延续字节不以 10 开头，所以是不符合规则的。
```

```java
class Solution {
    public boolean validUtf8(int[] data) {
        int numOfBytesToProcess = 0;
        for (int i = 0; i < data.length; i++) {
            String binRep = Integer.toBinaryString(data[i]);
            binRep = binRep.length() >= 8 ? binRep.substring(binRep.length() - 8) : "00000000".substring(binRep.length() % 8) + binRep;
            if (numOfBytesToProcess == 0) {
                for (int j = 0; j < binRep.length(); j++) {
                    if (binRep.charAt(j) == '0') {
                        break;
                    }
                    numOfBytesToProcess += 1;
                }
                if (numOfBytesToProcess == 0) {
                    continue;
                }
                if (numOfBytesToProcess > 4 || numOfBytesToProcess == 1) {
                    return false;
                }
            } else {
                if (!(binRep.charAt(0) == '1' && binRep.charAt(1) == '0')) {
                    return false;
                }
            }
            numOfBytesToProcess -= 1;
        }
        return numOfBytesToProcess == 0;
    }
}
```



#### 第二高的薪水

编写一个 SQL 查询，获取 `Employee` 表中第二高的薪水（Salary） 。

```
+----+--------+
| Id | Salary |
+----+--------+
| 1  | 100    |
| 2  | 200    |
| 3  | 300    |
+----+--------+
```

例如上述 `Employee` 表，SQL查询应该返回 `200` 作为第二高的薪水。如果不存在第二高的薪水，那么查询应返回 `null`。

```
+---------------------+
| SecondHighestSalary |
+---------------------+
| 200                 |
+---------------------+
```

```java
SELECT
    (SELECT DISTINCT
            Salary
        FROM
            Employee
        ORDER BY Salary DESC
        LIMIT 1 OFFSET 1) AS SecondHighestSalary
;
```

