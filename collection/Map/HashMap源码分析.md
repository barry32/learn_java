### HashMap源码分析

学过数据结构中散列表的同学，对于Hash应该不会陌生。本章我们一起来看HashMap，还是从先从族谱图看起。

<img src="./HashMap.png" alt="HashMap" style="zoom:75%;" />

看过族谱图之后，我们可以知道HashMap除了满足可序列化、克隆的性质外，还继承了AbstractMap。AbtractMap相对于HashMap正如AbstractList相对于ArrayList一样，是对于Map的基本骨架的实现类，用来减少HashMap一部分工作的。

>Hash table based implementation of the <tt>Map</tt> interface.  This
>implementation provides all of the optional map operations, and permits
><tt>null</tt> values and the <tt>null</tt> key.  (The <tt>HashMap</tt>
>class is roughly equivalent to <tt>Hashtable</tt>, except that it is
>unsynchronized and permits nulls.)  This class makes no guarantees as to
>the order of the map; in particular, it does not guarantee that the order
>will remain constant over time.

上文是摘自JDK1.8版本中对于HashMap的解释，HashMap是基于Map接口实现的哈希表，HashMap提供所有的可选的Map操作，并允许其中的元素存在key和value为null的情况。HashMap大致与Hashtable相同，除了HashMap不是线程安全的以及HashMap允许元素为null 的情况。HashMap不保证Map中的顺序。特别是，它不保证随着时间的推移，Map中元素的顺序仍然不变。

***HashMap构造函数***

```java
//装填因子满足于泊松分布，在7，8之间碰撞最少。
static final float DEFAULT_LOAD_FACTOR = 0.75f;
//逻辑左移4位，默认大小为16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
static final int MAXIMUM_CAPACITY = 1 << 30;
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}
  /**
    *tableSizeFor 方法就是将用户自定的初始化容量转变最接近的2的幂次的数字
    *那么为什么要让HashMap的容量为2的幂次呢？
    *在数据结构这门课程的散列表学习中，当我们向哈希表中插入的数据的时候。假设我们有一个length为7的表，
    *需要插入数值16到哈希表，我们用16取7的模为2，那么我们将16这个数存在放在哈希表中的index为2的位置
    *（index默认从0开始）
    * 然而在HashMap中的是采用的hash&(length-1)这样的位与元素来代替我们大学中学的取模运算的方式，
    *来获取所要存储元素的下标位置，而这一计算的前提都是依赖于HashMap的容量大小为2的幂次。
    *那么问题转化为，
    *假设HashMap的容量为2的幂次，证明hash&(length-1) = hash%length，其中2^n=length
    *解:
    *首先length是2的幂次，所以换算为二进制为1000...0,1 后面n个0，那么length-1必然是0111...1, 0后面n个1
    *那么hash&(length-1)的结果必然是hash二进制中的后n位数字。
    *hash/lenth等价于 hash/2^n等于二进制中hash右移n位，即移除二进制中hash的后n位数字。
    *（左移n位的话，右边需补n个0），被移除的hash中的后n位即为hash&（length-1），hash/lenth剩下的是商，
    *舍弃的是余数，而hash/lenth的余数等于hash%length。
    *所以hash&(length-1) = hash%length,当HashMap的容量为2的幂次的情况下成立。
    */
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

***HashMap元素添加***

```java
transient Node<K,V>[] table;
static final int UNTREEIFY_THRESHOLD = 6;
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
/**
  *首先计算hash值，hashCode是32位的,使用高16位来异或低16位
  *那么HashMap采用这样的方式来计算Hash？
  *HashMap为元素寻址的时候使用(n-1)&hash, 使用高16位来异或低16位，这样所得hash值中低16位中包含了	   *HashCode高16位的信息，可以有效减少碰撞
  */
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
//onlyIfAbsent 入参: 如果是true，不会覆盖值

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //底层是一个Node结点类型的数组，先判断table数组 是否是刚创建的，满足条件直接扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //这边运用(length-1)&hash == hash%length的公式,当length为2的幂次的情况下
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        //说明当前p通过(length-1)&hash计算得到的Node数组下标存在数值。
        //hash值相同并且key相同,该元素的value等待被覆盖
        //HashMap中允许key为null的情况
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //当前位置所在节点为红黑树，将新节点插入红黑树
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //当前位置有值,但是key不同，也不是树型结构
            //当前节点属于链表结构或者即将属于链表结构
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    //冲突个数达到8，开始树化
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //遍历当前节点(链表)，可能会遇到相同的key，等待被覆盖。
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //e不为空的话，说明e指向即将被覆盖的节点
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            //替换旧值
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    //新添加的值并没有覆盖别的节点
    ++modCount;
    //数量达到阈值,需要rehash
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
//扩容机制
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    //旧表的容量大于0，说明HashMap中原来就有元素，但是已添加的元素个数已经达到了阈值。所以需要扩容
    if (oldCap > 0) {
        //HashMap中容量过大的处理方式
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //此时旧表的容量不是很大，直接将新表的容量和thresold阈值翻倍即可
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    /**
      *这边的逻辑是旧表没有元素，但是阈值却大于0,这种情况发生在HashMap初始化的时候，
      *当用户传入指定的初始化容量时，通过tablesizefor方法将此值转化为距离该值最小的2的幂次数
      *并将此值复制给Threshold。此时，旧表中没有数据并且Threshold大于0，所以直接将thresold
      *赋值给新表
      */
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    /**
      *此时HashMap中既没有元素，用户也没有自定义HashMap的大小，说明用户调用的是无参构造方法
      *(比如: Map<String,String> map =new HashMap<>();)
      *而无参构造方法只是赋值了默认的装填因子0.75，所以capacity取默认的16，threshold的计算公式是
      * 阈值=装填因子*容量
      */
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    /**
      *杜绝newThr仍然为0的情况，比如 当前HashMap处于用户调用带参构造方法的初始化状态。
      */
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    //建立好新表节点数组，capacity和thresold已经设置好，剩下的就是需要同步数据了。
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    /**
      *开始顺序遍历旧表
      *同步数据的过程当中大致会遇到四种情况
      *1: 该节点为空，此时是不需要去迁移的。
      *2: 该节点没有后继节点，说明该节点是单节点，又因为hash&(length-1) = hash%length(证明见上)
      *所以直接将该节点存在 newTab[e.hash&newCap-1]的位置,此时e已经指向当前节点。
      *3: 如果当前节点是一棵红黑树，HashMap定义了内部类TreeNode,包含prev和next属性
      *这边处理方式跟链表类似使用 hash&oldCap 来生成高位链和低位链，如果生成的高低位链的冲突个数小于6
      *需要将其退化为链表。                                                                
      *4: 最后一种情况是，当前节点是一个链表, HashMap这边选择rehash的方式来处理。
      *(方案是: 判断e.hash&oldCap是否为0)
      *那么HashMap为什么选择这种方法来rehash呢？
      *解:
      *HashMap的容量始终为2的幂次，换算为二进制  10...0 (1后面N个0)，容量为2^n
      *hash位与运算oldCap,只有两边都为1该位数才为1,oldCap后面N个0,前面一个1。所以位与运算的结果是
      *hash的第N+1位，加上后面的N个0。因为是二进制，hash的第N+1位要么是1,要么是0。
      *所以hash&oldCap的结果要么是0,要么是oldCap,这样就将这个节点下的链表中的元素分为两半。
      *
      *在对节点为链表的情况处理后，HashMap采用尾插法去重新处理链表下的这些元素。
      */
    
    if (oldTab != null) {
        //开始顺序遍历旧表
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            //排除第一种情况:节点为空
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                //第二种情况，该节点为普通元素(既不是红黑树,也不是链表)
                if (e.next == null)
                    //运用e.hash&(newCap-1) == e.hash%newCap公式,当newCap是2的幂次的时候
                    newTab[e.hash & (newCap - 1)] = e;
                //第三种情况，该节点是一颗红黑树
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                //第四种情况，该节点为链表结构
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        //指向当前节点的后继节点
                        next = e.next;
                        //hash&oldCap 结果要么是0，要么是oldCap 
                        //(结果为0我们暂时称为低位节点元素，结果为oldCap称为高位节点元素)
                        if ((e.hash & oldCap) == 0) {
                            //loHead指向低位的头节点,loTail指向低位的尾节点
                            if (loTail == null)
                                loHead = e;
                            else
                             //存在新的后继节点，先关联当前的与后继节点
                                loTail.next = e;
                            //lotail继续指向低位的尾节点
                            loTail = e;
                        }
                        else {
                            //hiHead指向高位的头节点，hiTail指向高位的尾节点
                            if (hiTail == null)
                                hiHead = e;
                            else
                                //出现新的后继节点，先将当前节点关联后继节点
                                hiTail.next = e;
                            //hiTail继续指向尾节点
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    //当loTail为0,说明不存在低位节点元素
                    if (loTail != null) {
                        loTail.next = null;
                        //将低位节点的元素生成的链表，放在新表的j的位置，也就是旧表的原来的位置
                        newTab[j] = loHead;
                    }
                    //同理当hiTail为0,说明不存在高位节点元素
                    if (hiTail != null) {
                        hiTail.next = null;
                        //将高位节点的元素生成的新的链表，放在新表的j+oldCap的位置，
                        //当前的新的位置是旧表原来的位置向右偏移旧表的容量长度
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
//调用split方法拆分红黑树
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
            TreeNode<K,V> b = this;
            // Relink into lo and hi lists, preserving order
            //类似于高位链表、低位链表的处理方式
            //loHead指向低位链的头节点，loTail指向低位链的尾节点
            TreeNode<K,V> loHead = null, loTail = null;
            TreeNode<K,V> hiHead = null, hiTail = null;
            //高低位链的计数
            int lc = 0, hc = 0;
            //HashMap定义内部类TreeNode,带有prev,next属性，使用hash&cap来拆成高低位链
            for (TreeNode<K,V> e = b, next; e != null; e = next) {
                next = (TreeNode<K,V>)e.next;
                e.next = null;
                if ((e.hash & bit) == 0) {
                    if ((e.prev = loTail) == null)
                        loHead = e;
                    else
                        loTail.next = e;
                    loTail = e;
                    ++lc;
                }
                else {
                    if ((e.prev = hiTail) == null)
                        hiHead = e;
                    else
                        hiTail.next = e;
                    hiTail = e;
                    ++hc;
                }
            }

            if (loHead !q= null) {
                //区别于链表的处理方式，冲突个数小6的红黑树退化为链表
                //低位链依旧存放在表格原index处
                if (lc <= UNTREEIFY_THRESHOLD)
                    tab[index] = loHead.untreeify(map);
                else {
                    tab[index] = loHead;
                    if (hiHead != null) // (else is already treeified)
                        loHead.treeify(tab);
                }
            }
            if (hiHead != null) {
                //区别于链表的处理方式，冲突个数小6的红黑树退化为链表
                //高位链存放在表格原index+oldCap处
                if (hc <= UNTREEIFY_THRESHOLD)
                    tab[index + bit] = hiHead.untreeify(map);
                else {
                    tab[index + bit] = hiHead;
                    if (loHead != null)
                        hiHead.treeify(tab);
                }
            }
        }




```

***HashMap元素删除***

```java
public V remove(Object key) {
    Node<K,V> e;
    //用key的hashCode来计算hash值 见上文
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

//matchValue为true时，只有value跟值相匹配才删除
//movable为false,当删除时不移动剩下的节点
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    //忽略底层为Node类型的数组中没有数据的情况以及数组中找不到值的情况
    //(n-1)&hash == hash%n 这个公式证明解答见上文
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        //找到hash值相同并且key相同的值，并且该值存放在tab数组中，做好删除准备
        //[hash值的计算公式 : (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16) ]
        //(判断key是否相同,需要判断hashCode 和 equals，只是单纯判断equals 效率降低)
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        //但是当前指向的节点key不符合期望
        //满足当前要求有两种可能，当前节点要么是树型结构、要么是链表结构
        else if ((e = p.next) != null) {
            //当前节点时树形结构，需要遍历红黑树(细节到红黑树那篇再展开)
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                //遍历链表
                do {
                    //找到hash值相同的 && key相同的数值
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        //当前node节点指向被删除节点
        //当matchValue 为true，只有value跟元素相同时才能删除
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            //被删除节点为红黑树
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            //当前节点为期望中被删除的节点(确切讲，table数组中的当前指向的节点时需要被删除的节点)
            else if (node == p)
                tab[index] = node.next;
            //这边p节点已经是node的前驱节点
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            //返回被删除的节点
            return node;
        }
    }
    return null;
}
```

关于HashMap中的一点总结，HashMap在集合中占据比较重要的席位。HashMap初始容量是16，装填因子0.75

HashMap中的容量始终是2的幂次，当进行put操作的时候，如果是新创建的集合会调用resize方法，并使用key计算hash，依照情况插入数据这边需要注意的是，当容量达到64并且冲突个数达到8，HashMap才会树化。在resize扩容中，当冲突个数只有6的时候，树会退化为链表。进行remove操作的时候，还是会计算hash, 当hash与key都正确匹配的时候，考虑删除。