# currentHashMap-source-study
# 一、ConcurrentHashMap

​		ConcurrentHashMap是 J.U.C 包里面提供的一个线程安全并且高效的HashMap。

​		JDK8中采用的是位桶+链表/红黑树的方式，当某个位桶的链表的长度达到某个阀值的时候，这个链表就将转换成红黑树。当某个位桶长度大于8并且数组长度大于64时，此时位桶链表就将转换成红黑树。

**ConcurrentHashMap源码分析**

```java
Map<Object, Object> map = new ConcurrentHashMap<>();
map.put("key", "value");
//进入ConcurrentHashMap类public V put(K key, V value)方法，ConcurrentHashMap#put()
  map.put("key", "value");
	继续调用ConcurrentHashMap类方法
  1.final V putVal(K key, V value, boolean onlyIfAbsent)方法
    return putVal(key, value, false);
	2.private final Node<K,V>[] initTable()方法
    tab = initTable();
		//源码
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) { //首次时候，通过CAS来进行初始化
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
	3.static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i)方法
    else if ((f = tabAt(tab, i = (n - 1) & hash)) == null)
    table初始化结束后，put相关key到table。
	4.private final void addCount(long x, int check)方法
    //此处是核心，当并发极高时候，如何做到高性能计数，采取了分而治之思想，用一个CounterCell[]来记行计数
    addCount(1L, binCount);
		//源码
		private final void addCount(long x, int check) {
        CounterCell[] cs; long b, s;
        if ((cs = counterCells) != null ||
            !U.compareAndSetLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell c; long v; int m;
            boolean uncontended = true;
            if (cs == null || (m = cs.length - 1) < 0 ||
                (c = cs[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSetLong(c, CELLVALUE, v = c.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
          	//扩容
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);
                if (sc < 0) {  //有线程在扩容
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
              	//当第一个线程尝试进行扩容的时候，rs左移16位，相当于原本的二进制低位变成了高位 1000 0000 0001 1100 0000 0000 0000 0000,然后再+2 =1000 0000 0001 1100 0000 0000 0000 0000+10=1000 0000 0001 1100 0000 0000 0000 0010
                else if (U.compareAndSetInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
	5.private final void fullAddCount(long x, boolean wasUncontended)方法
    fullAddCount(x, uncontended);
		//首次初始化CounterCell[]
		else if (cellsBusy == 0 && counterCells == cs &&
                     U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
      boolean init = false;
      try {                           // Initialize table
        if (counterCells == cs) {
          CounterCell[] rs = new CounterCell[2];
          rs[h & 1] = new CounterCell(x);
          counterCells = rs;
          init = true;
        }
      } finally {
        cellsBusy = 0;
      }
      if (init)
        break;
    }
	6.static final int resizeStamp(int n)方法
    //生成一个唯一的扩容戳，resizeStamp(16)=32796
    //32796二进制:0000 0000 0000 0000 1000 0000 0001 1100
    int rs = resizeStamp(n);
    //源码
    static final int resizeStamp(int n) {
      	//首次n=16,Integer.numberOfLeadingZeros(n)表示返回2进制最高位之前0的个数
      	//16二进制: 10000,因为总位数是32位，所以返回就是27，32-5
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
    }
	7.private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab)方法
    transfer(tab, null);
		//源码
		private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
      	//将 (n>>>3 相当于 n/8) 然后除以 CPU 核心数。如果得到的结果小于 16，那么就使用 16 
      	// 这里的目的是让每个 CPU 处理的桶一样多，避免出现转移任务不均匀的现象，如果桶较少的话，默认一个 CPU(一个线程)处理 16 个桶，也就是长度为16的时候，扩容的时候只会有一个线程来扩容
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
        }
        int nextn = nextTab.length;
      	// 创建一个 fwd 节点，表示一个正在被迁移的 Node，并且它的 hash 值为-1(MOVED)，也就是前面我们在讲 putval 方法的时候，会有一个判断 MOVED 的逻辑。它的作用是用来占位，表示 原数组中位置 i 处的节点完成迁移以后，就会在 i 位置设置一个 fwd 来告诉其他线程这个位置已经 处理过了，
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
     	  // 首次推进为 true，如果等于 true，说明需要再次推进一个下标(i--)，反之，如果是 false，那么就不能推进下标，需要将当前的下标处理完毕才能继续推进
        boolean advance = true;
      	//判断是否已经扩容完成，完成就 return，退出循环
        boolean finishing = false; // to ensure sweep before committing nextTab
      	//通过 for 自循环处理每个槽位中的链表元素，默认 advace 为真，通过 CAS 设置 transferIndex 属性值，并初始化 i 和 bound 值，i 指当前处理的槽位序号，bound 指需要处理 的槽位边界，先处理槽位 15 的节点;
        for (int i = 0, bound = 0;;) {
          	// 这个循环使用 CAS 不断尝试为当前线程分配任务 
          	// 直到分配成功或任务队列已经被全部分配完毕
            // 如果当前线程已经被分配过 bucket 区域
            // 那么会通过--i 指向下一个待处理 bucket 然后退出该循环
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
              	//--i 表示下一个待处理的 bucket，如果它>=bound,表示当前线程已经分配过 bucket 区域
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) { //表示所有bucket已经被分配完毕
                    i = -1;
                    advance = false;
                }
              	//通过 cas 来修改 TRANSFERINDEX,为当前线程分配任务，处理的节点区间为 (nextBound,nextIndex)->(0,15)
                else if (U.compareAndSetInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
          	//i<0 说明已经遍历完旧的数组，也就是当前线程已经处理完所有负责的 bucket
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                if (finishing) { //如果完成了扩容
                    nextTable = null; //删除成员变量
                    table = nextTab; //更新table数组
                    sizeCtl = (n << 1) - (n >>> 1); //更新阈值(32*0.75=24)
                    return;
                }
             		// sizeCtl 在迁移前会设置为 (rs << RESIZE_STAMP_SHIFT) + 2 
                // 然后，每增加一个线程参与迁移就会将 sizeCtl 加 1，
                // 这里使用 CAS 操作对 sizeCtl 的低 16 位进行减 1，代表做完了属于自己的任务，释放线程
                if (U.compareAndSetInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                  	//第一个扩容的线程，执行 transfer 方法之前，会设置 sizeCtl =(resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)后续帮其扩容的线程，执行 transfer 方法之前，会设置 sizeCtl = sizeCtl+1,每一个退出 transfer 的方法的线程，退出之前，会设置 sizeCtl = sizeCtl-1 那么最后一个线程退出时:必然有sc == (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)，即 (sc - 2)== resizeStamp(n) << RESIZE_STAMP_SHIFT
										// 如果 sc - 2 不等于标识符左移 16 位。如果他们相等了，说明没有线程在帮助他们扩容了。也就是说，扩容结束了。
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                  	// 如果相等，扩容结束了，更新 finising 变量
                    finishing = advance = true;
                  	// 再次循环检查一下整张表
                    i = n; // recheck before commit
                }
            }
          	// 如果位置 i 处是空的，没有任何节点，那么放入刚刚初始化的 ForwardingNode ”空节点“
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
          	//表示该位置已经完成了迁移，也就是如果线程 A 已经处理过这个节点，那么线程 B 处理这个节点 时，hash 值一定为 MOVED
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
              	//对数组该节点位置加锁，开始处理数组该位置的迁移工作
                synchronized (f) {
                  	//再做一次校验
                    if (tabAt(tab, i) == f) {
                      	//ln 表示低位， hn 表示高位;接下来这段代码的作用 是把链表拆分成两部分，0 在低位，1 在高位
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                          	//遍历当前 bucket 的链表，目的是尽量重用 Node 链表尾部的一部分
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                          	//如果最后更新的runBit是0，设置低位节点
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else { //否则，设置高位节点
                                hn = lastRun;
                                ln = null;
                            }
                          	//构造高位以及低位的链表
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                          	//将低位的链表放在 i 位置也就是不动
                            setTabAt(nextTab, i, ln);
                          	//将高位链表放在 i+n 位置
                            setTabAt(nextTab, i + n, hn);
                          	//把旧table的hash桶中放置转发节点，表明此 hash 桶已经被处理
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                      	//红黑树的扩容部分
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
8.final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f)方法
  //如果正在扩容，那么增加一个线程帮助扩容
  tab = helpTransfer(tab, f);
```

**table标志sizeCtl**

<font color=red>sizeCtl</font> 这个标志是在 Node 数组初始化或者扩容的时候的一个控制位标识，负数代表正在进行初始化或者扩容操作。

-1: 代表正在初始化

为>0数字表示下一次扩容的大小

-N: 代表有N-1个线程正在进行扩容操作，这里不是简单的理解成 n 个线程，sizeCtl 就是-N

0 标识 Node 数组还没有被初始化，正数代表初始化或者下一次扩容的大小

**CounterCell[]标志cellsBusy**

CounterCell[] counterCells: counterCells数组，总数值的分值分别存在每个 cell 中

cellsBusy:标识当前 cell 数组是否在初始化或扩容中的CAS 标志位

cellsBusy=0 表示 counterCells 不在初始化或者扩容状态下

cellsBusy=1 表示 counterCells数组容量不够，线程竞争较大，正在扩容

**resizeStamp(int n)**

首次n=16,Integer.numberOfLeadingZeros(n)表示返回2进制最高位之前0的个数
16二进制: 10000,因为总位数是32位，所以返回就是27，32-5

生成一个唯一的扩容戳，resizeStamp(16)=32796

32796二进制:0000 0000 0000 0000 1000 0000 0001 1100

当第一个线程尝试进行扩容的时候，rs左移16位(rs=resizeStamp(n))，相当于原本的二进制低位变成了高位 1000 0000 0001 1100 0000 0000 0000 0000,然后再+2 =1000 0000 0001 1100 0000 0000 0000 0000+10=1000 0000 0001 1100 0000 0000 0000 0010
 U.compareAndSetInt(this, SIZECTL, sc, rs << RESIZE_STAMP_SHIFT) + 2)

**高 16 位代表扩容的标记、低 16 位代表并行扩容的线程数**

| 高 **RESIZE_STAMP_BITS** 位 | 低 **RESIZE_STAMP_SHIFT** 位 |
| --------------------------- | ---------------------------- |
| 扩容标记                    | 并行扩容线程数               |

 **这样来存储有优点**

1.首先在 CHM 中是支持并发扩容的，也就是说如果当前的数组需要进行扩容操作，可以由多个线程来共同负责
2.可以保证每次扩容都生成唯一的生成戳，每次新的扩容，都有一个不同的 n，这个生成戳就是根据 n 来计算出来的一个数字，n 不同，这个数字也不同

**第一个线程尝试扩容的时候，为什么是+2**

​		因为1表示初始化，2表示一个线程在执行扩容，而且对 sizeCtl 的操作都是基于位运算的， 所以不会关心它本身的数值是多少，只关心它在二进制上的数值，而 sc + 1 会在低16位上加 1。

**每一个CPU(一个线程)处理位桶数**

stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE

将 (n>>>3 相当于 n/8) 然后除以 CPU 核心数。如果得到的结果小于 16，那么就使用 16 ，这里的目的是让每个 CPU 处理的桶一样多，避免出现转移任务不均匀的现象，如果桶较少的话，默认一个 CPU(一个线程)处理 16 个桶，也就是长度为16的时候，扩容的时候只会有一个线程来扩容。

**为什么迁移后还能找到迁移数据**

f = tabAt(tab, i = (n - 1) & hash)

假设原table的n=16

1.某个 key 的 hash 值=9

9二进制: 0000 1001

(n-1) & hash 的算法 0000 1111 & 0000 1001 =0000 1001 ， 运算结果是 9

扩容以后，16 变成了 32，那么(n-1)的二进制是0001 1111

仍然以 hash 值=9 的二进制计算为例
0001 1111 & 0000 1001 =0000 1001 ，运算结果仍然是 9

2.某个 key 的 hash 值是 20，对应的二进制是0001 0100，仍然按照(n-1) & hash 算法，分别在 16 为长度和 32 位长度下的计算结果
16 位: 0000 1111 & 0001 0100 = 0000 0100

 32 位: 0001 1111 & 0001 0100 = 0001 0100

0000 0100(16位结果) + 16 = 0001 0100(32位结果)

实际就是在n=16位的结果上加了16就等于32位结果

因此在扩容后的新table能找到迁移后的高位链表数据。

