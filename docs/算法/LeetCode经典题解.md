### 001两数之和

给定一个整数数组 nums 和一个目标值 target，请你在该数组中找出和为目标值的那 两个 整数，并返回他们的数组下标。

你可以假设每种输入只会对应一个答案。但是，数组中同一个元素不能使用两遍。

示例:

```
给定 nums = [2, 7, 11, 15], target = 9

因为 nums[0] + nums[1] = 2 + 7 = 9
所以返回 [0, 1]
```

**解法：哈希表**

```
class Solution {
    public int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> record = new HashMap();
        for (int i = 0; i < nums.length; i++) {
            int temp = target - nums[i];
            //算出差值  查看map里是否有  有的话返回[之前的索引，现在索引]  否则放入map中
            if (record.containsKey(temp)) {
                return new int[]{record.get(temp),i};
            }
            record.put(nums[i],i);
        }
        return new int[]{-1,-1};
    }
}
```

### 146 LRU缓存机制

运用你所掌握的数据结构，设计和实现一个  LRU (最近最少使用) 缓存机制。它应该支持以下操作： 获取数据 get 和 写入数据 put 。

获取数据 get(key) - 如果关键字 (key) 存在于缓存中，则获取关键字的值（总是正数），否则返回 -1。
写入数据 put(key, value) - 如果关键字已经存在，则变更其数据值；如果关键字不存在，则插入该组「关键字/值」。当缓存容量达到上限时，它应该在写入新数据之前删除最久未使用的数据值，从而为新的数据值留出空间。

**进阶:**

你是否可以在 O(1) 时间复杂度内完成这两种操作？

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

**解题思路**

有出入顺序首先想到栈、队列和链表

使用栈    元素添加到栈，栈底被淘汰，如果栈中有被引用，从栈中删除，并添加到栈顶

缺点会产生大量的内存拷贝。

链表可以快速移动节点位置，无法随机访问。

**LinkedHashMap**

HashMap 底层是 数组 + 红黑树 + 链表   同时其是无序的，而 LinkedHashMap 刚好就比 HashMap 多这一个功能，就是其提供有序。

LinkedHashMap的有序可以按两种顺序排列，一种是按照插入的顺序，一种是按照**读取**的顺序

而其内部是靠 建立一个双向链表 来维护这个顺序的，在每次插入、删除后，都会调用一个函数来进行 双向链表的维护 ，准确的来说，是有三个函数来做这件事，这三个函数都统称为 回调函数 ，这三个函数分别是：

- `void afterNodeAccess(Node<K,V> p) { }`
  其作用就是在访问元素之后，将该元素放到双向链表的尾巴处(所以这个函数只有在按照读取的顺序的时候才会执行)，之所以提这个，是建议大家去看看，如何优美的实现在双向链表中将指定元素放入链尾！
- `void afterNodeRemoval(Node<K,V> p) { }`
  其作用就是在删除元素之后，将元素从双向链表中删除，还是非常建议大家去看看这个函数的，很优美的方式在双向链表中删除节点！
- `void afterNodeInsertion(boolean evict) { }`
  这个才是我们题目中会用到的，在插入新元素之后，需要回调函数判断是否需要移除一直不用的某些元素！