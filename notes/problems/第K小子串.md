## 第K小子串

[#链接](https://acm.nowcoder.com/questionTerminal/c59d9690061e448fb8ec7d744c20ebff)：输入一个字符串 s，s 由小写英文字母组成，保证 s 长度小于等于 5000 并且大于等于 1。在 s 的所有不同的子串中，输出字典序第 k 小的字符串。 字符串中任意个连续的字符组成的子序列称为该字符串的子串。
字母序表示英文单词在字典中的先后顺序，即先比较第一个字母，若第一个字母相同，则比较第二个字母的字典序，依次类推，则可比较出该字符串的字典序大小。

数据范围：$0<|s|<5000$ ，$0<k<6$ 

进阶：空间复杂度$O(n)$，时间复杂度$O(n^{2})$

```java
// 输入：一个字符串s，一个整数k
aabb
3
// 输出：一个字符串表示答案
aab

/*
* 不同的子串依次为：
* a aa aab aabb ab abb b bb
* 所以答案为aab
*/
```



## 思路

理解题意后，初步想法是用暴力遍历所有子串，在排序比较找出第k小的字符串。循着这个思路，一个关键问题是：遍历时间得到子串的时间复杂度已为$O(kn)$ ，想要在$O(n^{2})$ 的时间复杂度内找出第k小子串的话，那么比较排序部分时间复杂度应该接近 $O(1)$ 。

为解决上述问题，我们引入大根堆作为有序容器，并将堆的大小控制为k，当堆中元素数量小于k时，直接往堆中插入元素；当堆中元素数量达到了k时，为了保证堆中是字典序最小的k个子串，我们仅在待入队子串的字典序小于堆顶子串时将堆顶子串出队，然后将待入队子串插入。

此外，由于需要保证子串各不相同，还需要一个集合来对子串进行去重。

如此一来，遍历完所有子串后，直接取出堆顶元素即为字典序第k小的元素。

完整代码如下：

```java
public String kthSubstring() {
    Scanner in = new Scanner(System.in);
    String s = in.next();
    int k = in.nextInt();
    HashSet<String> set = new HashSet<>();
    PriorityQueue<String> queue = new PriorityQueue<>((s1, s2) -> s2.compareTo(s1));
    for (int len = 1; len <= k; len++) {   // 子串长度遍历,子串长度小于k
        for (int i = 0; i < s.length() - len + 1; i++) {    // 子符串顺序遍历
            String substr = s.substring(i, i + len);
            if (!set.contains(substr)) {    // 去重操作
                if (queue.size() < k) {
                    queue.offer(substr);
                } else {
                    if (substr.compareTo(queue.peek()) < 0) {   // 子串比队首串小
                        queue.poll();
                        queue.offer(substr);
                    }
                }
                set.add(substr);
            }

        }
    }

    return queue.peek();
}
```



