# LinkedHashMap源码解析

```text
    LinkedHashMap是继承HashMap的,大部分的实现还是由HashMap中的代码来实现的.从听说LinkedHashMap的时候,只是知道它是一个有序的map集合,它是怎么做到有序的呢,看看它的源码吧.
```

## LinkedHashMap如何排序的
```java
//默认排序
public static void main(String[] args) {
    LinkedHashMap<String,String> linkedHashMap = new LinkedHashMap<>();
    linkedHashMap.put("a","aa");
    linkedHashMap.put("d","dd");
    linkedHashMap.put("c","cc");
    linkedHashMap.put("b","bb");
    for(Map.Entry<String,String> set :linkedHashMap.entrySet()){
        System.out.println("key: "+set.getKey() +","+ "value: "+set.getValue());
    }
}
```
```java
结果:
key: a,value: aa
key: d,value: dd
key: c,value: cc
key: b,value: bb

是按照put的顺序输出的
```
```java
//accessOrder为true时的排序
public static void main(String[] args) {
    LinkedHashMap<String,String> linkedHashMap = new LinkedHashMap<>(16,0.75f,true);
    linkedHashMap.put("a","aa");
    linkedHashMap.put("d","dd");
    linkedHashMap.put("c","cc");
    linkedHashMap.put("b","bb");
    linkedHashMap.put("a","AA");
    linkedHashMap.get("d");
    for(Map.Entry<String,String> set :linkedHashMap.entrySet()){
        System.out.println("key: "+set.getKey() +","+ "value: "+set.getValue());
    }
}
```
```java
结果:
key: c,value: cc
key: b,value: bb
key: a,value: AA
key: d,value: dd

最新操作过的元素会排在最后面
```

## LinkedHashMap储存结构
![LinkedHashMap储存结构](https://github.com/yyrely/map_code/blob/master/LinkedHashMap%E5%82%A8%E5%AD%98%E7%BB%93%E6%9E%84.png)

## 初始化LinkedHashMap
```java
LinkedHashMap<String,String> map = new LinkedHashMap<>();

//无参构造器
//调用HashMap()无参构造器
//多了一个accessOrder属性
//     false: 按照插入的顺序排序
//     true:  按最新被操作的元素排序
public LinkedHashMap() {
    super();
    accessOrder = false;
}

//同上,多了出初始化容器大小参数设置
public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    accessOrder = false;
}

//同上,多了加载因子参数设置
public LinkedHashMap(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor);
    accessOrder = false;
}

//同上,多了accessOrder参数设置
public LinkedHashMap(int initialCapacity,float loadFactor,boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}

//参数为一个Map
public LinkedHashMap(Map<? extends K, ? extends V> m) {
    super();
    accessOrder = false;
    putMapEntries(m, false);
}


```

## put方法
```java
//调用的是HashMap的put方法
//走的还是putVal方法
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict{
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        //重点,区别
        tab[i] = newNode(hash, key, value, null);
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
            //重点,区别
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    //重点,区别
    afterNodeInsertion(evict);
    return null;
}

//LinkedHash重写了newNode()方法
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    //创建的是LinkedHashMap.Entry<K,V>对象,返回的也是
    LinkedHashMap.Entry<K,V> p = new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    //调用linkNodeLast()方法
    linkNodeLast(p);
    return p;
}

//继承了HashMap的HashMap.Node<K,V>,单向链表的结构
static class Entry<K,V> extends HashMap.Node<K,V> {
    //新加了两个属性,用来记录数据插入顺序或更改顺序的,双向链表结构
    Entry<K,V> before, after;
    //继承HashMap
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}

//将新put的Node加到链的结尾
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    //tail是上一个链尾Node
    LinkedHashMap.Entry<K,V> last = tail;
    //tail被赋值为最新put的Node
    tail = p;
    //若链是空的,头Node为最新put的Node
    if (last == null)
        head = p;
    else {
        //最新Node向前指向上一次最后的Node
        p.before = last;
        //上一次最后的Node向后指向最新的Node
        last.after = p;
    }
}


//这三个方法是HashMap预留的
//LinkedHashMap实现了这三个方法
void afterNodeAccess(Node<K,V> p) { }
void afterNodeInsertion(boolean evict) { }
void afterNodeRemoval(Node<K,V> p) { }

//回调函数,在加入一个新Node后,根据条件移除最老的Node
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    //removeEldestEntry()默认返回的是false
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}
//LinkedHashMap 默认返回false 则不删除节点。 返回true 代表要删除最早的节点。通常构建一个LruCache会在达到Cache的上限是返回true
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}

//accessOrder属性为true,在对Node做(覆盖、查)操作时,会将该Node移到链尾
void afterNodeAccess(Node<K,V> e) { // move node to last
    //定义链尾Node变量
    LinkedHashMap.Entry<K,V> last;
    //accessOrder为true, 链尾Node不是当前Node时,进入
    if (accessOrder && (last = tail) != e) {
        //e  当前被操作的Node
        //p  当前被操作的Node
        //b  当前被操作Node向前指向的Node
        //a  当前被操作Node向后指向的Node
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        //将当前节点移到链尾
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```
## get方法
```java
public V get(Object key) {
    Node<K,V> e;
    //调用HashMap实现的getNode()方法
    if ((e = getNode(hash(key), key)) == null)
        return null;
    //accessOrder: true,执行
    if (accessOrder)
        //上面解释了
        afterNodeAccess(e);
    return e.value;
}
```

##  remove方法
    调用的是HashMap的remove方法,LinkedHashMap重写了afterNodeRemoval()方法
```java
//将当前Node从双向链表中移除
void afterNodeRemoval(Node<K,V> e) { // unlink
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}
```
