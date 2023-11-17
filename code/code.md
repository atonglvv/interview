**题目：**

**思路1：**

**复杂度：**

**代码：**

## 接雨水

**题目：** https://leetcode.cn/problems/trapping-rain-water/description/



**思路1：**动态规划

首先用两个数组，max_left [i] 代表第 i 列左边最高的墙的高度，max_right[i] 代表第 i 列右边最高的墙的高度。（一定要注意下，第 i 列左（右）边最高的墙，是不包括自身的，和 leetcode 上边的讲的有些不同）

对于 max_left我们其实可以这样求。

max_left [i] = Max(max_left [i-1],height[i-1])。它前边的墙的左边的最高高度和它前边的墙的高度选一个较大的，就是当前列左边最高的墙了。

对于 max_right我们可以这样求。

max_right[i] = Max(max_right[i+1],height[i+1]) 。它后边的墙的右边的最高高度和它后边的墙的高度选一个较大的，就是当前列右边最高的墙了。

**复杂度：**

时间复杂度：O（n）。

空间复杂度：O（n），用来保存每一列左边最高的墙和右边最高的墙。

**代码：**

```java
public int trap(int[] height) {
    int sum = 0;
    int[] max_left = new int[height.length];
    int[] max_right = new int[height.length];
    
    for (int i = 1; i < height.length - 1; i++) {
        max_left[i] = Math.max(max_left[i - 1], height[i - 1]);
    }
    for (int i = height.length - 2; i >= 0; i--) {
        max_right[i] = Math.max(max_right[i + 1], height[i + 1]);
    }
    for (int i = 1; i < height.length - 1; i++) {
        int min = Math.min(max_left[i], max_right[i]);
        if (min > height[i]) {
            sum = sum + (min - height[i]);
        }
    }
    return sum;
}
```



**思路2：**双指针

动态规划中，我们常常可以对空间复杂度进行进一步的优化。

例如这道题中，可以看到，max_left [ i ] 和 max_right [ i ] 数组中的元素我们其实只用一次，然后就再也不会用到了。所以我们可以不用数组，只用一个元素就行了。我们先改造下 max_left。

```java
public int trap(int[] height) {
    int sum = 0;
    int max_left = 0;
    int[] max_right = new int[height.length];
    for (int i = height.length - 2; i >= 0; i--) {
        max_right[i] = Math.max(max_right[i + 1], height[i + 1]);
    }
    for (int i = 1; i < height.length - 1; i++) {
        max_left = Math.max(max_left, height[i - 1]);
        int min = Math.min(max_left, max_right[i]);
        if (min > height[i]) {
            sum = sum + (min - height[i]);
        }
    }
    return sum;
}
```

我们成功将 max_left 数组去掉了。但是会发现我们不能同时把 max_right 的数组去掉，因为最后的 for 循环是从左到右遍历的，而 max_right 的更新是从右向左的。

所以这里要用到两个指针，left 和 right，从两个方向去遍历。

那么什么时候从左到右，什么时候从右到左呢？根据下边的代码的更新规则，我们可以知道

```java
max_left = Math.max(max_left, height[i - 1]);
```

height [ left - 1] 是可能成为 max_left 的变量， 同理，height [ right + 1 ] 是可能成为 right_max 的变量。

只要保证 height [ left - 1 ] < height [ right + 1 ] ，那么 max_left 就一定小于 max_right。

因为 max_left 是由 height [ left - 1] 更新过来的，而 height [ left - 1 ] 是小于 height [ right + 1] 的，而 height [ right + 1 ] 会更新 max_right，所以间接的得出 max_left 一定小于 max_right。

反之，我们就从右到左更。

**复杂度：**

时间复杂度：O（n）。

空间复杂度：O（1）。

**代码：**

```java
public int trap(int[] height) {
    int sum = 0;
    int max_left = 0;
    int max_right = 0;
    int left = 1;
    int right = height.length - 2; // 加右指针进去
    for (int i = 1; i < height.length - 1; i++) {
        //从左到右更
        if (height[left - 1] < height[right + 1]) {
            max_left = Math.max(max_left, height[left - 1]);
            int min = max_left;
            if (min > height[left]) {
                sum = sum + (min - height[left]);
            }
            left++;
        //从右到左更
        } else {
            max_right = Math.max(max_right, height[right + 1]);
            int min = max_right;
            if (min > height[right]) {
                sum = sum + (min - height[right]);
            }
            right--;
        }
    }
    return sum;
}
```



## 汉诺塔

**题目：**https://leetcode.cn/problems/hanota-lcci/



**思路1：**

这是一道递归方法的经典题目，乍一想还挺难理清头绪的，我们不妨先从简单的入手。

假设 n = 1,只有一个盘子，很简单，直接把它从 A 中拿出来，移到 C 上；

如果 n = 2 呢？这时候我们就要借助 B 了，因为小盘子必须时刻都在大盘子上面，共需要 4 步。

如果 `n > 2` 呢？思路和上面是一样的，我们把 n 个盘子也看成两个部分，一部分有 1 个盘子，另一部分有 n - 1 个盘子。

你可能会问：“那 n - 1 个盘子是怎么从 A 移到 C 的呢？”

注意，当你在思考这个问题的时候，就将最初的 n 个盘子从 A 移到 C 的问题，转化成了将 n - 1 个盘子从 A 移到 C 的问题， 依次类推，直至转化成 1 个盘子的问题时，问题也就解决了。这就是分治的思想。

而实现分治思想的常用方法就是递归。不难发现，如果原问题可以分解成若干个与原问题结构相同但规模较小的子问题时，往往可以用递归的方法解决。具体解决办法如下：

n = 1 时，直接把盘子从 A 移到 C；
n > 1 时，
先把上面 n - 1 个盘子从 A 移到 B（子问题，递归）；
再将最大的盘子从 A 移到 C；
再将 B 上 n - 1 个盘子从 B 移到 C（子问题，递归）。

采用递归的思路 三要素如下： 

递归结束条件：只剩下最后一个盘子需要移动 

递归函数主功能： 1.首先将 n-1 个盘子，从第一个柱子移动到第二个柱子 2.然后将最后一个盘子移动到第三个柱子上 3.最后将第二个柱子上的 n-1 个盘子，移动到第三个柱子上

 函数的等价关系式： f(n,A,B,C) 表示将n个盘子从A移动到C f(n,A,B,C)=f(n-1,A,C,B)+f(1,A,B,C)+f(n-1,B,A,C)

**复杂度：**

时间复杂度：O(2n−1)O(2^n-1)*O*(2*n*−1)。一共需要移动的次数。

空间复杂度：O(1)O(1)*O*(1)。

**代码：**

```java
class Solution {
    public void hanota(List<Integer> A, List<Integer> B, List<Integer> C) {
        movePlant(A.size(),A,B,C);
    }
    //size 需要移动的盘子的数量
    //start 起始的柱子
    //auxiliary 辅助柱子
    //target 目标柱子
    public void movePlant(int size,List<Integer> start,List<Integer> auxiliary,List<Integer> target){
        //当只剩一个盘子时，直接将它从第一个柱子移动到第三个柱子
        if(size == 1){
            target.add(start.remove(start.size()-1));
            return;
        }
        //首先将 上面的n-1 个盘子，从第一个柱子移动到第二个柱子
        movePlant(size - 1,start,target,auxiliary);
        //然后将最后一个盘子移动到第三个柱子上
        target.add(start.remove(start.size()-1));
        //最后将第二个柱子上的 n-1 个盘子，移动到第三个柱子上
        movePlant(size - 1,auxiliary,start,target);
       
    }
}
```

# 二叉树

## 前序遍历

**题目：** https://leetcode.cn/problems/binary-tree-preorder-traversal/

**思路1：** 递归

**复杂度：**

**代码：**

```java
public static void preOrderRecur(TreeNode head) {
    if (head == null) {
        return;
    }
    System.out.print(head.value + " ");
    preOrderRecur(head.left);
    preOrderRecur(head.rig 。ht);
}
```

## 中序遍历

**题目：** 

**思路1：** 递归

**复杂度：**

**代码：**

```java
public static void preOrderRecur(TreeNode head) {
    if (head == null) {
        return;
    }
    preOrderRecur(head.left);
    System.out.print(head.value + " ");
    preOrderRecur(head.right);
}
```

## 后序遍历

**题目：** 

**思路1：** 递归

**复杂度：**

**代码：**

```java
public static void postOrderRecur(TreeNode head) {
    if (head == null) {
        return;
    }
    postOrderRecur(head.left);
    postOrderRecur(head.right);
    System.out.print(head.value + " ");
}
```



## Linux shell 脚本的第一行是什么？

## 手写shell脚本，将本服务器一个可执行文件复制到集群其他100台服务器上并全部运行起来

## 写一个一致性hash算法

# 设计模式

## 写一个观察者模式（冒泡排序，字符串拼接）

## 写一个单例模式（不要超过10min）

# 并发编程

## 定义三个方法（method1、method2、method3）分别执行时间是（1s、2s、3s），要求并行去查这三个方法拿到他们的结果

线程池

## 顺序打印

```java
public class ABC7 {

    private static CountDownLatch countDownLatchB = new CountDownLatch(1);
    private static CountDownLatch countDownLatchC = new CountDownLatch(1);

    public static void printA() {
        System.out.println("A");
        countDownLatchB.countDown();
    }

    public static void printB() throws InterruptedException {
        countDownLatchB.await();
        System.out.println("B");
        countDownLatchC.countDown();
    }

    public static void printC() throws InterruptedException {
        countDownLatchC.await();
        System.out.println("C");
    }

}
```

