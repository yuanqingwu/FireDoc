https://www.cnblogs.com/xiaoxi/p/6170590.html


LinkedHashMap的实现就是HashMap+LinkedList的实现方式，以HashMap维护数据结构，以LinkList的方式维护数据插入顺序。

LinkedHashMap有自己的静态内部类Entry，它继承了HashMap.Node


```
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
        ......
    }
```


```
    static class LinkedHashMapEntry<K,V> extends HashMap.Node<K,V> {
        LinkedHashMapEntry<K,V> before, after;
        LinkedHashMapEntry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```


### 利用LinkedHashMap实现LRU算法缓存
LinkedHashMap可以实现LRU算法的缓存基于两点：

1、LinkedList首先它是一个Map，Map是基于K-V的，和缓存一致

2、LinkedList提供了一个boolean值可以让用户指定是否实现LRU

LRU即Least Recently Used，最近最少使用，也就是说，当缓存满了，会优先淘汰那些最近最不常访问的数据。

