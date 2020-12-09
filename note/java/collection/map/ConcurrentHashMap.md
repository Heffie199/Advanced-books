## ConcurrentHashMap 源码分析

在上文我们对 `HashMap`进行了分析，而`HashMap`是线程不安全的，
本篇主要想讨论 `ConcurrentHashMap` 这样一个并发容器，为什么说它是并发的容器呢？
我们会在下文获得答案

### 构造方法

- ConcurrentHashMap() 

    创建一个新的，空的Map与默认的初始表大小（16）。  
- ConcurrentHashMap(int initialCapacity) 

    创建一个新的空的Map，其初始表格大小适应指定数量的元素，而不需要动态调整大小。  
- ConcurrentHashMap(int initialCapacity, float loadFactor) 

    根据给定的元素数量（ initialCapacity ）和初始表密度（ loadFactor ），创建一个新的，空的Map，初始的表格大小。  
- ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) 
    
    创建具有基于给定数量的元件（初始表大小的新的空映射 initialCapacity ），表密度（ loadFactor ），和同时更新线程（数 concurrencyLevel ）。  
- ConcurrentHashMap(Map<? extends K,? extends V> m) 
    
    创建与给定Map相同的映射的新Map。 
    
### 重要的变量

- private transient volatile int sizeCtl;
    
    正在初始化表或调整表大小：-1表示初始化，否则-（1+活动的调整大小线程数）。 否则，当table为null时，保留创建时要使用的初始表大小，或0表示默认值。初始化后，保存下一个要调整表大小的元素计数值。

  
    
### put(k,v)  

```java
 final V putVal(K key, V value, boolean onlyIfAbsent) {
        // 可以看出 ConcurrentHashMap 是不允许又 null 的存在的
        if (key == null || value == null) throw new NullPointerException();
        // 计算hash值
        int hash = spread(key.hashCode());
        // 转 tree 的阈值
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh; K fk; V fv;
            if (tab == null || (n = tab.length) == 0)
                // 判断我们的map是否被初始化，如果没有需要我们要进行初始化
                // initTable 这里需要注意的是，他会根据 sizeCtl 来调整策略
                //  正在初始化表或调整表大小：-1表示初始化，否则-（1+活动的调整大小线程数）。
                //  否则，当table为null时，保留创建时要使用的初始表大小，或0表示默认值。
                //  初始化后，保存下一个要调整表大小的元素计数值。
                tab = initTable();
            // 可变访问方法用于调整大小时的表元素以及正在进行的下一个表的元素。调用方必须对制表符参数的所有使用进行null检查。
            // 所有的调用者也会偏执地预先检查tab的长度不是零（或者等效检查），从而确保任何以散列值和（length-1）为形式的索引参数都是有效的索引。
            // 注意，为了纠正用户的任意并发错误，这些检查必须对局部变量进行操作，
            // 这解释了下面一些奇怪的内联作业。注意，对setTabAt的调用总是发生在锁定区域内，
            //因此，原则上只需要发布顺序，而不是完整的volatile语义，但目前被编码为volatile writes以保持保守。
            // tabAt() casTabAt() 采用的是 Unsafe 直接操作虚拟机 ，
            // 如果对 Unsafe 不了解 可以在阅读 https://github.com/yanghaiji/Advanced-books/blob/master/note/java/concurrency/README.md
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))//通过cas插入新值
                    break;                   // no lock when adding to empty bin
            }
            // 如果 条件成立，说明 正在调整大小(扩容)，当前进行帮助进行，通过 Unsafe 进行复制，将原始的数据复制到一个新的空间
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else if (onlyIfAbsent // check first node without acquiring lock
                     && fh == hash
                     && ((fk = f.key) == key || (fk != null && key.equals(fk)))
                     && (fv = f.val) != null)
                return fv;
            else {
                V oldVal = null;
                synchronized (f) {
                    //如果tab[i]不为0，且容量足够，那么这时需要对当前hash值对应的Node加锁，防止其他线程改变该Node，然后找到和key相同
                    //的键值对，把value替换旧value，返回旧value
                    //为什么再次检查？因为不能保证，当前线程运行到这里，有没有其他线程对该节点进行修改
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key, value);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            // 如果HashEntry下的链表已经变为红黑树，则找到key对应的树节点，然后替换value，返回旧value
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
                if (binCount != 0) {
                    // 如果链表长度大于阈值，那么链表转换为红黑树
                    if (binCount >= TREEIFY_THRESHOLD)
                        // 这里也采用的是 Unsafe 
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        // 计数增加1，有可能触发transfer操作(扩容)
        addCount(1L, binCount);
        return null;
    }
``` 

```java
    //transient volatile Node<K,V>[] table; tab变量确实是volatile
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {//获取table中索引 i 处的元素。
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);//如果tab是volatile变量，则该方法保证其可见性。
    }
    //通过CAS设置table索引为 i 处的元素。
    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }
    //transient volatile Node<K,V>[] table; tab变量确实是volatile
    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {//修改table 索引 i 处的元素。
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);//如果tab是volatile变量，则该方法保证其可见性。
    }
```

### initTable()

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // 赋值sc。并当sizeCtl == -1 即当前有线程正在执行初始化
        if ((sc = sizeCtl) < 0)
            //yield()暂停当前正在执行的线程，执行其他线程
            //（这是一个通知，但是这是不一定会让当前线程停止，要取决于线程调度器）
            //就是我想让出资源，但是这只是一厢情愿的事情，线程调度器会考虑你的方法，但是不一定采纳。
            Thread.yield();
        //修改 sizeCtl 的值为 -1。 SIZECTL 为 sizeCtl 的内存地址。
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                //执行初始化过程
                if ((tab = table) == null || tab.length == 0) {
                    //sc在上面已经赋值，=原来 sizeCtl的值。是非讨厌JDK源码这种赋值方式。
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    //创建一个sc长度的table。
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                //初始化完成, sizeCtl重新赋值为当前数组的长度。
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

