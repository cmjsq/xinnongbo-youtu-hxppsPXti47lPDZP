
**大纲**


**1\.JVM内存中的对象何时会被垃圾回收**


**2\.JVM中的垃圾回收算法及各算法的优劣**


**3\.新生代和老年代的垃圾回收算法**


**4\.避免本应进入S区的对象直接升入老年代**


**5\.Stop the World问题分析**


**6\.JVM垃圾回收的原理核心流程**


**7\.问题汇总**


 


**1\.JVM内存中的对象何时会被垃圾回收**


**(1\)什么时候会触发垃圾回收**


**(2\)被哪些变量引用的对象是不能回收的**


**(3\)Java中的对象有不同的引用类型**


**(4\)finalize()方法的作用**


 


**(1\)什么时候会触发垃圾回收**


Java系统运行时创建的对象都是优先分配在新生代里的，如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/e2993e20c320483f80991a5be2f955da~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=C8nbxlmLPLc%2BeSF5p5Ec3YWeaXw%3D)
如果新生代里的对象越来越多，当新生代快满的时候就会触发垃圾回收。把新生代里没有被引用的对象给回收掉，释放内存空间，而这就是新生代的垃圾回收触发时机。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/8c5013a388c9428ca488db9c591c6d01~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=BodLn4vOiMQUvssAC%2B%2F1wbTdz8I%3D)
接下来介绍触发垃圾回收时，到底是按什么样的规则来回收垃圾对象的。


 


**(2\)被哪些变量引用的对象是不能回收的**


当新生代快满了进行垃圾回收时，哪些对象能回收，哪些对象不能回收？JVM使用可达性分析算法来判定哪些对象可回收，哪些对象不可回收。这个算法会对每个对象都分析一下都有谁在引用它，然后一层一层往上去判断，看是否有一个GC Roots。


 


**一.最常见的就是对象被方法的局部变量引用**



```
public class Kafka {
    public static void main(String[] args) {
        loadReplicasFromDisk();
    }
    
    public static void loadReplicasFromDisk() {
        ReplicaManager replicaManager = new ReplicaManager();
    }
}
```

上述代码就是在一个方法中创建了一个对象，然后有一个局部变量引用了该对象，这种情况是最常见的。


 


此时如下图示：首先main()方法的栈帧入栈，然后调用loadReplicasFromDisk()方法，其栈帧也入栈，接着让局部变量replicaManager引用堆内存的ReplicaManager实例对象。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/87c412d7a7f243aea9fa1106ca123f3b~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=XdjqJEqmtni%2FRXaSY4XtearbWds%3D)
现在假设上图中ReplicaManager对象被局部变量给引用了，此时新生代满了要垃圾回收，会去分析ReplicaManager对象的可达性。发现它是不能被回收的，因为它还在被栈引用，也就是被局部变量replicaManager引用。


 


在JVM规范中，局部变量就是可以作为GC Roots的。一个对象只要被局部变量引用，就说明它有一个GC Roots，不能被回收。


 


**二.另外常见的就是对象被类的静态变量引用**



```
public class Kafka {
    public static ReplicaManager replicaManager = new ReplicaManager();
}
```

分析上面的代码，如下所示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/eff453685e01424fbb1b19ffe3b977ed~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=HTWuG5iO2k73YkJXmIMSonyWots%3D)
垃圾回收时进行分析，发现ReplicaManager对象被Kafka类的静态变量replicaManager引用了。而在JVM的规范里，静态变量也可以看做是一种GC Roots。只要一个对象被GC Roots引用了，就不会去回收它。所以不会回收被Kafka类静态变量引用的ReplicaManager对象。


 


因此一句话总结就是：只要对象被方法的局部变量、类的静态变量给引用了，就不会回收它们。


 


**(3\)Java中的对象有不同的引用类型**


Java中的对象有不同的引用类型，分别是：强引用、软引用、弱引用和虚引用。


 


**一.强引用**


就是类似下面的代码：



```
public class Kafka {
    public static ReplicaManager replicaManager = new ReplicaManager();
}
```

强引用就是最普通的代码，一个变量引用一个对象。只要是强引用的类型，那么垃圾回收的时候绝对不会去回收这个对象。


 


**二.软引用**


类似下面的代码：



```
public class Kafka {
    public static SoftReference replicaManager = 
        new SoftReference(new ReplicaManager());
}
```

把ReplicaManager对象用一个SoftReference软引用类型对象包裹起来，此时replicaManager变量对ReplicaManager对象的引用就是软引用了。


 


正常情况下垃圾回收是不会回收软引用对象的。但如果垃圾回收后，发现内存空间不够存放新对象，内存都快溢出了，就会把这些软引用对象给回收掉，哪怕它被变量引用着。但是因为它是软引用，所以还是要回收。


 


**三.弱引用**


类似下面的代码：



```
public class Kafka {
    public static WeakReference replicaManager = 
        new WeakReference(new ReplicaManager());
}
```

弱引用就与没有引用类似，如果发生垃圾回收，就会回收这个对象。


 


**四.虚引用**


可以暂时忽略它，因为很少用。


 


比较常用的就是强引用、软引用和弱引用。强引用就是代表绝对不能回收的对象。软引用就是对象可有可无，如果内存实在不够要OOM，才进行回收。弱引用就是每次发生垃圾回收的时候，都会进行回收。


 


**(4\)finalize()方法的作用**


有GC Roots引用的对象不能回收，没有GC Roots引用的对象可以回收。如果有GC Roots引用，但是如果是软引用或弱引用，也有可能被回收。


 


在回收环节，假设没有GC Roots引用的对象，一定马上被回收吗？其实不是，因为有一个finalize()方法可以拯救对象自己。如下代码所示：



```
public class ReplicaManager {
    public static ReplicaManager instance;
    
    @Override
    protected void finalize() throws Throwable {
        ReplicaManager.instance = this;
    }
}
```

假设有一个ReplicaManager对象准备要被JVM垃圾回收，如果它重写了Object类的finialize()方法，JVM会先调用其finalize()方法，看看在finalize()方法里是否会把自己这个实例对象给某个GC Roots变量。比如代码中就给了ReplicaManager类的静态变量，如果在finalize()方法重新让某GC Roots变量引用自己，那就不用被回收。


 


**(5\)问题**



```
public class Kafka {
    public static ReplicaManager replicaManager = new ReplicaManager();
}


public class ReplicaManager {
    public ReplicaFetcher replicaFetcher = new ReplicaFetcher();
}
```

上述代码如果发生垃圾回收，会回收ReplicaFetcher对象吗？不会。


 


因为ReplicaFetcher对象被ReplicaManager对象中的实例变量replicaFetcher引用，而ReplicaManager对象又被Kafka类的静态变量replicaManager引用。所以垃圾回收时，会发现它被GC Roots引用，于是不会回收它的。


 


**2\.JVM中的垃圾回收算法及各算法的优劣**


**(1\)复制算法的背景引入**


**(2\)一种不太好的垃圾回收思路**


**(3\)一个合理的垃圾回收思路**


**(4\)复制算法有什么缺点**


**(5\)复制算法的优化：Eden区和Survivor区**


**(6\)新生代垃圾回收的各种万一怎么处理**


 


**(1\)复制算法的背景引入**


针对新生代的垃圾回收算法，叫做复制算法。


 


**一.首先把新生代的内存分为两块**


如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/7ae0e227d4fe41329860bda1f7b504b7~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=PX3sXazzpbLivLAilSjgrvrV1HI%3D)
**二.接着loadReplicasFromDisk()创建一个对象**


此时就会分配新生代中的一块内存空间给这个对象，由main线程栈内存的loadReplicasFromDisk()方法栈帧的局部变量引用。



```
public class Kafka {
    public static void main(String[] args) {
        loadReplicasFromDisk();
    }
    
    public static void loadReplicasFromDisk() {
        ReplicaManager replicaManager = new ReplicaManager();
    }
}
```

如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/f0828b8f27ee476885abdfb25abcaf33~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=dhm%2FX%2BeFgH2nbopzi%2BPCOzl8kpU%3D)
**三.接着与此同时代码在不停地运行**


然后大量对象都分配在新生代的内存区域里，而且这些对象很快就失去局部变量或类静态变量的引用，成为垃圾对象。此时如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/4f221454137f4be5840aec97840e1951~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=qTZQYsG9MUbWM7RQ1zhenoi1lkA%3D)
**四.接着新生代内存区域基本都快满了**


再次要分配对象时，发现新生代里的内存空间不足了。那么此时就会触发YGC去回收掉新生代内存空间里的垃圾对象，那么回收的时候应该怎么做呢？


 


**(2\)一种不太好的垃圾回收思路**


假设采用的垃圾回收思路是：直接对上图中给新生代使用的那块内存区域中的垃圾对象进行标记。标记出哪些对象是可以被垃圾回收的，然后直接清空这些垃圾对象。按这种思路去回收，给新生代使用的内存区域在回收完毕后如下图示。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/56ccad2bc3384b1c8bf536d3cbfc70c5~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=Ts7MbpRncdozTS7s4mmN%2BdlwEyE%3D)
在新生代的内存区域会回收大量垃圾对象，保留一些被引用的存活对象。存活对象在这个内存区域里分布非常凌乱，从而造成内存碎片。这些内存碎片的大小不一，有的可能很大，有的可能很小。当内存碎片太多就会造成内存浪费的问题，比如打算分配一个新对象，尝试在上图那块被使用的内存区域里分配。但由于内存碎片太多，虽然所有的内存碎片加起来有很大的一块内存，但因这些内存都是分割的，所以导致没有完整的内存空间来分配新对象。


 


因此直接清除一块内存空间里的垃圾对象，保留存活对象，不太可取。这种方法会造成内存碎片太多，造成大量的内存浪费。


 


**(3\)一个合理的垃圾回收思路**


那么能不能用一种合理的思路来进行垃圾回收呢？可以，这时上图中一直没派上用场的另外一块空白的内存区域就出场了。首先并非直接对已使用的内存区域回收全部垃圾，然后保留存活对象。而是先标记出该内存区域哪些对象是不能进行垃圾回收的、需要存活的，然后把那些需要存活的对象转移到另外一块空白的内存。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/fd1ee920554f4195a9dcac1cec671b18~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=5FVQwokDxYUVOkce2kYsuqTlOng%3D)
通过把存活对象先转移到另外一块空白内存区域，就可以让这些对象都比较紧凑地、按顺序排列在内存里，这样就可以让转移到的那块内存区域没有内存碎片了。然后转移到的那块内存区域，也会多出一大块连续的、可用的内存空间。此时就可以将新对象分配在那块连续内存空间里了，如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/efa57dc95c1a418c9fd04b83d1e41cc8~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=jkQ%2FfMfldybERTAflX2Bo%2BAH6OQ%3D)
这时再把原来使用的那块内存区域中的垃圾对象全部回收掉，这样就可以空出一大块内存区域了。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/d6b48157ea1848699778ec5b8c450431~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=661FDo2ktk1JYNFtWi1Vrwxiysc%3D)
这就是所谓的"复制算法"：把新生代内存划分为两块内存区域，然后只使用其中一块内存。等该内存快满时，就把里面存活的对象一次性转移到另外一块内存，这样就能保证没有内存碎片了。接着一次性回收原来那块内存区域的对象，从而再次空出一块内存区域。两块内存区域就这样重复循环使用。


 


**(4\)复制算法有什么缺点**


复制算法的缺点其实非常的明显：假设给新生代1G的内存空间，那么只有512M的内存空间是可以用的，另外512M的内存空间是一直要放在那里空着的。然后512M内存空间满了，就把存活对象转移到另一块512M内存空间去。也就是只有一半的内存可以用，这样的算法显然对内存的使用效率太低。


 


**(5\)复制算法的优化：Eden区和Survivor区**


系统运行时对JVM内存的使用就是：将不断创建的对象分配在新生代里，这些对象中的绝大部分很快就会没被引用而成为垃圾对象。接着过一段时间新生代满了，就会回收掉这些垃圾对象，从而空出内存空间给其他对象使用。


 


其实在一次新生代垃圾回收后：99%的对象可能都会被垃圾回收，只有1%的对象存活下来。所以JVM对复制算法做出如下优化，把新生代内存区域划分为三块：1个Eden区，2个Survivor区。其中Eden区占80%内存空间，每块Survivor区占10%内存空间。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/e5758cb88f1d43a9a08f896a98cf6456~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=YiJEIXs1%2BK30gZ9uLyXDr0v8bgw%3D)
平时可以使用的就是Eden区和其中一块Survivor区，所以有90%的内存是可以使用的。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/16521ca2ee6e41ce894b7b60fbc173eb~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=GRUTNyuHkkhbmYtMXmFfuhAGYAA%3D)
刚开始都是在Eden区给对象分配内存，如果Eden区满了就会触发垃圾回收，此时就会把Eden区中存活的对象一次性转移到一块空着的Survivor区。接着Eden区就会被清空，然后再次分配新对象到Eden区里。然后就会如上图示，Eden区和一块Survivor区里是有对象的，其中Survivor区里放的是上一次Young GC后存活的对象。


 


如果随后Eden区又满了，那么会再次触发Young GC。这时会把Eden区和放着上次YGC存活对象的Survivor区的所有存活对象，都转移到另外一块Survivor区里。


 


这样做最大的好处是：只有10%的内存空间是被闲置的，90%的内存都被使用上了。无论是垃圾回收的性能、内存碎片的控制、内存使用效率，都非常好。


 


**(6\)新生代垃圾回收的各种万一怎么处理**


万一垃圾回收后，存活的对象超过了10%内存空间，Survivor区放不下。


 


万一分配一个大对象，新生代找不到连续内存空间存放，应怎么处理？


 


一个存活对象在新生代Survivor区来回移动多少次才会被转移到老年代？


 


**3\.新生代和老年代的垃圾回收算法**


**(1\)新生代的垃圾回收算法与内存区域划分**


**(2\)躲过15次GC之后进入老年代**


**(3\)对象的动态年龄判断规则**


**(4\)大对象直接进入老年代**


**(5\)YGC后存活对象太多无法放入S区的处理**


**(6\)老年代空间分配担保机制**


**(7\)老年代垃圾回收算法**


**(8\)什么是JVM优化**


 


**(1\)新生代的垃圾回收算法与内存区域划分**


一.首先代码运行过程中会不断创建各种各样的对象


这些对象会先放到新生代的Eden区和Survivor1区。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/eaa425094b0649a2a076bb096fee4ac6~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=7ygA8ZmdqStfJpbC6RltiFF3dg8%3D)
二.接着假如新生代的Eden区和Survivor1区都满了


此时就会触发Young GC，把存活对象转移到Survivor2区。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/a3c23f05e19e4b2aa4a5e868055daa25~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=ejWzCL1j0NLW%2FQwgM2D0dqC9nBI%3D)
三.然后使用Eden区和Survivor2区来存放新的对象


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/4ac610618d9b485b91f90029ef2c267c~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=K5yLWxyB2dN15ESJRSVRYBh6aBQ%3D)
接下来看看各种情况下，对象是如何进入老年代的，以及老年代的垃圾回收算法是怎么样的。


 


**(2\)躲过15次GC之后进入老年代**


按照上面图示过程：系统刚启动时，创建的各种对象基本都会分配在新生代里的。然后系统继续运行，新生代满了，此时就会触发Young GC。可能1%的少量存活对象会转移到空着的Survivor区中。然后系统继续运行，继续在Eden区里分配各种对象。但系统中会有一些对象是长期存在的，它是不会轻易的被回收掉的。如下代码所示：



```
public class Kafka {
    private static ReplicaManager replicaManager = new ReplicaManager();
}
```

只要Kafka类还存在，则其静态变量就会长期引用ReplicaManager对象。所以无论新生代发生多少次垃圾回收，类似这种对象都不会被回收掉。这类对象每次在新生代里躲过一次GC被转移到S区，其年龄就会\+1。默认当对象年龄达到15岁时(即躲过15次GC)，就会转移到老年代里。


 


具体多少岁进入老年代，可设置JVM参数\-XX:MaxTenuringThreshold。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/ec293c13c9fc4aaa83e04995ba406c94~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=Ozi0J9%2Bw6CPA0Qx1nFERQ7BC0SM%3D)
**(3\)对象的动态年龄判断规则**


让一个对象进入老年代，其实也可以不用等15次GC让对象年龄到15岁。而这判断依据就是动态年龄判断规则：在存放一批对象的S区里，如果这批对象总大小已大于该区大小的50%，那么此时大于等于这批对象年龄的对象，就可以直接进入老年代。


 


比如在S区内，年龄1 \+ 年龄2 \+ 年龄3 \+ 年龄n的对象和大于S区的50%。此时年龄n及以上的对象会进入老年代，不一定需要n达到15岁。


 


所以动态年龄判断规则有个推论：如果S区中的同龄对象大小超过S区内存的一半，那么这些同龄对象就要直接升入老年代。


 


假设如下图的Survivor2区有两个对象，其对象年龄都一样，都是2岁。然后其总大小超过5%内存，即超过了Survivor2区的10%内存大小一半。这时Survivor2区里大于等于2岁的对象，就可以全部进入老年代里了。这就是所谓的动态年龄判断规则，动态年龄判断规则会让一些新生代的对象提前年龄进入老年代。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/4a85738421f04ccb8ad47a6e8563332b~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=LbKvo3a6wTZkRaRxZXANTAzRdzU%3D)
**总结：**


这个动态年龄判断规则运行时会按如下的逻辑处理：年龄1 \+ 年龄2 \+ 年龄n的对象，大小总和超过了Survivor区的50%，此时就会把年龄为n及以上的对象都放入老年代。


 


无论是年龄15岁进入老年代规则，还是动态年龄判断规则，都是希望那些可能是长期存活的对象，尽早进入老年代。


 


**(4\)大对象直接进入老年代**


参数\-XX:PretenureSizeThreshold设置为1048576字节，意思是如果要创建一个大于1M的大对象，就会直接把这个大对象放到老年代，无须经过新生代。


 


之所以这么做，就是要避免新生代里出现大对象，然后屡次躲过GC。还得对它在两个Survivor区里进行来回复制多次，之后才进入老年代。这么大的一个大对象在内存里来回复制，必然耗费时间。所以这也是一个对象进入老年代的规则。


 


**(5\)YGC后存活对象太多无法放入S区的处理**


如果在YGC后存活对象太多，比如存活对象已超Eden区内存的15%，如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/8f70a6bc6bde4a659b581d8761b79c0c~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=6DyLCh%2Bae7xi7x7UIPy%2Fvw4Llos%3D)
那么此时没办法放入Survivor区，就会把这些对象都直接转移到老年代，如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/0d3a9e10e67449109e6a8f1636645b3c~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=rqVhcoKjEh3815CSN3U1XISK4Zw%3D)
**(6\)老年代空间分配担保机制**


如果新生代有大量对象存活，Survivor区放不下，必须转移到老年代。而此时老年代的空间也不够存放这些对象，那该怎么办？


 


首先在执行任何一次YGC前，JVM会先检查一下老年代的可用内存空间，判断老年代的可用内存空间是否大于新生代所有对象总大小。


 


做这个检查是因为最极端情况下，新生代YGC后所有对象都存活下来，新生代所有对象都要进入老年代。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/018f03076a0947d7910f438604c982a7~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=mO2QbqtvUnS89ie4k5oZLA2G5rw%3D)
**一.如果在执行YGC前发现老年代的可用内存大于新生代所有对象大小**


此时就可以放心大胆的对新生代发起一次YGC，因为即使YGC后所有对象都存活，S区放不下，也可以转移到老年代。


 


**二.如果在执行YGC前发现老年代的可用内存小于新生代所有对象大小**


那么这时就有可能在YGC后新生代的对象全部存活，然后全部要转移到老年代，而老年代空间又不够。


 


所以在执行YGC前，发现老年代的可用内存小于新生代全部对象大小，就会判断参数\-XX:\-HandlePromotionFailure是否被设置了。如果设置了\-XX:\-HandlePromotionFailure参数，就会继续进行判断：老年代可用内存是否大于之前每次YGC后进入老年代的对象的平均大小。


 


举个例子：之前每次YGC后，平均有10M对象会进入老年代，说明这次YGC过后也很可能有10M对象会进入老年代。而此时老年代可用内存大于10M，此时老年代空间也很可能是够的。


 


**情况一：**如果老年代可用内存小于历次YGC转移来的对象平均大小或\-XX:\-HandlePromotionFailure参数没设置，此时会触发一次FGC。FGC会对老年代进行垃圾回收，尽量腾出一些内存空间，然后再YGC。FGC就是对老年代进行垃圾回收，同时一般也对新生代进行垃圾回收。


 


**情况二：**如果\-XX:\-HandlePromotionFailure参数已经设置且老年代内存大于历次YGC转移的对象平均大小，此时就会尝试YGC。但是此时进行的YGC有如下三种可能。


 


**第一种可能**：YGC过后，剩余的存活对象小于S区的大小，此时存活对象进入S区。


 


**第二种可能：**YGC过后，剩余的存活对象大于S区的大小，但小于老年代可用内存大小，此时存活对象就直接进入老年代。


 


**第三种可能：**YGC过后，剩余的存活对象大于S区大小，也大于老年代可用内存大小。此时老年代也放不下这些存活对象，就会发生Handle Promotion Failure。这时就会触发一次FGC，把老年代里没被引用的对象给回收掉，然后才可能让这次YGC过后剩余的存活对象进入老年代中。


 


整个判断流程如下：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/9dd09a4b93964446a45e9ae4df7e451d~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=%2B8t2852xTkxfotf0xElSs4pqx00%3D)
如果FGC过后，老年代还是没有足够空间存放YGC过后的剩余存活对象。那么此时就会导致所谓的OOM内存溢出了，因为内存实在是不够了，还是要不停的往里面放对象，自然就崩溃了。


 


**(7\)老年代垃圾回收算法**


对老年代触发垃圾回收的时机，一般就是两个。


 


时机一：在YGC前，检查发现YGC后可能要进入老年代的对象太多了，老年代放不下这么多存活对象，此时可能要提前触发一次FGC，然后再进行YGC。这里有3种情况：参数是否设置 \+ 历次YGC转移进老年代的对象平均大小。


 


时机二：在YGC后，发现剩余对象太多，老年代放不下。此时必须马上触发FGC然后再进行YGC。


 


那么对老年代进行垃圾回收采用的是什么算法呢？老年代采取的是标记\-整理算法。首先标记出来老年代当前存活的对象，这些对象可能是东一个西一个。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/6ae1b248e50e442386979cae3b23fe1b~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=PWzCSi2%2BVJqMyvTddQvpG3YI1dk%3D)
接着会让这些存活对象在内存里进行移动，把存活对象都移到一边去。让存活对象紧凑靠在一起，避免垃圾回收后出现过多内存碎片，然后再一次性把垃圾对象都回收掉。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/27e2bf2c8aaf468e859434ac7c233d1b~tplv-obj.image?lk3s=ef143cfe&traceid=2024122723013551734767D479E2B48283&x-expires=2147483647&x-signature=OPnZVP5v8sHByWMNnl0sWJ8wDio%3D)
需要注意的是：老年代的垃圾回收速度至少比新生代的垃圾回收速度慢10倍。如果系统频繁出现老年代FGC，会严重影响系统性能，出现频繁卡顿。


 


**(8\)什么是JVM优化**


所谓JVM优化，就是尽可能让对象都在新生代里分配和回收。尽量别让太多对象频繁进入老年代，避免频繁对老年代进行垃圾回收。同时给系统充足的内存大小，避免新生代频繁地进行垃圾回收。


 


**4\.避免本应进入S区的对象直接升入老年代**


**(1\)一个日处理上亿数据的计算系统**


**(2\)这个系统多久会塞满新生代**


**(3\)触发YGC时会有多少对象进入老年代**


**(4\)系统运行多久老年代就会被填满**


**(5\)这个系统运行多久，老年代会触发1次FGC**


**(6\)该案例应该如何进行JVM优化**


**(7\)垃圾回收器简介**


 


**(1\)一个日处理上亿数据的计算系统**


当时团队里自研的一个数据计算系统，日处理数据量在上亿的规模。这个系统会不停的从MySQL数据库以及其他数据源里提取大量的数据，然后加载到自己的JVM内存里来进行计算处理，如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/aa600f51783a49f88924189b9d2b874e~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=YQ2CSFIgqQc5oQ3vcwiHMOKPxJY%3D)
这个数据计算系统会不停的通过SQL语句和其他方式，从各种数据存储中提取数据到内存中来进行计算，大致当时的生产负载是每分钟需要执行500次数据提取和计算的任务。


 


由于这是一套分布式运行的系统，所以生产环境部署了多台机器。每台机器大概每分钟负责执行100次数据提取和计算的任务。每次提取大概1万条数据到内存计算，平均每次计算大概耗费10秒时间。然后每台机器4核8G，新生代和老年代分别是1\.5G和1\.5G的内存空间。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/53ccad11761f44bda166e6f9a2d6f854~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=pqXcCq87P6M7lFgCihoLYEqLnA8%3D)
**(2\)这个系统多久会塞满新生代**


现在明确了一些核心数据，那么该系统到底多久会塞满新生代内存空间。既然每台机器上部署的该系统实例，每分钟会执行100次数据计算任务。每次1万条数据需要计算10秒，故一台机器大概开启15个线程去执行。


 


那么先来看看每次1万条数据大概会占用多大的内存空间。这里每条数据都是比较大的，每条数据大概包含了20个字段，可以认为平均每条数据的大小在1K左右，那么每次计算任务的1万条数据就对应了10M大小。


 


如果新生代按照8 : 1 : 1的比例来分配Eden和两块Survivor的区域。那么Eden区就是1\.2G，每块Survivor区域在100M左右。如下图示：


![](https://p26-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/4c1ddee7dc0f409f96e64009ab01fc82~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=fn%2FSELyD2UtU%2BXpcc54VbDZkbIg%3D)
由于每次执行一个计算任务，就要提取1万条数据到内存，每条数据1K。所以每次执行一个计算任务，JVM会在Eden区里分配10M的对象。由于一分钟需要执行大概100次计算任务，所以基本上一分钟过后，Eden区里就全是对象，基本全满了。因此，新生代里的Eden区，基本上1分钟左右就迅速填满了。


 


**(3\)触发YGC时会有多少对象进入老年代**


假设新生代的Eden区在1分钟后都塞满对象了，然后继续执行计算任务时，必然导致需要进行YGC回收部分垃圾对象。


 


**一.在执行YGC前会先进行检查**


首先会看老年代的可用内存空间是否大于新生代全部对象。此时老年代是空的，大概有1\.5G的可用内存空间，而新生代的Eden区大概有1\.2G对象。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/aba2b77a844646c19390088e11fd8df1~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=2WhKxvh6E0sRmeVusmHThP9ErzE%3D)
于是会发现老年代的可用内存空间有1\.5G，新生代的对象总共有1\.2G。一次YGC过后，即使全部对象都存活，老年代也能放的下，所以此时就会直接执行YGC。


 


**二.执行YGC后，Eden区里有多少对象是存活的无法被垃圾回收的**


由于新生代的Eden区在1分钟就塞满对象需要YGC了，而1分钟内会执行100次任务，每个计算任务处理1万条数据需要10秒钟。


 


假设执行YGC时，有80个计算任务都执行结束了，但还有20个计算任务共计200M的数据还在计算中。那么此时就有200M的对象是存活的，不能被垃圾回收，所以总共有1G的对象可以进行垃圾回收。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/a3b08b5192ce42aaa703b8aec88c2fb5~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=ZBVKMO9LxdZIQQZxx43Ki93L8Is%3D)
**三.此时执行一次YGC会回收1G对象，然后出现200M的存活对象**


这200M的存活对象并不能直接放入S区，因为一块S区只有100M大小。此时老年代会通过空间分配担保机制，让这200M对象直接进入老年代。直接占用老年代里的200M内存空间，然后对Eden区进行清空。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/1269c62f69764ea3959cbf743e3ca852~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=%2BWKjkQ1gBnklrzJq17XmfXeRelU%3D)
**(4\)系统运行多久老年代就会被填满**


按照上述计算，每分钟都是一个轮回，大概算下来是每分钟都会把新生代的Eden区填满。然后触发一次YGC，接着大概会有200M左右的数据进入老年代。


 


假设2分钟过去了，此时老年代已经有400M内存被占用了，只有1\.1G的内存可用，此时老年代的可用内存空间已经开始少于新生代的内存大小了。所以如果第3分钟运行完毕，又要进行YGC，会做如下检查：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/76701113ecac4335adbc7c966e447c5b~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=NcJNQRhFge8SWDsESxJwtmLij5s%3D)
**一.首先检查老年代可用空间是否大于新生代全部对象**


此时老年代可用空间1\.1G，新生代对象有1\.2G。那么此时假设一次YGC过后新生代对象全部存活，老年代是放不下的。


 


**二.接着检查HandlePromotionFailure是否打开**


如果\-XX:\-HandlePromotionFailure参数被打开了(一般都会打开)，此时会进入下一个检查：老年代可用空间是否大于历次YGC过后进入老年代的对象的平均大小。


 


前面已计算过：大概每分钟执行一次YGC，每次200M对象进入老年代。此时老年代可用1\.1G，大于每次YGC进入老年代的对象平均大小200M。所以推测，本次YGC后大概率还是有200M对象进入老年代，1\.1G足够。因此这时就可以放心执行一次YGC，然后又有200M对象进入老年代。


 


**三.转折点大概在运行了7分钟后**


执行了7次YGC后，大概1\.4G对象进入老年代。老年代剩余空间不到100M了，几乎满了。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/841b0d88a9fb4c41b23161e3a8ab571e~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=Zl26kokRTVfRULSDPIsmUaSmTeE%3D)
**(5\)这个系统运行多久，老年代会触发1次FGC**


大概在第8分钟运行结束时，新生代又满了。执行YGC之前进行检查，发现老年代此时只有100M的可用内存空间，比历次YGC后进入老年代的200M对象要小，于是直接触发一次FGC。FGC会把老年代的垃圾对象都给回收掉。


 


假设此时老年代被占据的1\.4G空间里，全部都是可以回收的对象，那么此时就会一次性把这些对象都给回收掉。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/41d690be7a9349b0a72b8d5b5bafa935~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=qSMjsDEyGS07ZrcHEaY1W2pvVFg%3D)
然后执行完FGC后，还会接着执行YGC。此时Eden区情况，200M对象再次进入老年代。之前的FGC就是为这些新生代本次YGC要进入老年代的对象准备的，如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/c5c8339fe33e46359eb9ab20e475c2ec~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=NglSYNjY6ed43gK0u4b1itxINIk%3D)
所以按照这个运行模型：平均八分钟会发生一次FGC，这个频率就很高了。而每次FGC速度都是很慢的、性能很差。


 


**(6\)该案例应该如何进行JVM优化**


通过上述这个案例，可以清楚看到：新生代和老年代应该如何配合使用，什么情况下会触发YGC和FGC，什么情况下会导致频繁YGC和FGC。


 


如果要对这个系统进行优化，因为该系统是数据计算系统，每次YGC时必然有一批数据没计算完毕。按现有的内存模型，最大问题就是每次Survivor区域放不下存活对象。


 


所以可以对生产系统进行调整，增加新生代内存比例，3G堆内存的2G分配给新生代，1G留给老年代。这样S区大概就是200M，每次刚好能放得下YGC过后存活的对象。如下图示：


![](https://p26-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/6e4c76582acf464585684082a2a07181~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=fEjAyE41U18QBBab0ykwGxpkfwQ%3D)
只要每次YGC过后200M存活对象可以放进Survivor区域，那么等下次YGC时，这个S区的对象对应的计算任务早就结束可回收了。


 


比如此时Eden区里1\.6G空间被占满了，然后Survivor1区里有200M上一轮YGC后存活的对象。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/ac706d86830e4102be8c265a5ed17404~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=ReftV%2F8edFYwDAJzcpGBr3B2isc%3D)
此时执行YGC就会把Eden区里1\.6G对象回收掉，Survivor1区的200M对象也会被回收掉。而Eden区里剩余的200M存活对象便会被放入到Survivor2区里，如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/e11cb30a09dd4c7095720826fa904915~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=xabTpM4lTSHrwqBehX90uDslncA%3D)
以此类推，基本就很少有对象会进入老年代，老年代的对象也不会太多，这样成功把生产系统老年代FGC的频率从几分钟一次降低到几小时一次。大幅度提升了系统的性能，避免了频繁FGC对系统运行的影响。


 


前面说过一个动态年龄判定升入老年代的规则：如果S区中的同龄对象大小超过S区内存的一半，就要直接升入老年代。


 


所以这里的优化方式仅仅是做一个示例说明，实际S区200M还是不行。但核心是要增加S区大小，让YGC后的对象进入S区，避免进入老年代。


 


实际上为了避免动态年龄判定规则把S区中的对象直接升入老年代，如果新生代内存有限，那么可以调整"\-XX:SurvivorRatio\=8"参数。比如降低Eden区的比例(默认80%)，给两块S区更多的内存空间。让每次YGC后的对象进入S区，避免因为动态年龄规则把它们升入老年代。


 


**(7\)垃圾回收器简介**


新生代和老年代进行垃圾回收时都是用垃圾回收器进行回收的，不同的区域会用不同的垃圾回收器。


 


**JVM常见的垃圾回收器以及各自的特点如下：**


 


一.Serial和Serial Old垃圾回收器


分别用来回收新生代和老年代的垃圾对象。工作原理就是单线程运行，垃圾回收时会停止我们系统的其他工作线程。让我们系统直接卡死不动，让它们进行垃圾回收。现在的后台Java系统几乎不用这种垃圾回收器了。


 


二.ParNew和CMS垃圾回收器


ParNew是用在新生代的垃圾回收器，CMS是用在老年代的垃圾回收器。采用多线程并发机制，性能更好，一般是线上生产系统的标配组合。


 


三.G1垃圾回收器


统一收集新生代和老年代，采用了更加优秀的算法和设计机制。


 


**5\.Stop the World问题分析**


**(1\)新生代GC的场景**


**(2\)YGC的时候是否还能继续创建新的对象**


**(3\)JVM的痛点——Stop the World**


**(4\)Stop the World造成的系统停顿**


**(5\)不同的垃圾回收器的不同的影响**


 


**(1\)新生代GC的场景**


**一.首先新生代的内存会分为Eden区和两个S区**


如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/4efb19b0c72444f7af5adae3b86bba18~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=techlT%2FG2f59HFZsO0cEJRbN9%2FA%3D)
**二.然后系统不停运行把Eden区给塞满了**


如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/4450a34a1923402faf5b8c8c6f40e5c8~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=AjEsE8sXlLgxgfeuopm%2FHh8Cyp0%3D)
**三.这时就会触发YGC**


执行垃圾回收会有专门的垃圾回收线程负责，而且对不同的内存区域也会有不同的垃圾回收器。即垃圾回收线程和垃圾回收器会配合起来，使用相应的垃圾回收算法对指定的内存区域进行垃圾回收。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/1245f08d7bbe4b638590088042b17406~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=VLRC7%2BhvX5Xzpb8tXBfuhMSuhhs%3D)
进行垃圾回收时会通过一个后台运行的垃圾回收线程来执行具体逻辑，比如针对新生代可能会用ParNew垃圾回收器来进行回收。而ParNew垃圾回收器针对新生代采用的是复制算法来进行垃圾回收，这时垃圾回收器会先把Eden区中的存活对象标记出来，全部转移到S1区，再一次性清空Eden区中的垃圾对象。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/6900a963b5014d36a1bb03407e4e12a2~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=09HScnVfnbcI1%2Fr9igO9nUpJ8Uk%3D)
**四.接着系统继续运行并在Eden区分配新对象**


如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/aa54a8183a884433941e50ae00fe1019~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=j3MlE0SWDC%2BKx2ovqnOQQchUY2Y%3D)
**五.当Eden区再次塞满时就又会触发YGC**


此时依然是垃圾回收线程执行垃圾回收器中的复制算法逻辑，先去Eden区和Survivor1区中标记出存活的对象，再一次性把存活对象转移到Survivor2，接着把Eden和Survivor1的垃圾对象都回收掉。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/ba5a3c6b6b55424cadbc4faa3cb57671~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=vWTK5pNjL%2FjaWIG9HFCvwPnNuRg%3D)
**(2\)YGC的时候是否还能继续创建新的对象**


在YGC时，Java系统在运行期间还能不能继续在新生代里创建新的对象？假设在YGC期间还可以允许系统继续在新生代的Eden区里创建新的对象。那么情况会如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/5a3fcdf9d2ea48f499e36cb67260b80e~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=GV7CDqy2w4VS69lImW9St8S%2BuaE%3D)
根据上图所示：如果垃圾回收器一边把Eden和S2里的存活对象标记出来转移到S1，然后一边还在把Eden和S2里的垃圾对象都清理掉，而这时系统程序还不停在Eden里创建新对象。这些新对象有的很快就成了垃圾对象，有的还在被引用成为存活对象。


 


那么对于系统程序新创建的这些对象：怎么让垃圾回收器去持续追踪它们的状态？怎么想办法在这次垃圾回收中把新对象中的那些存活对象转移到S2中？怎么想办法把新创建的对象中的垃圾都给回收掉？


 


如果要在JVM中去解决这一系列的问题，那么就会很复杂、成本极高、且很难做到。所以在YGC垃圾回收的过程中：如果还允许继续不停地在Eden里创建新的对象，是不合适的。


 


**(3\)JVM的痛点——Stop the World**


所以使用JVM最大的痛点，就是垃圾回收的过程。在垃圾回收时，尽可能地让垃圾回收器专心致志的干工作。不能随便让Java系统继续创建新对象，此时JVM会在后台进入STW状态。JVM会直接停止Java系统的所有工作线程，不再运行Java系统上的代码。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/2b65153757f645a98a95008f9ec62d4d~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=FXD0IJvrJdYCM36W8XPUKlEAio0%3D)
这样Java系统暂停运行，不再创建新的对象。同时让垃圾回收线程尽快完成垃圾回收的工作，也就是标记和转移Eden以及Survivor2的存活对象到Survivor1中，然后尽快一次性回收掉Eden和Survivor2中的垃圾对象。如下图示：


![](https://p26-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/946c76236d3b41ee8ebb4153e5267189~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=JaUoWST7k0e9R%2B5K2wbmCZL3T60%3D)
接着一旦垃圾回收完毕，就可以恢复运行Java系统的工作线程了，然后Java系统就可以继续在Eden中创建新的对象。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/6bbf3280c2ec46aa930bc6d150e3dac9~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=kDY4B%2BRDfPjtIRquwdLev3nHBGE%3D)
**(4\)Stop the World造成的系统停顿**


一.YGC停顿ms级


假设YGC要运行100ms，那么可能就会导致Java系统直接停顿100ms不能处理任何请求，在这100ms期间用户发起的所有请求都会出现短暂的卡顿。


 


如果是一个Web系统就可能导致用户从网页或者APP上点击一个按钮，平时只要几十ms就可以返回响应，现在因为JVM正在执行YGC，暂停所有的工作线程，导致用户请求过来到响应返回需要等待几百毫秒。


 


二.FGC停顿秒级


因为内存分配不合理，导致对象频繁进入老年代，平均八分钟一次FGC。FGC是最慢的，有时一次回收要进行几秒～几十秒，极端下可能几分钟。而一旦频繁FGC，每隔八分钟系统可能就卡死几十秒，在几十秒内任何请求全部无法处理，用户体验极差。


 


所以，无论是YGC还是FGC，都尽量不要频率过高、避免持续时间过长。避免影响系统正常运行，这也是使用JVM过程中一个最需要优化的地方。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/508f94e78df74a15a685b22a26f9a400~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=voj9qWhzqjjOXGV%2BSLk0Zvf4KhY%3D)
**(5\)不同的垃圾回收器的不同的影响**


比如对新生代的回收：Serial用一个线程进行垃圾回收，然后暂停系统工作线程，一般很少用。


ParNew是常用的新生代垃圾回收器，它针对多核CPU做了优化。ParNew会使用多个线程进行垃圾回收，可缩短回收时间。


 


大致原理图如下：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/4639a2be2f7b4ef6b1fa3926e18c3c7d~tplv-obj.image?lk3s=ef143cfe&traceid=202412272301368C1D607614F3902CF218&x-expires=2147483647&x-signature=jfvV2HcsP49OnBkaG4mklT52atw%3D)
 


**6\.JVM垃圾回收的原理核心流程**


梳理GC的全流程：


一.什么时候会尝试触发YGC


二.YGC前如何检查老年代大小，涉及哪些步骤条件


三.什么情况下YGC前会提前触发FGC


四.FGC的算法是什么


五.YGC过后可能对应哪几种情况


六.YGC后有哪几种情况对象会进入老年代


 


一.什么时候会尝试触发YGC


当新生代的Eden区和其中一个Survivor区空间不足时，就会触发YGC。


 


二.YGC前如何检查老年代大小，涉及哪些步骤条件


步骤1：


先判断新生代中所有对象的大小是否小于老年代的可用区域。如果是则触发YGC，如果否则继续进行下面2中的判断。


 


步骤2：


如果设置了\-XX:HandlePromotionFailure参数，那么进入步骤3。如果没有设置\-XX:HandlePromotionFailure参数，那么就触发FGC。


 


步骤3：


判断YGC历次进入老年代的平均大小是否小于老年代可用区域。如果是则触发YGC，如果否则触发FGC。


 


三.什么情况下YGC前会提前触发FGC


(新生代现有存活对象 \> 老年代剩余内存情况) \+ 未设置空间担保。


 


(新生代现有存活对象 \> 老年代剩余内存情况) \+ (设置了空间担保 \+ 但担保失败)。


 


四.FGC的算法是什么


标记整理算法(但是CMS是标记清理再整理，FGC包含CMS)。老年代对象存活时间较长，复制算法不太适合且老年代区域不再细分。标记清除算法会产生内存碎片，标记整理算法则可以规避碎片。


 


五.YGC过后可能对应哪几种情况


情况1：存活对象所占空间 \< S区域内存大小，那么存活的对象进入Survivor区。


 


情况2：S区域内存大小 \< 存活对象所占空间 \< 老年代可用大小，那么存活的对象直接进入老年代。


 


情况3：(存活对象大小 \> S区大小) \& (存活对象大小 \> 老年代可用大小)，那么会触发FGC，老年代腾出空间后，再进行YGC。如果腾出空间后还不能存放存活对象，则会导致OOM。OOM也就是堆内存空间不足、堆内存溢出。


 


六.哪些情况下YGC后的对象会进入老年代


情况1：S区域内存大小 \< 存活对象所占空间 \< 老年代可用大小。


 


情况2：经过XX:MaxTenuringThreshold次YGC的，默认最大是15次。


 


情况3：对象动态年龄判断机制。年龄1 \+ 年龄2 \+ 年龄n的对象，大小总和超过了Survivor区的50%，此时就会把年龄为n及以上的对象都放入老年代。


 


**7\.问题汇总**


**问题一：**


一个ParNew \+ CMS的GC，如何保证只做YGC，JVM参数如何配置？


 


答：首先上线系统后，要借助一些工具统计每秒在新生代新增多少对象。然后多长时间触发一次YGC，平均每次YGC后会有多少对象存活，YGC后存活的对象在Survivor区是否可以放得下。


 


关键就是要让S区放得下，且不能因动态年龄判定规则直接升入老年代。只要S区可以放下，那么下次YGC后还是存活这么多对象，依然可以在另外一块S区放下，基本就不会有对象升入老年代里了。


 


要做到仅仅YGC而几乎没有FGC是不难的，只要结合系统的运行，根据它的内存占用情况，YGC后的对象存活情况，合理分配Eden、Survivor、老年代的内存大小，合理设置一些参数即可。


 


**问题二：**


为什么老年代不采用复制算法，像新生代那样一个Eden两个Survivor；


 


答：老年代存活对象太多了。如果老年代采用复制算法，每次都挪动可能90%的存活对象。所以采用先把存活对象移动到一起紧凑些，然后回收垃圾对象的方式。


 


**问题三：**


假设YGC之前老年代空间担保成功，但是不幸的是YGC之后老年代放不下而触发了FGC，之后马上又会伴随一次YGC，相当于短时间内进行了两次YGC，这个两次YGC有必要吗？


 


答：其实多一次YGC相对于FGC来说没什么的，因为它的速度很快，ms级别。


 本博客参考[悠兔机场](https://xinnongbo.com)。转载请注明出处！
