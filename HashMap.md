# HashMap源码解析
## 初始化HashMap
```java
//新建一个Hashmap
HashMap<String,String> map = new HashMap<>();

//无参构造器
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR;// 默认加载因子0.75f
}

//一个参数的构造器,指定容器大小,实际调用的是两个参数的构造器
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

//两个参数的构造器,指定容器大小,设定加载因子(一般使用默认)
public HashMap(int initialCapacity, float loadFactor) {
    //小于0,抛出异常
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " initialCapacity);
    
    //大于2^30,大小为2^30
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;

    //加载因子小于等于0,或者为null,抛出异常
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " loadFactor);
    
    this.loadFactor = loadFactor;
    //容器临界大小
    this.threshold = tableSizeFor(initialCapacity);
}

//很牛的一个算法

//来假设计算一下,cap = 9
//n = 9 - 1 = 8
//n :          00000000000000000000000000001000
//n >>> 1 :    00000000000000000000000000000100
//或运算结果 :  00000000000000000000000000001100
//n >>> 2 :    00000000000000000000000000000011
//或运算结果 :  00000000000000000000000000001111
//n >>> 4 :    00000000000000000000000000000000
//或运算结果 :  00000000000000000000000000001111
//n >>> 8 和 n >>> 16 最后结果都是一样的
//n : 15
//最后返回的是16
//找大于等于cap且最小的2的幂
//cap - 1 操作是防止已经是2的幂,计算出来的结果会是原来的2倍(可带16,跳过cap - 1直接试)
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```
## put第一个值
```java
map.put("a","aa");

//调用put方法,key: "a",value: "aa"
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

//将key传入hash方法中进行计算
static final int hash(Object key) {
    int h;
    //h = 97 
    //h >>> 16 = 0
    //异或运算:  1100001
    //          0000000
    //          1100001
    //结果还是 97
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

//调用putVal方法,hash: 97,key: "a",value: "aa", onlyIfAbsent: "false",evict: true
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict) {
    //定义变量,tab: 节点数组,p: 节点
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //table初始为null
    if ((tab = table) == null || (n = tab.length) == 0)
        //tab = resize() = Node<K,V>[16],N = 16
        n = (tab = resize()).length;
    //(n - 1) & hash,计算0-15的数组下标,i=1
    if ((p = tab[i = (n - 1) & hash]) == null)
        //tab[1]为一个新的节点
        tab[i] = newNode(hash, key, value, null);
    //不走
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    //++size = 1,threshold = 12
    if (++size > threshold)
        resize();
    //未实现
    afterNodeInsertion(evict);
    return null;
}

final Node<K,V>[] resize() {
    //table = null, oldTab = null
    Node<K,V>[] oldTab = table;
    //oldCap = 0
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    //oldCap = 0
    int oldThr = threshold;
    int newCap, newThr = 0;
    //不走
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; 
    }
    //不走
    else if (oldThr > 0) 
        newCap = oldThr;
    else {    
        //newCap = 16           
        newCap = DEFAULT_INITIAL_CAPACITY;
        //newThr = 16 * 0.75 = 12
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    //不走
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }
    //容器临界大小值为12
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    //新建一个大小为16的节点数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    //容器为16大小的节点数组
    table = newTab;
    //不走
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
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
    //返回新建的数组
    return newTab;
}

//一个节点类,当前节点会指向下一个节点
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

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```
> map.put("a","aa");这个语句的过程  
初始化了容器,容器大小为16,临界值为12  
并通过计算在数组[1]中加入了一个新的Node,hash = 97,key = a,value = aa ,next = null

## put第二个值
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //table不为空,不走
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;


    //主要差别在这里
    //table是一个长度为16的数组
    //通过计算得出数组的下标
    //判断该[下标]中是否有值
    //无: 创建一个新的Node加到该位置
    //有: 继续做判断
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        //取出来的Node为p
        //判断p的key是否于新的key相等
        //相等:   覆盖原来位置上的Node
        //不相等: 继续做判断
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //判断节点是否为TreeNode,红黑树
        //true:  加上一个枝节点
        //false: 继续做判断
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //若不为TreeNode,这就是一个单项链
            //循环这个单项链
            for (int binCount = 0; ; ++binCount) {
                //如果是单项链的结尾
                if ((e = p.next) == null) {
                    //在结尾Node上指向该Node,就是加一个Node在链的结尾
                    p.next = newNode(hash, key, value, null);
                    //如果链的长度大于等于8 - 1
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        //将链转为TreeNode,可更快的遍历
                        treeifyBin(tab, hash);
                    break;
                }
                //若新的Key和循环过程中链上的一个Key相等,覆盖这个Node
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //覆盖原Node后做的操作
        //覆盖了原先的Node,设置Node新的Value值,将原来Node中的Value返回
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    //加入新Node做的操作
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
### 上面Node的存储结构
>![图片](https://github.com/yyrely/map_code/blob/master/HashMap%E7%BB%93%E6%9E%84.png)

## 存储的Node数量大于临界值(扩容)
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict) {
    
    ....
    //这个添加为真,再调用resize方法
    //我们假设++size = 13
    //13 > 12 第一次重设容器大小
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}

final Node<K,V>[] resize() {
    //与初始化的时候不同,table不为null
    Node<K,V>[] oldTab = table;
    //oldCap = 16
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    //oldThr = 12
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        //不满足不走
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //newCap = oldCap << 1 = 32
        //32 < 2^30 true
        //32 > 12   ture
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
            //newThr = oldThr << 1 = 24
            newThr = oldThr << 1; // double threshold
    }
    //不走
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    //不走
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }

    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }
    //threshold = 24
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    //newTab = new Node[32] 长度为32的数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    //table = new Node[32] 长度为32的数组
    table = newTab;
    //true
    if (oldTab != null) {
        //遍历老的Node数组
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            //若取出来的Node为null,下一次循环
            if ((e = oldTab[j]) != null) {
                //将数组下标为j的内容置为null
                oldTab[j] = null;
                //链中只有一个Node,没有下一个Node
                if (e.next == null)
                    //将这个老数组中取出来的值,存在新数组中
                    //通过key的hash值计算,取出[0-31]值中的一个
                    newTab[e.hash & (newCap - 1)] = e;
                //如果是TreeNode的在这里做操作
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                //是链的Node在这里做操作
                else {
                    //定义变量
                    //以原来的下标存储在数组,链头,和链中当前的Node
                    Node<K,V> loHead = null, loTail = null;
                    //以原下标 + oldCap为下标存储在数组
                    Node<K,V> hiHead = null, hiTail = null;
                    //链中下一个Node
                    Node<K,V> next;
                    do {
                        //取出当前Node指向的下一个Node
                        next = e.next;
                        //做下标运算
                        // e.hash & oldCap
                        // 若hash为15
                        //      0000 1111
                        // &    0001 0000
                        // 结果 0
                        // 以原来的下标为新下标存储在数组中
                        // 若hash为20
                        //      0001 0100
                        // &    0001 0000
                        // 结果 1
                        // 以原下标 + oldCap为新下标存储在数组中
                        if ((e.hash & oldCap) == 0) {
                            //第一次进循环,设置头节点
                            if (loTail == null)
                                loHead = e;
                            else
                                //上一个链节点指向当前节点
                                loTail.next = e;
                            //设置当前节点
                            loTail = e;
                        }
                        //与上面的操作一致
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    //遍历当前链表
                    } while ((e = next) != null);
                    if (loTail != null) {
                        //将链表的尾结点设为空
                        loTail.next = null;
                        //以原来的下标为新下标存储在数组中
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                         //将链表的尾结点设为空
                        hiTail.next = null;
                        //以原下标 + oldCap为新下标存储在数组中
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```
### HashMap扩容图
![HashMap扩容图](\hashMap扩容例图.png)

## 从HashMap中Get值
```java
String value = map.get("a");


public V get(Object key) {
    Node<K,V> e;
    //将key的hash值,和key传入getNode()方法中
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    //定义变量
    //tab   map中数组
    //first 数组中存放的链表头(树根)
    //n     map数组长度
    //K     节点中key值
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    //数组不为空,数组长度大于0
    //根据传进来key的hash值计算数组下标,取出该下标中的链表(Tree)
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        //判断第一个节点的hash值和key和传进来的hash值,key值都相等
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        //取链中的下一个Node
        if ((e = first.next) != null) {
            //结构为TreeNode的操作
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            //遍历链表,判断每一个Node的hash值和key值是否和传进来的相等
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

[参考文章一](https://blog.csdn.net/u013494765/article/details/77837338)  
[参考文章二](https://www.cnblogs.com/zhaojj/p/7805376.html)



