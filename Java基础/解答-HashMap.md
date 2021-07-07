### HashMap 源码

```java
public class HashMap<K, V> extends AbstractMap<K, V> implements Map<K, V>, Cloneable, Serializable {
    private static final long serialVersionUID = 362498820763181265L;
    static final int DEFAULT_INITIAL_CAPACITY = 16;
    static final int MAXIMUM_CAPACITY = 1073741824;
    static final float DEFAULT_LOAD_FACTOR = 0.75F;
    static final int TREEIFY_THRESHOLD = 8;
    static final int UNTREEIFY_THRESHOLD = 6;
    static final int MIN_TREEIFY_CAPACITY = 64;
    transient HashMap.Node<K, V>[] table;
    transient Set<Entry<K, V>> entrySet;
    transient int size;
    transient int modCount;
    int threshold;
    final float loadFactor;
}
```

- table 实际是一个实现了 Entry 接口的 Node 类型的数组，即哈希桶数组。
- entrySer 作为一个entrySet缓存，使用entrySet方法首先检查其是否为null，不为null则使用这个缓存，否则生成一个并缓存至此。
- size 是实际键值对个数
- loadFactor 表示扩展因子。
- threshold 表示阈值。threshold = table.length * loadFactor



### HashMap 底层数据结构
HashMap使用哈希表来存储的。哈希表为解决冲突，可以采用开放地址法和链地址法等来解决问题。HashMap采用了链地址法，简单的说，就是数组加链表的数组。每个数组元素指向一个单向链表。当链表过长会转换成红黑树以实现 O(logN)时间复杂度查找。

#### 哈希冲突
```java
map.put('1', 'name');
```

哈希表添加元素的时候，会获取 key 的 hashCode 值，然后再通过 Hash 算法的后两步运算来定位键值对的存储位置，有时两个 key 会定位到相同的位置，这就是哈希冲突。如果 Hash 算法计算结果越分散均匀，哈希冲突的概率就越小，map 存取的效率就会越高。



#### 如果链表不转换成红黑树，有什么影响

目前 HashMap 发生哈希冲突，会把冲突的值放入 key 所指向的单向链表。如果 Hash 算法不够优秀，导致链表的长度很长。那么查找一个键值对，就要遍历链表，我们知道链表的遍历的时间复杂度是 O(N)。这样就会**严重影响 HashMap 的性能。**

所以在 JDK1.8 版本中，引入了红黑树，当链表长度太长（默认超过8）时，链表就转换为红黑树，**利用红黑树快速增删改查的特点提高HashMap的性能**，其中会用到红黑树的插入、删除、查找等算法。



### 扩容机制
```java
    final HashMap.Node<K, V>[] resize() {
        HashMap.Node<K, V>[] oldTab = this.table;
        int oldCap = oldTab == null ? 0 : oldTab.length;
        int oldThr = this.threshold;
        int newThr = 0;
        int newCap;
        if (oldCap > 0) {
            if (oldCap >= 1073741824) {
                this.threshold = 2147483647;
                return oldTab;
            }

            if ((newCap = oldCap << 1) < 1073741824 && oldCap >= 16) {
                newThr = oldThr << 1;
            }
        } else if (oldThr > 0) {
            newCap = oldThr;
        } else {
            newCap = 16;
            newThr = 12;
        }

        if (newThr == 0) {
            float ft = (float)newCap * this.loadFactor;
            newThr = newCap < 1073741824 && ft < 1.07374182E9F ? (int)ft : 2147483647;
        }

        this.threshold = newThr;
        HashMap.Node<K, V>[] newTab = new HashMap.Node[newCap];
        this.table = newTab;
        if (oldTab != null) {
            for(int j = 0; j < oldCap; ++j) {
                HashMap.Node e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null) {
                        newTab[e.hash & newCap - 1] = e;
                    } else if (e instanceof HashMap.TreeNode) {
                        ((HashMap.TreeNode)e).split(this, newTab, j, oldCap);
                    } else {
                        HashMap.Node<K, V> loHead = null;
                        HashMap.Node<K, V> loTail = null;
                        HashMap.Node<K, V> hiHead = null;
                        HashMap.Node hiTail = null;

                        HashMap.Node next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null) {
                                    loHead = e;
                                } else {
                                    loTail.next = e;
                                }

                                loTail = e;
                            } else {
                                if (hiTail == null) {
                                    hiHead = e;
                                } else {
                                    hiTail.next = e;
                                }

                                hiTail = e;
                            }

                            e = next;
                        } while(next != null);

                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }

                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }

        return newTab;
    }
```

时机：
- 一般情况下当实际数量 size 数量大于等于 阈值 threshold ，此刻触发扩容机制，将其容量扩大为 2 倍。HashMap的容量是有上限的，必须小于 **1<<30**，即1073741824。如果容量超出了这个数，则不再增长，且阈值会被设置为**Integer.MAX_VALUE**，即永远不会超出阈值)。
- 首次 put 时，先会触发扩容（算是初始化容量，默认 size 16 threshold 12），然后存入数据，然后判断是否需要扩容

步骤：

1. 计算新的容量，并初始化一个新的 Node 数组
2. 将原有的 Node  数组的元素拷贝到新的 Node 数组。
   - 在拷贝数据过程中 jdk1.8 不会重新计算hash ，jdk 1.7会重新计算hash 。（至于为什么，我还没搞懂）

