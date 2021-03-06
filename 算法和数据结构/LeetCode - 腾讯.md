---
title: "LeetCode 腾讯"
tags: ["算法"]
categories: ["算法"]
date: "2019-01-12T09:00:00+08:00"
---

[toc]



### 数组与字符串 (18)

#### 两数之和

给定一个整数数组 `nums` 和一个目标值 `target`，请你在该数组中找出和为目标值的那 **两个** 整数，并返回他们的数组下标。

你可以假设每种输入只会对应一个答案。但是，数组中同一个元素不能使用两遍。

**示例:**

```
给定 nums = [2, 7, 11, 15], target = 9

因为 nums[0] + nums[1] = 2 + 7 = 9
所以返回 [0, 1]
```

---

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> nm = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            if (nm.containsKey(target - nums[i])) {
                return new int[]{nm.get(target - nums[i]), i};
            } else {
                nm.put(nums[i], i);
            }
        }
        return new int[2];
    }
}
```

#### 寻找两个正序数组的中位数

给定两个大小为 m 和 n 的正序（从小到大）数组 `nums1` 和 `nums2`。

请你找出这两个正序数组的中位数，并且要求算法的时间复杂度为 O(log(m + n))。

你可以假设 `nums1` 和 `nums2` 不会同时为空。

**示例 1:**

```
nums1 = [1, 3]
nums2 = [2]

则中位数是 2.0
```

**示例 2:**

```
nums1 = [1, 2]
nums2 = [3, 4]

则中位数是 (2 + 3)/2 = 2.5
```

---

```java
class Solution {
    // https://www.youtube.com/watch?v=KB9IcSCDQ9k&t=1s
    public double findMedianSortedArrays(int[] nums1, int[] nums2) {
        int l1 = nums1.length;
        int l2 = nums2.length;
        if (l1 > l2) {
            return findMedianSortedArrays(nums2, nums1);
        }
        // ![题解](https://i.loli.net/2020/05/07/L7pZ42CyIbQRSVJ.png)
        // k 是中位数的话
        // 从 l1 中取出 m1 个元素，从 l2 中取出 m2 个元素，构成 k 个元素
        // m1 + m2 = k = (l1 + l2 + 1) / 2
        int left = 0;
        int right = l1;
        int half = (l1 + l2 + 1) / 2; // 即 k 值
        while(left < right) {
            int m1 = left + (right - left) / 2;
            int m2 = half - m1;
            // 如果 m1 < m2 - 1 则表明左边的元素少了，需要增加左边的元素
            if (nums1[m1] < nums2[m2 - 1]) {
                left = m1 + 1;
            } else {
                right = m1;
            }
        }
        
        int ml = left;
        int mr = half - right;
        // m1 + m2 = k
        // 如果是奇数个元素，即 k 是 m1 - 1 和 m2 - 1 中大的一个
        int c1 = Math.max(ml <= 0 ? Integer.MIN_VALUE : nums1[ml - 1], mr <= 0 ? Integer.MIN_VALUE : nums2[mr - 1]);
        if ((l1 + l2) % 2 == 1) {
            return c1;
        }
        // 如果是偶数个元素则还需要找 k + 1 的值，即 m1 和 m2 中 小的一个
        int c2 = Math.min(ml >= l1 ? Integer.MAX_VALUE : nums1[ml], mr >= l2 ? Integer.MAX_VALUE : nums2[mr]);
        return (c1 + c2) / 2.0;
    }
}
```

![题解](https://i.loli.net/2020/05/07/L7pZ42CyIbQRSVJ.png)



#### 最长回文子串

给定一个字符串 `s`，找到 `s` 中最长的回文子串。你可以假设 `s` 的最大长度为 1000。

**示例 1：**

```
输入: "babad"
输出: "bab"
注意: "aba" 也是一个有效答案。
```

**示例 2：**

```
输入: "cbbd"
输出: "bb"
```

---

```java
class Solution {
    public String longestPalindrome(String s) {
        // 边界值设置
        if (s == null || s.length() < 2) {
            return s;
        }
        // 最大长度和起始点
        int maxLength = 1;
        int start = 0;
        int length = s.length();
        // 记录 j 到 i 是否是回文
        boolean[][] dp = new boolean[length][length];
        // 两层遍历。找到所有 j 到 i 为回文的值
        for (int i = 1; i < length; i++) {
            for (int j = 0; j < i; j++) {
                // 如果 si = sj 则进一步判断是否是回文
                if (s.charAt(i) == s.charAt(j)) {
                    // 如果长度小于 3 则一定是回文
                    if (i - j < 3) {
                        dp[j][i] = true;
                    } else {
                        // 否则根据长度减2的内部一位的长度是否是回文判断当前是否是回文
                        dp[j][i] = dp[j+1][i-1];
                    }
                } else {
                    // 如果不等则直接不是回文
                    dp[j][i] = false;
                }
                
                // 如果是回文
                if (dp[j][i]) {
                    // 计算当前的长度
                    int curLength = i - j + 1;
                    // 如果超出最大值则更新最大值和初始值
                    if (curLength > maxLength) {
                        maxLength = curLength;
                        start = j;
                    }
                }
            }
        }
        // 返回结果
        return s.substring(start, start + maxLength);
    }
}
```

#### 字符串转换整数 (atoi)

请你来实现一个 `atoi` 函数，使其能将字符串转换成整数。

首先，该函数会根据需要丢弃无用的开头空格字符，直到寻找到第一个非空格的字符为止。接下来的转化规则如下：

- 如果第一个非空字符为正或者负号时，则将该符号与之后面尽可能多的连续数字字符组合起来，形成一个有符号整数。
- 假如第一个非空字符是数字，则直接将其与之后连续的数字字符组合起来，形成一个整数。
- 该字符串在有效的整数部分之后也可能会存在多余的字符，那么这些字符可以被忽略，它们对函数不应该造成影响。

注意：假如该字符串中的第一个非空格字符不是一个有效整数字符、字符串为空或字符串仅包含空白字符时，则你的函数不需要进行转换，即无法进行有效转换。

在任何情况下，若函数不能进行有效的转换时，请返回 0 。

**提示：**

- 本题中的空白字符只包括空格字符 `' '` 。
- 假设我们的环境只能存储 32 位大小的有符号整数，那么其数值范围为 [−231, 231 − 1]。如果数值超过这个范围，请返回  INT_MAX (231 − 1) 或 INT_MIN (−231) 。

 

**示例 1:**

```
输入: "42"
输出: 42
```

**示例 2:**

```
输入: "   -42"
输出: -42
解释: 第一个非空白字符为 '-', 它是一个负号。
     我们尽可能将负号与后面所有连续出现的数字组合起来，最后得到 -42 。
```

**示例 3:**

```
输入: "4193 with words"
输出: 4193
解释: 转换截止于数字 '3' ，因为它的下一个字符不为数字。
```

**示例 4:**

```
输入: "words and 987"
输出: 0
解释: 第一个非空字符是 'w', 但它不是数字或正、负号。
     因此无法执行有效的转换。
```

**示例 5:**

```
输入: "-91283472332"
输出: -2147483648
解释: 数字 "-91283472332" 超过 32 位有符号整数范围。 
     因此返回 INT_MIN (−231) 。
```

---

```java
class Solution {
    public int myAtoi(String str) {
        int len = str.length();
        int index = 0;
        // 去掉前面的空格
        while (index < len) {
            if (str.charAt(index) != ' ') {
                break;
            }
            index += 1;
        }
        
        // 空值判断
        if (index == len) {
            return 0;
        }
        
        // 判断是否是符号开头
        int sign = 1;
        if (str.charAt(index) == '-') {
            sign = -1;
            index += 1;
        } else if (str.charAt(index) == '+') {
            index += 1;
        }
        
        int sum = 0;
        while (index < len) {
            int cc = str.charAt(index);
            // 结尾控制
            if (cc < '0' || cc > '9') {
                break;
            }
            // 判断是否超出最大值
            if (sum > Integer.MAX_VALUE / 10 || (sum == Integer.MAX_VALUE / 10 && (cc - '0') > Integer.MAX_VALUE % 10)) {
                return Integer.MAX_VALUE;
            }
            // 判断是否超出最小值
            if (sum < Integer.MIN_VALUE / 10 || (sum == Integer.MIN_VALUE / 10 && (cc - '0') > -(Integer.MIN_VALUE % 10))) {
                return Integer.MIN_VALUE;
            }
            // 计算结果
            sum = sum * 10 + sign * (cc - '0');
            index += 1;
        }
        return sum;
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
        if (strs == null || strs.length == 0) {
            return "";
        }
        // 把第一个作为最长的公共字符串
        String prefix = strs[0];
        for (int i = 1; i < strs.length; i++) {
            // 如果不是公共字串则把将 prefix - 1
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

---

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
        for (int i = 0; i < nums.length; i++) {
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

####  最接近的三数之和

给定一个包括 *n* 个整数的数组 `nums` 和 一个目标值 `target`。找出 `nums` 中的三个整数，使得它们的和与 `target` 最接近。返回这三个数的和。假定每组输入只存在唯一答案。

```
例如，给定数组 nums = [-1，2，1，-4], 和 target = 1.

与 target 最接近的三个数的和为 2. (-1 + 2 + 1 = 2).
```

---

```java
class Solution {
    public int threeSumClosest(int[] nums, int target) {
        // 先排序
        Arrays.sort(nums);
        // 把前三个值作为默认结果
        int ans = nums[0] + nums[1] + nums[2];
        // 遍历
        for (int i = 0; i < nums.length; i++) {
            // i 是第一个数字，left 是第二个，right 是第三个
            int left = i + 1;
            int right = nums.length - 1;
            while (left < right) {
                // 获取结果并判断是否和 target 更接近如何是则更新 ans
                int sum = nums[i] + nums[left] + nums[right];
                if (Math.abs(target - sum) < Math.abs(target - ans)) {
                    ans = sum;
                }
                if (sum > target) {
                    right -= 1;
                } else if (sum < target) {
                    left += 1;
                } else {
                    return ans;
                }
            }
        }
        return ans;
    }
}
```

#### 有效的括号

给定一个只包括 `'('`，`')'`，`'{'`，`'}'`，`'['`，`']'` 的字符串，判断字符串是否有效。

有效字符串需满足：

1. 左括号必须用相同类型的右括号闭合。
2. 左括号必须以正确的顺序闭合。

注意空字符串可被认为是有效字符串。

**示例 1:**

```
输入: "()"
输出: true
```

**示例 2:**

```
输入: "()[]{}"
输出: true
```

**示例 3:**

```
输入: "(]"
输出: false
```

**示例 4:**

```
输入: "([)]"
输出: false
```

**示例 5:**

```
输入: "{[]}"
输出: true
```

---

```java
class Solution {
    public boolean isValid(String s) {
        Stack<Character> stack = new Stack<>();
        for (int i = 0; i < s.length(); i++) {
            char cc = s.charAt(i);
            // 若是左括号则直接存入
            if (cc == '(' || cc == '[' || cc == '{') {
                stack.push(cc);
            }
            // 如果是右括号则弹出并匹配
            if (stack.isEmpty()) {
                return false;
            }
            if (cc == ')' && stack.pop() != '(') return false;
            if (cc == ']' && stack.pop() != '[') return false;
            if (cc == '}' && stack.pop() != '{') return false;
        }
        return stack.isEmpty();
    }
}
```

#### 删除排序数组中的重复项

给定一个排序数组，你需要在**[ 原地](http://baike.baidu.com/item/原地算法)** 删除重复出现的元素，使得每个元素只出现一次，返回移除后数组的新长度。

不要使用额外的数组空间，你必须在 **[原地 ](https://baike.baidu.com/item/原地算法)修改输入数组** 并在使用 O(1) 额外空间的条件下完成。

 

**示例 1:**

```
给定数组 nums = [1,1,2], 

函数应该返回新的长度 2, 并且原数组 nums 的前两个元素被修改为 1, 2。 

你不需要考虑数组中超出新长度后面的元素。
```

**示例 2:**

```
给定 nums = [0,0,1,1,1,2,2,3,3,4],

函数应该返回新的长度 5, 并且原数组 nums 的前五个元素被修改为 0, 1, 2, 3, 4。

你不需要考虑数组中超出新长度后面的元素。
```

 

**说明:**

为什么返回数值是整数，但输出的答案是数组呢?

请注意，输入数组是以**「引用」**方式传递的，这意味着在函数里修改输入数组对于调用者是可见的。

你可以想象内部操作如下:

```java
// nums 是以“引用”方式传递的。也就是说，不对实参做任何拷贝
int len = removeDuplicates(nums);

// 在函数里修改输入数组对于调用者是可见的。
// 根据你的函数返回的长度, 它会打印出数组中该长度范围内的所有元素。
for (int i = 0; i < len; i++) {
    print(nums[i]);
}
```

---

```java
class Solution {
    public int removeDuplicates(int[] nums) {
        int first = 0;
        for (int i = 1; i < nums.length; i++) {
            // 如果不相等的时候才移动 first 的位置
            if (nums[i] != nums[first]) {
                nums[++first] = nums[i];
            }
        }
        return first + 1;
    }
}
```

#### 盛最多水的容器

给你 *n* 个非负整数 *a*1，*a*2，...，*a*n，每个数代表坐标中的一个点 (*i*, *ai*) 。在坐标内画 *n* 条垂直线，垂直线 *i* 的两个端点分别为 (*i*, *ai*) 和 (*i*, 0)。找出其中的两条线，使得它们与 *x* 轴共同构成的容器可以容纳最多的水。

**说明：**你不能倾斜容器，且 *n* 的值至少为 2。

 

![img](https://aliyun-lc-upload.oss-cn-hangzhou.aliyuncs.com/aliyun-lc-upload/uploads/2018/07/25/question_11.jpg)

图中垂直线代表输入数组 [1,8,6,2,5,4,8,3,7]。在此情况下，容器能够容纳水（表示为蓝色部分）的最大值为 49。

 

**示例：**

```
输入：[1,8,6,2,5,4,8,3,7]
输出：49
```

---

```java
class Solution {
    public int maxArea(int[] height) {
        int left = 0;
        int right = height.length - 1;
        // 初始最大值
        int maxa = 0;
        // 从两端遍历
        while(left < right) {
            // 找到短板
            int minh = Math.min(height[left], height[right]);
            // 更新最大值
            maxa = Math.max(maxa, (right - left) * minh);
            // 判断是左边移动还是右边移动
            if (height[left] < height[right]) {
                left += 1;
            } else {
                right -= 1;
            }
        }
        return maxa;
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

---


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

#### 反转字符串

编写一个函数，其作用是将输入的字符串反转过来。输入字符串以字符数组 `char[]` 的形式给出。

不要给另外的数组分配额外的空间，你必须**[原地](https://baike.baidu.com/item/原地算法)修改输入数组**、使用 O(1) 的额外空间解决这一问题。

你可以假设数组中的所有字符都是 [ASCII](https://baike.baidu.com/item/ASCII) 码表中的可打印字符。

 

**示例 1：**

```
输入：["h","e","l","l","o"]
输出：["o","l","l","e","h"]
```

**示例 2：**

```
输入：["H","a","n","n","a","h"]
输出：["h","a","n","n","a","H"]
```

---

```java
class Solution {
    public void reverseString(char[] s) {
        int lo = 0;
        int hi = s.length - 1;
        while (lo < hi) {
            swap(s, lo++, hi--);;
        }
    }
    
    private void swap(char[] s, int i, int j) {
        char t = s[i];
        s[i] = s[j];
        s[j] = t;
    }
}
```

#### 反转字符串中的单词 III

给定一个字符串，你需要反转字符串中每个单词的字符顺序，同时仍保留空格和单词的初始顺序。

**示例 1:**

```
输入: "Let's take LeetCode contest"
输出: "s'teL ekat edoCteeL tsetnoc" 
```

***\**\*\*\*注意：\*\*\*\*\****在字符串中，每个单词由单个空格分隔，并且字符串中不会有任何额外的空格。

---

```java
class Solution {
    public String reverseWords(String s) {
        if (s == null || s.length() == 0) {
            return s;
        }
        String[] arr = s.split(" ");
        StringBuilder res = new StringBuilder();
        for (int i = 0; i < arr.length; i++) {
            if (i == 0) {
                res.append(new StringBuilder(arr[i]).reverse());
            } else {
                 res.append(" ").append(new StringBuilder(arr[i]).reverse());
            }
        }
        return res.toString();
    }
}
```

#### 除自身以外数组的乘积

给你一个长度为 *n* 的整数数组 `nums`，其中 *n* > 1，返回输出数组 `output` ，其中 `output[i]` 等于 `nums` 中除 `nums[i]` 之外其余各元素的乘积。

 

**示例:**

```
输入: [1,2,3,4]
输出: [24,12,8,6]
```

 

**提示：**题目数据保证数组之中任意元素的全部前缀元素和后缀（甚至是整个数组）的乘积都在 32 位整数范围内。

**说明:** 请**不要使用除法，**且在 O(*n*) 时间复杂度内完成此题。

**进阶：**
你可以在常数空间复杂度内完成这个题目吗？（ 出于对空间复杂度分析的目的，输出数组**不被视为**额外空间。）

---

```java
class Solution {
    public int[] productExceptSelf(int[] nums) {
        int[] target = new int[nums.length];
        // 将值全部默认为 1 方便后续处理
        Arrays.fill(target, 1);
        for (int i = 1, left = 1; i < nums.length; i++) {
            // 前 i - 1 位的乘积
            left *= nums[i - 1];
            target[i] *= left;
        }
        for (int i = nums.length - 2, right = 1; i >= 0; i--) {
            // 后 n - i 位的乘积
            right *= nums[i + 1];
            target[i] *= right;
        }
        return target;
    }
}
```

#### 存在重复元素

给定一个整数数组，判断是否存在重复元素。

如果任意一值在数组中出现至少两次，函数返回 `true` 。如果数组中每个元素都不相同，则返回 `false` 。

 

**示例 1:**

```
输入: [1,2,3,1]
输出: true
```

**示例 2:**

```
输入: [1,2,3,4]
输出: false
```

**示例 3:**

```
输入: [1,1,1,3,3,4,3,2,4,2]
输出: true
```

---

```java
class Solution {
    public boolean containsDuplicate(int[] nums) {
        Set<Integer> set = new HashSet<>();
        for (int num : nums) {
            set.add(num);
        }
        return set.size() != nums.length;
    }
}
```

#### 螺旋矩阵

给定一个包含 *m* x *n* 个元素的矩阵（*m* 行, *n* 列），请按照顺时针螺旋顺序，返回矩阵中的所有元素。

**示例 1:**

```
输入:
[
 [ 1, 2, 3 ],
 [ 4, 5, 6 ],
 [ 7, 8, 9 ]
]
输出: [1,2,3,6,9,8,7,4,5]
```

**示例 2:**

```
输入:
[
  [1, 2, 3, 4],
  [5, 6, 7, 8],
  [9,10,11,12]
]
输出: [1,2,3,4,8,12,11,10,9,5,6,7]
```

---

```java
class Solution {
    public List<Integer> spiralOrder(int[][] matrix) {
        List<Integer> res = new ArrayList<>();
        if (matrix.length == 0) {
            return res;
        }
      	// 注意不一定是正方形
        int rows = matrix.length;
        int cols = matrix[0].length;
        int[] rd = new int[]{0, 1, 0, -1};
        int[] cd = new int[]{1, 0, -1, 0};
        boolean[][] visited = new boolean[rows][cols];
        int ri = 0;
        int ci = 0;
        int di = 0;
        for (int i = 0; i < rows * cols; i++) {
            res.add(matrix[ri][ci]);
            visited[ri][ci] = true;
            int nr = ri + rd[di];
            int nc = ci + cd[di];
            if (nr >= 0 && nc >= 0 && nr < rows && nc < cols && !visited[nr][nc]) {
                ri = nr;
                ci = nc;
            } else {
                di = (di + 1) % 4;
                ri += rd[di];
                ci += cd[di];
            }
        }
        return res;
    }
}
```

#### 螺旋矩阵 II

给定一个正整数 *n*，生成一个包含 1 到 *n*2 所有元素，且元素按顺时针顺序螺旋排列的正方形矩阵。

**示例:**

```
输入: 3
输出:
[
 [ 1, 2, 3 ],
 [ 8, 9, 4 ],
 [ 7, 6, 5 ]
]
```

---

```java
class Solution {
    public int[][] generateMatrix(int n) {
        int[] rd = new int[]{0, 1, 0, -1};
        int[] cd = new int[]{1, 0, -1, 0};
        int[][] res = new int[n][n];
        int ri = 0;
        int ci = 0;
        int di = 0;
        for (int i = 1; i <= n * n; i++) {
            res[ri][ci] = i;
            int nr = ri + rd[di];
            int nc = ci + cd[di];
            if (nr >= 0 && nc >= 0 && nr < n && nc < n && res[nr][nc] == 0) {
                ri = nr;
                ci = nc;
            } else {
                di = (di + 1) % 4;
                ri += rd[di];
                ci += cd[di];
            }
        }
        return res;
    }
}
```

#### 合并两个有序数组

给你两个有序整数数组 *nums1* 和 *nums2*，请你将 *nums2* 合并到 *nums1* 中*，*使 *nums1* 成为一个有序数组。

 

**说明:**

- 初始化 *nums1* 和 *nums2* 的元素数量分别为 *m* 和 *n* 。
- 你可以假设 *nums1* 有足够的空间（空间大小大于或等于 *m + n*）来保存 *nums2* 中的元素。

 

**示例:**

```
输入:
nums1 = [1,2,3,0,0,0], m = 3
nums2 = [2,5,6],       n = 3

输出: [1,2,2,3,5,6]
```

---

```java
class Solution {
    public void merge(int[] nums1, int m, int[] nums2, int n) {
        int last = m + n - 1;
        m -= 1;
        n -= 1;
        while (m >= 0 || n >= 0) {
            if (m < 0) {
                nums1[last--] = nums2[n--];
            } else if (n < 0) {
                nums1[last--] = nums1[m--];
            } else if (nums1[m] < nums2[n]) {
                nums1[last--] = nums2[n--];
            } else {
                nums1[last--] = nums1[m--];
            }
        }
    }
}
```



### 链表(9)

####  反转链表

反转一个单链表。

**示例:**

```
输入: 1->2->3->4->5->NULL
输出: 5->4->3->2->1->NULL
```

**进阶:**
你可以迭代或递归地反转链表。你能否用两种方法解决这道题？

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    public ListNode reverseList(ListNode head) {
        if (head == null || head.next == null) {
            return head;
        }
        ListNode next = head.next;
        ListNode newHead = reverseList(next);
        next.next = head;
        head.next = null;
        return newHead;
    }
}
```

#### 两两交换链表中的节点

给定一个链表，两两交换其中相邻的节点，并返回交换后的链表。

**你不能只是单纯的改变节点内部的值**，而是需要实际的进行节点交换。

 

示例:

```
给定 1->2->3->4, 你应该返回 2->1->4->3.
```

---

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    public ListNode swapPairs(ListNode head) {
        if (head == null || head.next == null) {
            return head;
        }
        ListNode next = head.next;
        head.next = swapPairs(next.next);
        next.next = head;
        return next;
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

---

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        ListNode dummyNode = new ListNode(-1);
        ListNode tempHead = dummyNode;
        int carry = 0;
        while (l1 != null || l2 != null) {
            int v1 = l1 != null ? l1.val : 0;
            int v2 = l2 != null ? l2.val : 0;
            // 计算值
            int sum = v1 + v2 + carry;
            // 设置节点
            tempHead.next = new ListNode(sum % 10);
            // 计算进位
            carry = sum / 10;
            tempHead = tempHead.next;
            l1 = l1 != null ? l1.next : l1;
            l2 = l2 != null ? l2.next : l2;
        }
        if (carry != 0) {
            tempHead.next = new ListNode(carry);
        }
        return dummyNode.next;
    }
}
```

#### 合并两个有序链表

将两个升序链表合并为一个新的升序链表并返回。新链表是通过拼接给定的两个链表的所有节点组成的。 

**示例：**

```
输入：1->2->4, 1->3->4
输出：1->1->2->3->4->4
```

---

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        if (l1 == null) return l2;
        if (l2 == null) return l1;
        if (l1.val < l2.val) {
            l1.next = mergeTwoLists(l1.next, l2);
            return l1;
        }
        l2.next = mergeTwoLists(l1, l2.next);
        return l2;
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

---

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
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
        // 分治法，两个两个的进行 merge
        int mid = lo + (hi - lo) / 2;
        ListNode l1 = merge(lists, lo, mid);
        ListNode l2 = merge(lists, mid + 1, hi);
        return mergeTwoList(l1, l2);
        
    }
    
    private ListNode mergeTwoList(ListNode l1, ListNode l2) {
        if (l1 == null) return l2;
        if (l2 == null) return l1;
        if (l1.val < l2.val) {
            l1.next = mergeTwoList(l1.next, l2);
            return l1;
        }
        l2.next = mergeTwoList(l1, l2.next);
        return l2;
    }
}
```

#### 旋转链表

给定一个链表，旋转链表，将链表每个节点向右移动 *k* 个位置，其中 *k* 是非负数。

**示例 1:**

```
输入: 1->2->3->4->5->NULL, k = 2
输出: 4->5->1->2->3->NULL
解释:
向右旋转 1 步: 5->1->2->3->4->NULL
向右旋转 2 步: 4->5->1->2->3->NULL
```

**示例 2:**

```
输入: 0->1->2->NULL, k = 4
输出: 2->0->1->NULL
解释:
向右旋转 1 步: 2->0->1->NULL
向右旋转 2 步: 1->2->0->NULL
向右旋转 3 步: 0->1->2->NULL
向右旋转 4 步: 2->0->1->NULL
```

---

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    public ListNode rotateRight(ListNode head, int k) {
        if (head == null || k == 0) {
            return head;
        }
        int cnt = 1;
        ListNode dummyHead = head;
        // 计算总数量
        while(head.next != null) {
            head = head.next;
            cnt += 1;
        }
        
        // 此时 head 为末尾的一个，next 指向原 head 形成闭环
        head.next = dummyHead;
        // 计算 target k 避免超出长度
        int tk = cnt - k % cnt;
        // 新下标
        int cc = 1;
        // 从原 head 开始计算
        while (dummyHead != null) {
            // 找到确认值
            if (cc == tk) {
                // 断开环，target 节点的下一个节点是新的 head
                ListNode newHead = dummyHead.next;
                // 断开环
                dummyHead.next = null;
                // 然后新 head
                return newHead;
            }
            cc += 1;
            // 下一个
            dummyHead = dummyHead.next;
        }
        return head;
        
    }
}
```

环形链表

给定一个链表，判断链表中是否有环。

为了表示给定链表中的环，我们使用整数 `pos` 来表示链表尾连接到链表中的位置（索引从 0 开始）。 如果 `pos` 是 `-1`，则在该链表中没有环。

 

**示例 1：**

```
输入：head = [3,2,0,-4], pos = 1
输出：true
解释：链表中有一个环，其尾部连接到第二个节点。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist.png)

**示例 2：**

```
输入：head = [1,2], pos = 0
输出：true
解释：链表中有一个环，其尾部连接到第一个节点。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist_test2.png)

**示例 3：**

```
输入：head = [1], pos = -1
输出：false
解释：链表中没有环。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist_test3.png)

 

**进阶：**

你能用 *O(1)*（即，常量）内存解决此问题吗？

---

```java
/**
 * Definition for singly-linked list.
 * class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) {
 *         val = x;
 *         next = null;
 *     }
 * }
 */
public class Solution {
    public boolean hasCycle(ListNode head) {
        if (head == null || head.next == null) {
            return false;
        }
        // 快慢指针
        ListNode fast = head;
        ListNode slow = head;
        while (fast != null && fast.next != null) {
            fast = fast.next.next;
            slow = slow.next;
            if (fast == slow) {
                return true;
            }
        }
        return false;
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

---

```java
/**
 * Definition for singly-linked list.
 * class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) {
 *         val = x;
 *         next = null;
 *     }
 * }
 */
public class Solution {
    public ListNode detectCycle(ListNode head) {
        if (head == null || head.next == null) {
            return null;
        }
        // 获取环节点
        ListNode cycNode = getCycNode(head);
        // 如果为空则不存在环
        if (cycNode == null) {
            return null;
        }
        // 从头开始找，找到 target node
        while(head != cycNode) {
            head = head.next;
            cycNode = cycNode.next;
        }
        return head;
    }
    
    private ListNode getCycNode(ListNode head) {
        ListNode fast = head;
        ListNode slow = head;
        while (fast != null && fast.next != null) {
            fast = fast.next.next;
            slow = slow.next;
            if (fast == slow) {
                return fast;
            }
        }
        return null;
    }
}
```

#### 相交链表

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

---

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) {
 *         val = x;
 *         next = null;
 *     }
 * }
 */
public class Solution {
    public ListNode getIntersectionNode(ListNode headA, ListNode headB) {
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

#### 删除链表中的节点

请编写一个函数，使其可以删除某个链表中给定的（非末尾）节点，你将只被给定要求被删除的节点。

现有一个链表 -- head = [4,5,1,9]，它可以表示为:

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2019/01/19/237_example.png)

 

**示例 1:**

```
输入: head = [4,5,1,9], node = 5
输出: [4,1,9]
解释: 给定你链表中值为 5 的第二个节点，那么在调用了你的函数之后，该链表应变为 4 -> 1 -> 9.
```

**示例 2:**

```
输入: head = [4,5,1,9], node = 1
输出: [4,5,9]
解释: 给定你链表中值为 1 的第三个节点，那么在调用了你的函数之后，该链表应变为 4 -> 5 -> 9.
```

 

**说明:**

- 链表至少包含两个节点。
- 链表中所有节点的值都是唯一的。
- 给定的节点为非末尾节点并且一定是链表中的一个有效节点。
- 不要从你的函数中返回任何结果。

---

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    public void deleteNode(ListNode node) {
        node.val = node.next.val;
        node.next = node.next.next;
    }
}
```

### 数学(5)

#### 整数反转

给出一个 32 位的有符号整数，你需要将这个整数中每位上的数字进行反转。

**示例 1:**

```
输入: 123
输出: 321
```

 **示例 2:**

```
输入: -123
输出: -321
```

**示例 3:**

```
输入: 120
输出: 21
```

**注意:**

假设我们的环境只能存储得下 32 位的有符号整数，则其数值范围为 [−231, 231 − 1]。请根据这个假设，如果反转后整数溢出那么就返回 0。

---

```java
class Solution {
    public int reverse(int x) {
        int res = 0;
        while (x != 0) {
            int cv = x % 10;
            // 判断溢出！！！
            if (res > Integer.MAX_VALUE / 10 || (res == Integer.MAX_VALUE / 10 && cv > Integer.MAX_VALUE % 10)) {
                return 0;
            }
            if (res < Integer.MIN_VALUE / 10 || (res == Integer.MIN_VALUE / 10 && cv < Integer.MIN_VALUE % 10)) {
                return 0;
            }
            res = res * 10 + cv;
            x /= 10;
        }
        return res;
    }
}
```

#### 回文数

判断一个整数是否是回文数。回文数是指正序（从左向右）和倒序（从右向左）读都是一样的整数。

**示例 1:**

```
输入: 121
输出: true
```

**示例 2:**

```
输入: -121
输出: false
解释: 从左向右读, 为 -121 。 从右向左读, 为 121- 。因此它不是一个回文数。
```

**示例 3:**

```
输入: 10
输出: false
解释: 从右向左读, 为 01 。因此它不是一个回文数。
```

**进阶:**

你能不将整数转为字符串来解决这个问题吗？

---

```java
class Solution {
    public boolean isPalindrome(int x) {
        if (x < 0) {
            return false;
        }
        int nv = 0;
        int cv = x;
        while (cv != 0) {
            nv = nv * 10 + cv % 10;
            cv /= 10;
        }
        return nv == x;
    }
}
```

#### 只出现一次的数字

给定一个**非空**整数数组，除了某个元素只出现一次以外，其余每个元素均出现两次。找出那个只出现了一次的元素。

**说明：**

你的算法应该具有线性时间复杂度。 你可以不使用额外空间来实现吗？

**示例 1:**

```
输入: [2,2,1]
输出: 1
```

**示例 2:**

```
输入: [4,1,2,1,2]
输出: 4
```

---

```java
class Solution {
    public int singleNumber(int[] nums) {
        int res = nums[0];
        for (int i = 1; i < nums.length; i++) {
            res ^= nums[i];
        }
        return res;
    }
}
```

#### 多数元素

给定一个大小为 *n* 的数组，找到其中的多数元素。多数元素是指在数组中出现次数**大于** `⌊ n/2 ⌋` 的元素。

你可以假设数组是非空的，并且给定的数组总是存在多数元素。

 

**示例 1:**

```
输入: [3,2,3]
输出: 3
```

**示例 2:**

```
输入: [2,2,1,1,1,2,2]
输出: 2
```

---

```java
class Solution {
    public int majorityElement(int[] nums) {
        int res = nums[0];
        int cnt = 1;
      	// 超过 1/2 的元素
        for (int i = 1; i < nums.length; i++) {
            if (nums[i] == res) {
                cnt += 1;
            } else {
                cnt -= 1;
                if (cnt <= 0) {
                    res = nums[i];
                    cnt = 1;
                }
            }
        }
        return res;
    }
}
```

#### 2的幂

给定一个整数，编写一个函数来判断它是否是 2 的幂次方。

**示例 1:**

```
输入: 1
输出: true
解释: 20 = 1
```

**示例 2:**

```
输入: 16
输出: true
解释: 24 = 16
```

**示例 3:**

```
输入: 218
输出: false
```

---

```java
class Solution {
    public boolean isPowerOfTwo(int n) {
        return n > 0 && (n & (n - 1)) == 0; 
    }
}
```

### 搜索排序(8)

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

---

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

---

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

---

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

#### 二叉搜索树中第K小的元素

给定一个二叉搜索树，编写一个函数 `kthSmallest` 来查找其中第 **k** 个最小的元素。

**说明：**
你可以假设 k 总是有效的，1 ≤ k ≤ 二叉搜索树元素个数。

**示例 1:**

```
输入: root = [3,1,4,null,2], k = 1
   3
  / \
 1   4
  \
   2
输出: 1
```

**示例 2:**

```
输入: root = [5,3,6,2,4,null,null,1], k = 3
       5
      / \
     3   6
    / \
   2   4
  /
 1
输出: 3
```

**进阶：**
如果二叉搜索树经常被修改（插入/删除操作）并且你需要频繁地查找第 k 小的值，你将如何优化 `kthSmallest` 函数？

---

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    private int cnt = 0;
    private TreeNode target = null;
    public int kthSmallest(TreeNode root, int k) {
        inOrder(root, k);
        return target.val;
    }
    private void inOrder(TreeNode root, int k) {
        if (root == null) {
            return;
        }
        inOrder(root.left, k);
        cnt += 1;
        if (cnt == k) {
            target = root;
        }
        inOrder(root.right, k);
    }
}
```

#### 二叉树的最大深度

给定一个二叉树，找出其最大深度。

二叉树的深度为根节点到最远叶子节点的最长路径上的节点数。

**说明:** 叶子节点是指没有子节点的节点。

**示例：**
给定二叉树 `[3,9,20,null,null,15,7]`，

```
    3
   / \
  9  20
    /  \
   15   7
```

返回它的最大深度 3 。

---

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    public int maxDepth(TreeNode root) {
        if (root == null) {
            return 0;
        }
        return Math.max(maxDepth(root.left), maxDepth(root.right)) + 1;
    }
}
```

#### 二叉树中的最大路径和

给定一个**非空**二叉树，返回其最大路径和。

本题中，路径被定义为一条从树中任意节点出发，达到任意节点的序列。该路径**至少包含一个**节点，且不一定经过根节点。

**示例 1:**

```
输入: [1,2,3]

       1
      / \
     2   3

输出: 6
```

**示例 2:**

```
输入: [-10,9,20,null,null,15,7]

   -10
   / \
  9  20
    /  \
   15   7

输出: 42
```

---

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    private int res = Integer.MIN_VALUE;
    public int maxPathSum(TreeNode root) {
        retrieveVal(root);
        return res;
    }
    
    private int retrieveVal(TreeNode root) {
        if (root == null) {
            return 0;
        }
        // 左边的值
        int left = Math.max(retrieveVal(root.left), 0);
        // 右边的值
        int right = Math.max(retrieveVal(root.right), 0);
        // 更新最大值
        res = Math.max((left + right + root.val), res);
        return root.val + Math.max(left, right);
    }
}
```

#### 二叉搜索树的最近公共祖先

给定一个二叉搜索树, 找到该树中两个指定节点的最近公共祖先。

[百度百科](https://baike.baidu.com/item/最近公共祖先/8918834?fr=aladdin)中最近公共祖先的定义为：“对于有根树 T 的两个结点 p、q，最近公共祖先表示为一个结点 x，满足 x 是 p、q 的祖先且 x 的深度尽可能大（**一个节点也可以是它自己的祖先**）。”

例如，给定如下二叉搜索树: root = [6,2,8,0,4,7,9,null,null,3,5]

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/14/binarysearchtree_improved.png)

 

**示例 1:**

```
输入: root = [6,2,8,0,4,7,9,null,null,3,5], p = 2, q = 8
输出: 6 
解释: 节点 2 和节点 8 的最近公共祖先是 6。
```

**示例 2:**

```
输入: root = [6,2,8,0,4,7,9,null,null,3,5], p = 2, q = 4
输出: 2
解释: 节点 2 和节点 4 的最近公共祖先是 2, 因为根据定义最近公共祖先节点可以为节点本身。
```

 

**说明:**

- 所有节点的值都是唯一的。
- p、q 为不同节点且均存在于给定的二叉搜索树中。

---

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        if (root == null) {
            return root;
        }
        if (root.val < p.val && root.val < q.val) {
            return lowestCommonAncestor(root.right, p, q);
        } else if (root.val > p.val && root.val > q.val) {
            return lowestCommonAncestor(root.left, p, q);
        }
        return root;
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

---

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
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

### 回溯(4)

#### 括号生成

数字 *n* 代表生成括号的对数，请你设计一个函数，用于能够生成所有可能的并且 **有效的** 括号组合。

 

**示例：**

```
输入：n = 3
输出：[
       "((()))",
       "(()())",
       "(())()",
       "()(())",
       "()()()"
     ]
```

---

```java
class Solution {
    public List<String> generateParenthesis(int n) {
        List<String> res = new ArrayList<>();
        bc(res, "", 0, 0, n);
        return res;
    }
    
    private void bc(List<String> res, String str, int open, int close, int max) {
        if (open + close == max * 2) {
            res.add(str);
            return;
        } 
        
        if (open < max) {
            bc(res, str + "(", open + 1, close, max);
        }
        if (close < open) {
            bc(res, str + ")", open, close + 1, max);
        }
    }
}
```

#### 子集

给定一组**不含重复元素**的整数数组 *nums*，返回该数组所有可能的子集（幂集）。

**说明：**解集不能包含重复的子集。

**示例:**

```
输入: nums = [1,2,3]
输出:
[
  [3],
  [1],
  [2],
  [1,2,3],
  [1,3],
  [2,3],
  [1,2],
  []
]
```

---

```java
class Solution {
    public List<List<Integer>> subsets(int[] nums) {
        List<List<Integer>> res = new ArrayList<>();
        for (int i = 0; i <= nums.length; i++) {
            bc(0, res, new ArrayList<>(), i, nums);
        }
        return res;
    }
    
    private void bc(int start, List<List<Integer>> res, List<Integer> tmp, int size, int[] nums) {
        if (tmp.size() == size) {
            res.add(new ArrayList<>(tmp));
            return;
        }
        for (int i = start; i < nums.length; i++) {
            tmp.add(nums[i]);
            bc(i + 1, res, tmp, size, nums);
            tmp.remove(tmp.size() - 1);
        }
    }
}
```

#### 全排列

给定一个 **没有重复** 数字的序列，返回其所有可能的全排列。

**示例:**

```
输入: [1,2,3]
输出:
[
  [1,2,3],
  [1,3,2],
  [2,1,3],
  [2,3,1],
  [3,1,2],
  [3,2,1]
]
```

---

```java
class Solution {
    public List<List<Integer>> permute(int[] nums) {
        List<List<Integer>> res = new ArrayList<>();
        bc(res, new ArrayList<>(), nums, new boolean[nums.length]);
        return res;
    }
    
    private void bc(List<List<Integer>> res, List<Integer> tmp, int[] nums, boolean[] visited) {
        if (tmp.size() == nums.length) {
            res.add(new ArrayList<>(tmp));
            return;
        }
        for (int i = 0; i < nums.length; i++) {
            if (visited[i]) {
                continue;
            }
            visited[i] = true;
            tmp.add(nums[i]);
            bc(res, tmp, nums, visited);
            visited[i] = false;
            tmp.remove(tmp.size() - 1);
        }
    }
}
```

#### 格雷编码

格雷编码是一个二进制数字系统，在该系统中，两个连续的数值仅有一个位数的差异。

给定一个代表编码总位数的非负整数 *n*，打印其格雷编码序列。即使有多个不同答案，你也只需要返回其中一种。

格雷编码序列必须以 0 开头。

 

**示例 1:**

```
输入: 2
输出: [0,1,3,2]
解释:
00 - 0
01 - 1
11 - 3
10 - 2

对于给定的 n，其格雷编码序列并不唯一。
例如，[0,2,3,1] 也是一个有效的格雷编码序列。

00 - 0
10 - 2
11 - 3
01 - 1
```

**示例 2:**

```
输入: 0
输出: [0]
解释: 我们定义格雷编码序列必须以 0 开头。
     给定编码总位数为 n 的格雷编码序列，其长度为 2n。当 n = 0 时，长度为 20 = 1。
     因此，当 n = 0 时，其格雷编码序列为 [0]。
```

---

```java
class Solution {
    public List<Integer> grayCode(int n) {
        List<Integer> res = new ArrayList<>();
        res.add(0);
        int head = 1;
        for (int i = 0; i < n; i++) {
            for (int j = res.size() - 1; j >= 0; j--) {
                res.add(head + res.get(j));
            }
            head <<= 1;
        }
        return res;
    }
}
```



### 动态规划(6)

#### 爬楼梯

假设你正在爬楼梯。需要 *n* 阶你才能到达楼顶。

每次你可以爬 1 或 2 个台阶。你有多少种不同的方法可以爬到楼顶呢？

**注意：**给定 *n* 是一个正整数。

**示例 1：**

```
输入： 2
输出： 2
解释： 有两种方法可以爬到楼顶。
1.  1 阶 + 1 阶
2.  2 阶
```

**示例 2：**

```
输入： 3
输出： 3
解释： 有三种方法可以爬到楼顶。
1.  1 阶 + 1 阶 + 1 阶
2.  1 阶 + 2 阶
3.  2 阶 + 1 阶
```

-----

```java
class Solution {
    public int climbStairs(int n) {
        if (n < 3) {
            return n;
        }
        int pre1 = 2;
        int pre2 = 1;
        for (int i = 2; i < n; i++) {
            int tmp = pre1 + pre2;
            pre2 = pre1;
            pre1 = tmp;
        }
        return pre1;
    }
}
```

#### 最大子序和

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
            sum = (sum <= 0 ? num : sum + num);
            gSum = Math.max(sum, gSum);
        }
        return gSum;
    }
}
```

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

---

```java
class Solution {
    public int maxProfit(int[] prices) {
        if (prices == null || prices.length == 0) {
            return 0;
        }
        int mv = prices[0];
        int profit = 0;
        for (int i = 1; i < prices.length; i++) {
            mv = Math.min(mv, prices[i]);
            profit = Math.max(profit, prices[i] - mv);
        }
        return profit;
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

---

```java
class Solution {
    public int maxProfit(int[] prices) {
        int sum = 0;
        for (int i = 1; i < prices.length; i++) {
            if (prices[i] > prices[i - 1]) {
                sum += (prices[i] - prices[i - 1]);
            }
        }
        return sum;
    }
}
```

#### 不同路径

一个机器人位于一个 *m x n* 网格的左上角 （起始点在下图中标记为“Start” ）。

机器人每次只能向下或者向右移动一步。机器人试图达到网格的右下角（在下图中标记为“Finish”）。

问总共有多少条不同的路径？

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/10/22/robot_maze.png)

例如，上图是一个7 x 3 的网格。有多少可能的路径？

 

**示例 1:**

```
输入: m = 3, n = 2
输出: 3
解释:
从左上角开始，总共有 3 条路径可以到达右下角。
1. 向右 -> 向右 -> 向下
2. 向右 -> 向下 -> 向右
3. 向下 -> 向右 -> 向右
```

**示例 2:**

```
输入: m = 7, n = 3
输出: 28
```

 

**提示：**

- `1 <= m, n <= 100`
- 题目数据保证答案小于等于 `2 * 10 ^ 9`

---

```java
class Solution {
    public int uniquePaths(int m, int n) {
        int[] dp = new int[n];
        Arrays.fill(dp, 1);
        for (int i = 1; i < m; i++) {
            for (int j = 1; j < n; j++) {
                dp[j] += dp[j-1];
            }
        }
        return dp[n - 1];
    }
}
```

#### 子集

给定一组**不含重复元素**的整数数组 *nums*，返回该数组所有可能的子集（幂集）。

**说明：**解集不能包含重复的子集。

**示例:**

```
输入: nums = [1,2,3]
输出:
[
  [3],
  [1],
  [2],
  [1,2,3],
  [1,3],
  [2,3],
  [1,2],
  []
]
```

---

```java
class Solution {
    public List<List<Integer>> subsets(int[] nums) {
        List<List<Integer>> res = new ArrayList<>();
        for (int i = 0; i <= nums.length; i++) {
            bc(0, res, new ArrayList<>(), i, nums);
        }
        return res;
    }
    
    private void bc(int start, List<List<Integer>> res, List<Integer> tmp, int size, int[] nums) {
        if (tmp.size() == size) {
            res.add(new ArrayList<>(tmp));
            return;
        }
        for (int i = start; i < nums.length; i++) {
            tmp.add(nums[i]);
            bc(i + 1, res, tmp, size, nums);
            tmp.remove(tmp.size() - 1);
        }
    }
}
```

### 设计(2)

####  LRU缓存机制

运用你所掌握的数据结构，设计和实现一个 [LRU (最近最少使用) 缓存机制](https://baike.baidu.com/item/LRU)。它应该支持以下操作： 获取数据 `get` 和 写入数据 `put` 。

获取数据 `get(key)` - 如果密钥 (key) 存在于缓存中，则获取密钥的值（总是正数），否则返回 -1。
写入数据 `put(key, value)` - 如果密钥已经存在，则变更其数据值；如果密钥不存在，则插入该组「密钥/数据值」。当缓存容量达到上限时，它应该在写入新数据之前删除最久未使用的数据值，从而为新的数据值留出空间。

 

**进阶:**

你是否可以在 **O(1)** 时间复杂度内完成这两种操作？

 

**示例:**

```
LRUCache cache = new LRUCache( 2 /* 缓存容量 */ );

cache.put(1, 1);
cache.put(2, 2);
cache.get(1);       // 返回  1
cache.put(3, 3);    // 该操作会使得密钥 2 作废
cache.get(2);       // 返回 -1 (未找到)
cache.put(4, 4);    // 该操作会使得密钥 1 作废
cache.get(1);       // 返回 -1 (未找到)
cache.get(3);       // 返回  3
cache.get(4);       // 返回  4
```

---

```java
import java.util.Hashtable;

class LRUCache {
    private Hashtable<Integer, DLinkedNode> cached = new Hashtable<>();
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
        DLinkedNode node = cached.get(key);
        if (node == null) {
            return -1;
        }
        moveToHead(node);
        return node.value;
    }
    
    public void put(int key, int value) {
        DLinkedNode node = cached.get(key);
        if (node == null) {
            DLinkedNode newNode = new DLinkedNode();
            newNode.key = key;
            newNode.value = value;
            cached.put(key, newNode);
            addNode(newNode);
            ++size;
            if (size > capacity) {
                DLinkedNode tail = popTail();
                cached.remove(tail.key);
                --size;
            }
        } else {
            node.value = value;
            moveToHead(node);
        }
    }
}

/**
 * Your LRUCache object will be instantiated and called as such:
 * LRUCache obj = new LRUCache(capacity);
 * int param_1 = obj.get(key);
 * obj.put(key,value);
 */
```

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

---

```java
class MinStack {
    private Stack<Integer> dataStack;
    private Stack<Integer> minStack;
    
    /** initialize your data structure here. */
    public MinStack() {
        dataStack = new Stack<>();
        minStack = new Stack<>();
    }
    
    public void push(int x) {
        dataStack.push(x);
        if (minStack.isEmpty()) {
            minStack.push(x);
        } else {
            minStack.push(minStack.peek() < x ? minStack.peek() : x);
        }
    }
    
    public void pop() {
        if (!dataStack.isEmpty()) {
            dataStack.pop();
            minStack.pop();
        }
    }
    
    public int top() {
        return dataStack.isEmpty() ? 0 : dataStack.peek();
    }
    
    public int getMin() {
        return minStack.isEmpty() ? 0 : minStack.peek();
    }
}

/**
 * Your MinStack object will be instantiated and called as such:
 * MinStack obj = new MinStack();
 * obj.push(x);
 * obj.pop();
 * int param_3 = obj.top();
 * int param_4 = obj.getMin();
 */
```

