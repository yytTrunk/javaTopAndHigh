### 一 List、Set、Map

### 1.List、Set、Map接口

- Map 接口和 Collection 接口是所有集合框架的父接口；
- Collection 接口的子接口包括：Set 接口、List 接口和Queue接口；
- Map 接口的实现类主要有：HashMap、TreeMap、LinkedHashMap、Hashtable、ConcurrentHashMap 以及 Properties 等；
- Set 接口的实现类主要有：HashSet、TreeSet、LinkedHashSet 等；
- List 接口的实现类主要有：ArrayList、LinkedList、Stack 、Vector以及CopyOnWriteArrayList 等；
- Queue接口的主要实现类有：ArrayDeque、ArrayBlockingQueue、LinkedBlockingQueue、PriorityQueue等；

![](https://gitee.com/codeyyt/my_pic/raw/master/image-blog/javath/01/01_01.png)

### 二 LIst

#### 1. ArrayList与LinkedList

##### 1.1 ArrayList的实现原理

实际维护了一个Object的动态数组，动态是指能够动态扩容，由于是数组，所以支持随机快速访问。

指定位置插入、删除元素较慢，需要拷贝元素至新的数组中。

有序、可重复，能存储null元素。

##### 1.2 ArrayList的动态扩容

初始化时可以设定数组大小，未设置默认为0；第一次添加元素，默认设置容量为10。

- 在每次添加元素的时候，都会先调用ensureExplicitCapacity()方法，去判断elementData中容量，当容量不够时，调用grow()方法，去增大容量，首先通过 oldCapacity + (oldCapacity >> 1) 增大为elementData中原有容量的1.5倍，若容量仍然不够，则直接扩充至请求的容量。这样可以保证elementData每次都接近实际size的大小。

- 扩容后，会调用 Arrays.copyOf()方法，将元素移动至新的数组中，所以每次使用ArrayList添加元素时，因此添加元素时会慢一些；同时在remove()时，也会去移动元素。Arrays.copyOf()实现是调用System.arraycopy()方法。

- 删除元素时，不会调整数组容量。

##### 1.3 LinkedList的实现原理

基于双向链表实现，不用考虑扩容。

有序、可重复，能存储null元素。

查找或删除元素时需要从头或者尾节点遍历。优化，查找时会根据index，判断元素是靠近头节点还是尾结点，从而选择从头还是尾进行遍历。

##### 1.4 ArrayList与LinkedList对比，有什么异同

- ArrayList基于数组实现，内存连续；LinkedList基于双向链表实现，可以不连续，因此内存利用率更高；
- ArrayList随机访问效率更高；LinkedList随机相·对效率低，查找或删除元素时需要从头或者尾节点，一个个遍历下去。
- LinkedList批量插入快，只用找到指定位置，再改变前后索引即可。ArrayList需要移动其它元素至新的数组中；
- 若要进行大量的随机访问，使用ArrayList；如果要经常从表中插入或删除元素，可以使用LinkedList。
- 两者都不是线程安全

#### 2. CopyOnWriteArrayList

##### 2.1 CopyOnWriteArrayList特点

- 底部实现在ArrayList基础上，保证了线程安全。读写分离，写操作加锁，读操作无锁。
- 写操作时，使用ReentrantLock加锁，然后拷贝源数组至临时数组中，在临时数组中进行元素添加，添加完成后将临时数组赋值给源数组，实现元素的添加。
- CopyOnWriteArrayList迭代器会基于创建时的数据快照，进行增删改，能够正确结果

##### 2.2 不足

- 每次进行增删改，都会进行一次数据的拷贝，效率较低
- 由于写和读分别作用在新老不同容器上，写操作时，读取到的数据为老容器的数据

### 三 Set

#### 1.HashSet

##### 1.1 特点及底层实现

- 基于HashMap实现，元素存储在HashMap的key值上，value都为Object对象
- 无序，不能重复，能够存储null元素
- 非线程安全

#### 2.LinkedHashSet

##### 2.1 特点及底层实现

- 继承HashSet，提供了四个构造方法，调用父类构造器，基于LinkedHashMap实现，元素存储在LinkedHashMap的key值上，value都为Object对象，其它方法与HashSet相同，调用HashSet方法
- LinkedHashMap有序，因此，LinkedHashSet有序，不能重复，能够存储null元素
- 非线程安全

#### 3.TreeSet

基于TreeMap实现，能够排序。



### 四 Map

#### 1.HashMap

##### 1.1 简单介绍HashMap

HashMap用于存储键值对，基于数组和链表实现。元素存储无序，能存储null元素，非线程安全。

根据传入Key值计算对应的哈希值hashCode，找到对应的哈希桶（即数组中位置），数组中每个元素为一个单向链表，再将存储的对象放入链表中。并且能够自动扩容。

在实现上Java8版本进行了优化。



##### 1.2 什么是hash冲突？还有什么其他解决办法吗？

hash冲突是指插入元素时，根据hash算法计算key对应hash值，来算出位于数组的下标，然后将这个key，value插入到对应数组table[i]的位置。当两个key计算得到的hash值一样，即都位于同一个table[i]的位置，就发生了hash冲突，此时会在table[i]上形成一个数组或者树。

可以在hash之后，找到了数组位置后，再进行一次哈希。

##### 1.3 Java8中对hash算法进行了优化，为什么？

Java7中的hash算法会引起Hash Collision DoS （Hash碰撞的拒绝式服务攻击），即根据key值算出来的hash值一样，但value不一样，导致单向链表会不断增大，需要不断扩容rehash，会导致程序运行性能下降。Java7后面也出现了随机种子进行了优化。

因此，在Java8中对hash算法又进行了优化。

##### 1.4 Java8中插入元素流程？

初始化时，能够设置初始容量和负载因子，默认容量为16（会设置为2的幂），负载因子为0.75。在Java8里，

- 存入元素时，根据hash值，计算出元素位于数组table中的位置，若该位置为空，则直接新建一个节点，存入该位置；
- 若恰好哈希桶里的第一个元素就等于插入的元素，直接进行替换
- 若节点为红黑树的话，按照红黑树的插入方法
- 若节点为链表，按照链表的方法进行插入。如果在插入过程中元素个数大于变成树的预值（为8，因为设置为8，哈希桶中节点的分布符合泊松分布，链表长度为8的概率最小），然后将这个哈希桶变为树
- 如果插入个数大于扩容阀值，会进行扩容，扩容为原来的2倍。



##### 1.5 Java8中HashMap的改进？

主要是以下四点

- 由数组+链表变为了数组+链表+红黑树（当链表个数大于8时，会由链表变成红黑树。当从红黑树元素个数小于6后，又会退回链表，6和8的选择均为经验值。因为设置为8，哈希桶中节点的分布符合泊松分布，链表长度为8的概率最小。因为红黑树的插入和查询，相比于链表和完全平衡二叉树相对效率更高）
- hash算法优化（Java8中会将计算的hashcode的高16为不变，低16位为高16位和低位进行异或的值，来保证分布均匀，降低hash冲突的概率）
- 新节点插入链表顺序不相同（Java8插入链表时插入在链表尾部，保持顺序，Java7为头部，java7rehash后会改变链表顺序，并发条件下可能会成环）
- resize逻辑修改（Java7上rehash多线程会出现死循环，Java8进行了优化）



##### 1.6 什么时候进行扩容？负载因子设置为1会怎样？

当插入元素个数大于初始容量*负载因子时，需要进行扩容，扩容容量为现有的2倍。容量均需要为2的幂。

负载因子为1，意味着需要数组里所有元素都填满了，才能出现扩容，此时会出现大量的hash冲突，底层的红黑树也会变得复杂，对于再查询或者插入都不利。该种情况牺牲了时间来保证空间的充分利用。

当负载因子较小时，相反，空间会得不到充分利用。



##### 1.7 为什么数组大小（哈希桶的个数）要为2的幂？

计算插入元素位于数组的位置时，采用数组长度减1，然后与key计算的hash进行按位与（tab[i = (n - 1) & hash]）的方法。由于数组的个数均为2的幂，减1后，二进制将全为1，此时与hash值进行按位与，使得能够完整取到hash值的后几位（依据1的个数），因此，能够快速（体现在能够用按位与）的拿到数组的下标，同时分布还是均匀的。由于数组的个数不为2的幂，减1后，二进制将存在0的可能，使得该位置进行按位与一直0，导致数组该位置，永远不会放入元素。



##### 1.8 扩容reSize方法效率较低？

扩容分为两个过程，一个是扩大数组table容量，一个是将原先元素移动到新的数组中。

因此，对于需要频繁插入元素的操作，可指定容量，用空间换时间。



##### 1.9 为什么使用红黑树？

红黑树属于平衡二叉树，在插入新数据时能够通过左旋、右旋、变色这些操作来保持平衡，使得查找快。但是为了维持这种平衡，也要付出代价，所以也限制了当链表长度大于8再转化为红黑树，链表长度短时，不需要引入红黑树。

##### 1.10 说说对红黑树的理解？





##### 1.11 HashMap中扩容导致死循环问题？

Java7中多线程情况，扩容时可能会出现环形链表，导致死循环。

多线程情况可以使用ConcurrentHashMap解决。

Java8中已经优化，resize时，使用两个链表，都是尾插法。

[疫苗：JAVA HASHMAP的死循环](https://coolshell.cn/articles/9606.html)

##### 1.12  HashMap和HashTable有什么区别？

- HashMap是支持null键和null值的，而HashTable在遇到null时，会抛出NullPointerException异常。HashMap中，key为null时，计算所得Hashcode为0。
- 默认容量，扩容不同。Hashtable默认的初始大小为11，之后每次扩充为原来的2n+1。HashMap默认的初始化大小为16，之后每次扩充为原来的2倍。
- Hashtable是线程安全的，HashMap不是。Hashtable中方法采用synchronized修饰。Hashtable会慢些。
- HashTable为数组+链表组成，HashMap在Java8后，数组+链表+红黑树



##### 1.13 你通常用什么作为HashMap的key？

HashMap中键值均可以为null值。

一般用Integer、String这种不可变类当HashMap当key，而且String最为常用。

- 字符串不可变，创建时hashCode已被缓存，不需要重新计算，再在hashMap中使用速度更快

- String中已经重写了 equals() 和 hashCode()方法，HashMap中查找对象时需要用到equals()和hashCode()方法进行元素比较。

  

##### 1.14 一般在多线程的场景，会使用哪几种不同的方式去代替HashMap？

- 使用Collections.synchronizedMap(Map)创建线程安全的map集合（外层套一个synchronized）
- Hashtable（对数据操作的时候都会通过synchronized进行同步）
- ConcurrentHashMap



#### 2. ConcurrentHashMap

##### 2.1 简单介绍ConcurrentHashMap

线程安全的HashMap，基本结构一样。

相比于Java7，Java8中有了较大改变

- ConcurrentHashMap，key和value均不能为null，会抛空指针异常，HashMap中当key值为null，设定hash为null
- Java7中采用分段锁；Jsava8中采用了 CAS + synchronized来保证并发安全性，使用CAS来插入元素。

  

##### 2.2 put方法流程？

- 若第一次table为null，先初始化一个默认容量为16的数组table。若不为null，根据(n - 1) & hash)计算数组下标，若table[i]不存在元素，通过CAS写入元素
- 若table[i]存在元素，根据table[i]的hash值判断是否正在扩容，若在扩容，协助扩容
- 若不在扩容，使用synchronized同步，锁为table[i]元素,，根据table[i]判断是进行链表还是树方式进行节点插入（因为转化为树后节点hash值为负数），插入时利用CAS。（锁的选择进行了优化）
- 插入完成后，如果链表节点数为8个，将链表转化为红黑树



##### 2.3 get方法流程？

读元素不需要加锁

- 根据key计算hash值，找到数组位置table[i]
- 如果table[i]就为目标值，直接返回
- 若为红黑树，按照红黑树方式遍历
- 若为链表，按照链表方式



#### 3. TreeMap

能够根据key值进行排序，基于红黑树实现。



### 五 集合中其它常见问题

##### 5.1 什么是fail-fast机制？什么是fail-safe?

遍历集合时，进行增加、删除操作，导致抛出异常ConcurrentModificationException。

因为集合在遍历时，直接访问内部数据，为了防止在遍历时，被修改，使用变量modCount来记录已修改次数。迭代器每次的hasNext()和next()方法都会检查该modCount是否被改变，当检测到被修改时，抛出ConcurrentModificationException。

java.util包下的集合均是快速失败，ava.util.concurrent包下的容器，均为fail-safe安全失败。

**fail-safe**是指遍历时取得的数据均为原始数据的拷贝，即使原集合发生改变，也不会影响当前遍历。

##### 5.2 如何在遍历的同时删除ArrayList中的元素?

- 方法1，使用迭代器Iterator中remove方法，删除当前遍历到的元素
- 方法2，使用CopyOnWriteArrayList

