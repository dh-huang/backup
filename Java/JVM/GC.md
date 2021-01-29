# GC

GC其实就是把程序不用的内存空间进行回收，使这些内存空间可以被重复使用；

1. 找到内存空间里的垃圾，将其与活对象分开
2. 回收垃圾对象

**如何找到垃圾**

- 可达性分析法

  从根节点集合（`gc root`）出发，根据引用关系进行**广度优先搜索**或者**深度优先搜索**；

  实现简单，但是在GC阶段需要挂起应用（Stop-the-world），所以很多的GC算法其实都是在致力于降低STW时间；

- 引用计数法

  在堆内存中分配对象时，会为对象分配一段额外的空间，这个空间用于维护一个计数器，如果有一个新的引用指向这个对象，则计数器的值加1；如果指向该对象的引用被置空或指向其它对象，则计数器的值减1；

  算法简单，但是实现起来复杂，且无法解决循环引用无法回收的问题；

**执行方式**

- 串行执行：GC时STW，且只有一个GC线程进行垃圾回收
- 并行执行：GC时STW，但是有多个GC线程进行垃圾回收，减少垃圾回收时间
- 并发执行：GC时，只在必要情况下STW，非STW时GC线程和应用线程可以一起工作

**三色标记法**

深度优先搜索无论是显示还是隐式（通常意味着递归调用）都需要用到一个栈，而且递归的话有个隐患就是爆栈，基于可达性分析的GC算法，标记过程几乎都使用的是广度优先搜索，即三色标记的算法思想，只是实现方式有所不同（栈、队列、多色指针）

**基础回收算法**

- 标记 - 复制算法（`Copy`）：不会发生内存碎片化、高吞吐、可实现快速分配、对象是连续分配的（访问的局部性（Locality）好）；需要重新转移(`relocate`目标是一个不同的内存区域)和对移动后的对象引用重定向(`remap`)，新分配对象可以直接通过**指针碰撞**实现；**空间利用率不高问题**

  > 1. 它使用一块连续的地址空间来实现GC堆，并将其划分为2个半分空间（semispace），分别称为from-space与to-space。平时只用其中一个，也就是from-space；
  > 2. 逐个扫描指针，每扫描到一个对象的时候就把它从from-space拷贝到to-space，并在原来的对象里记录下一个转发指针（forwarding pointer），记住该对象被拷贝到哪里了。要知道一个对象有没有被扫描（标记）过，只要看该对象是否有转发指针即可；
  > 3. 每扫描完一个指针就顺便把该指针修正为指向拷贝后的新对象。这样，对象的标记（mark）、整理（compaction）、指针的修正就合起来在一步都做好了；
  > 4. 它不需要显式为扫描队列分配空间，而是复用了to-space的一部分用作隐式队列。用一个scanned指针来区分to-space的对象的颜色：在to-space开头到scanned指针之间的对象是黑色的，在scanned指针到to-space已分配对象的区域的末尾之间的对象是灰色的。如何知道还有没有灰色对象呢？只要scanned追上了to-space已分配对象区域的末尾就好了。
  >    这种做法也叫做“两手指”（two-finger）：“scanned”与“free”。只需要这两个指针就能维护隐式扫描队列。“free”在我的伪代码里就是to_space->top()。
  >
  > |[ 已分配并且已扫描完的对象 ]|[ 已分配但未扫描完的对象 ]|[ 未分配空间 ]|
  > ^                                                   ^                                               ^                        ^
  > bottom                                       scanned                                  top                     end

- 标记 - 压缩算法（`Compact`）：需要重新转移(`relocate`)和对移动后的对象引用重定向(`remap`)，新分配对象可以直接通过**指针碰撞**实现；**暂停时间过长问题**

- 标记 - 清除算法（`Sweep`）：`freeList`、内存碎片；**堆碎片化问题**



**对基础回收算法的优化**

- 分代：基于**弱分代假说**，一般分为两代：`Young`和`Old`，主要目的是减少GC的作用范围，达到减小回收过程中的停顿时间（`YGC`停顿时间与存活对象的数量正相关），同时提升空间的利用率；但是也会带来以下问题：

  - 跨代引用的处理，需要通过`write-barrier`来解决，这样会对赋值这样的操作带来比较大的压力
  - 不适用于大部分对象会存活很久的应用

  分代式GC是一种**部分垃圾收集**的实现方式（还有**Regional collector**，比如G1），进行部分垃圾收集的时候必然需要**考虑非收集部分对收集部分的引用**，需要使用`Remembered Set` + `barrier`的方式解决（CMS使用的是**`points-out`**的方式，即`card table`；G1使用的是**`points-into`**）

  > Remembered Set是在实现部分垃圾收集（partial GC）时用于记录从非收集部分指向收集部分的指针的**集合**的抽象数据结构

- 增量：通过并发的方式，降低STW时间，即通过GC线程和应用线程交替执行的方式，来控制应用程序的最大暂停时间；但是这种方式会带来一个很严重的问题：**对象引用关系的改变，从而出现对对象的的错标和漏标，错标会产生浮动垃圾，漏标则会影响程序的正确性**；

  其中漏标问题是必须要解决的，产生漏标的充分必要条件是：

  1. 应用线程插入了一个从黑色对象到白色对象的新引用
  2. 应用线程删除了从灰色对象到该白色对象的直接或者间接引用

  那么要解决漏标的办法即是打破以上两个条件中的一个即可：

  1. 使用`write-barrier`（`post-write-barrier`），当**新增引用关系**后，通过两种方式修改对象颜色（打破第1个条件）：

     - Steele 写入屏障：将发出引用的对象（无论是什么颜色）标记为灰色

       ```c++
       write_barrier(obj, field, new_obj){
           if(obj.mark == TRUE && new_obj == FALSE){
               obj.mark = FALSE
               push(obj, mark_stack)
           }
          *field = new_obj
       }
       ```

     - Dijkstra 写入屏障：将被引用的新对象标记为灰色，CMS即是使用这种方式解决漏标的问题 (**card table + mod-union table**)（**incremental update write barrier**）

       ```c++
       write_barrier(obj, field, new_obj){
         	// 其实CMS里的实现是无条件的，即不会有这样的判断
           if(new_obj == FALSE){
               new_obj.mark == TRUE
               push(new_obj, mark_stack)
           }
          *field = new_obj
       }
       ```

       Dijkstra 写入屏障相对来说不会有多判断一次的性能开销，但是可能会产生浮动垃圾

  2. 使用`write-barrier`（`pre-write-barrier`），将所有即将被删除的引用关系记录下来，最后再以这些旧的引用为根重新扫描一次（比如G1的STAB，即`Snapshot At The Begining`）（**SATB write barrier**）（打破第2个条件，接触到的白对象都被grey-protected）

  3. 使用`read-barrier`，当引用新的对象时，那就将该对象标记为灰色对象（打破第1个条件）

  >如果把mutator看作一个抽象的对象（里面包含root set），那么mutator也可以用三色抽象来描述：有使用黑色mutator的算法，也有使用灰色mutator的算法。关键在于是否允许mutator在concurrent marking的过程中持有白对象的引用，允许则为灰色mutator，不允许则为黑色mutator。
  >
  >SATB write barrier是一种黑色mutator做法，而incremental update write barrier是一种灰色mutator做法。
  >
  >灰色mutator做法要完成marking就需要重新扫描根集合。这就是为什么使用incremental update的CMS在remark的时候要重新扫描整个根集合。

  实际上因为`read-barrier`的开销相对与`write-barrier`的开销大很多，且引用的读要远多余写，所以很少会使用`read-barrier`；

  其中增量分为两种：

  - 增量更新（`Incremental Update`）：
  - 增量复制（`Incremental Copying`）：

- 并发：



#### 常见GC分析

**ParNew**

与CMS配合，负责`Young gen`的垃圾回收，采用**标记 - 复制算法 + 并行执行**，基于的是分代算法；

`new Object()`时，会调用`CollectedHeap::obj_allocate`进行内存分配（其中会涉及一个`TLAB`的概念），分配内存的过程是一个死循环，直到`return or throw`，如果未能成功分配空间，则会向`VMThread`提交`VM_GenCollectForAllocation`事件，原引用线程阻塞，直到GC完成；

`VMThread`在到达可以启动GC的节点（`SafePoint`）时，便会调用`GenCollectorPolicy::satisfy_failed_allocation`进行处理，其实就是调用对应的`GenCollectedHeap`的`do_collection`方法，GC成功后会再次尝试分配内存，如果还是失败，就得强制清理软引用，强制压实Java堆(compact)，这里其实就是进行一次`Full GC`；

跨代引用处理：**是一种`bit map`，Card Table（Remembered Set的具体实现，粒度为字级别，HotSpot里是12字节，可能包含多个对象）**，通过`write barrier`（汇编指令）在对象引用变更的时后`mark`，也会在进行`_next_gen->promote`时`mark dirty`；在进行垃圾回收时，扫描`card table`对应位置时就`mark clear`，如果最后对象还是在`young gen`时，`card`还原为`dirty`；

使用**隐式队列（双指针）**的方式进行深度优先搜索，搜索的同时就进行对象的`relocate`和`remap`，如果对象还在`young gen`，就`increment age`；对于已经处理过的对象，会对其进行`mark`以及设置其`mark word`为新的指针，如果再次处理到该对象，只需直接根据其`mark word`里的指针更新引用即可；

```c++
oop new_obj = obj->is_forwarded() ? obj->forwardee()
                                        : _g->copy_to_survivor_space(obj);
```



**CMS**

`Concurrent Mark-Sweep`，与`ParNew`配合，负责`Old gen`的垃圾回收，采用**标记 - 清除 + 并行执行**，是分代 + 增量 + 并发的组合；

内存是不连续的，需要一个`FreeList`帮助内存分配；

两种模式：`foregroud`和`backgroud`

`foregroud`其实就是使用的`concurrent mark-compact`算法，但是针对的只是`old gen`

`backgroud`是所谓的`Major GC`，通过后台线程不断的去扫描（间隔时间`CMSWaitDuration`，默认值2000ms），如果没有配置`CMSWaitDuration`就等待`cms_lock event`，判断是否需要进行GC，触发条件：

1. 配置了`GCLockerInvokesConcurrent`参数，且`GC cause`是`_gc_locker`或者`_java_lang_system_gc`
2. `UseCMSInitiatingOccupancyOnly`为false（默认即为false，表示是否都以配置的容量来判断），根据统计数据动态判断是否需要进行GC，这里的统计数据就是**GC工作时间预计会大于填满剩余Old gen的时间**；如果还没有统计数据，那么就判断当前`Old gen`的容量是否达到`CMSBootstrapOccupancy`（默认是50%）；如果两者判断出来都是false，进入下一步
3. 容量达到一定的程度：如果有配置`CMSInitiatingOccupancyFraction`（默认是-1），则以其为准；如果没有配置，那么就是`MinHeapFreeRatio`（默认是40）和`CMSTriggerRatio`（默认是80）结合计算出来值（默认计算出来是92）
4. `Old gen`刚因为分配空间而扩容，或者根据`free list`进行判断
5. `Young GC`已经失败，或者即将失败（`Old gen`没有足够的数据来容纳晋升对象了）
6. `Metaspace`的情况需要进行一次GC

`backgroud`执行阶段：

1. **Initial Mark：**遍历**`gc root和young gen`**中对`old gen`的引用，如果对象处于`old gen`，则进行`mark`(**`bit map`**)，相当于扫描出第一批灰色对象，是需要`stop the world`的

2. **Concurrent Marking：**并发标记，处理`initial mark`中灰色对象，即`bit map`中已经`mark`了的对象，这个阶段是GC线程和用户线程并发执行的，那么肯定会出现引用关系的变化，同样**使用`write barrier`记录引用变化（incremental update write barrier**），具体实现是`card table`（`points-out`，记录的是其出发指向别的范围的指针），同时需要配合`mod-union table`，解决可能出现的`Young GC`改变了`card table`的值

   > 注：实际上HotSpot VM一般用的post-write barrier非常简单，就是无条件的记录下发生过引用关系变化的card；
   >
   > 这里既不关心field所在的分代，也不关心val的值，所以其实只要有引用改变，其对应的card都会被记录。也就是说这个card table记录的不只是old -> young引用，而是所有发生了变化的引用的出发端，无论在old还是young。
   >
   > 但是HotSpot VM只使用了old gen部分的card table，也就是说只关心old -> ?的引用。这是因为一般认为young gen的引用变化率（mutation rate）非常高，其对应的card table部分可能大部分都是dirty的，要把young gen当作root的时候与其扫描card table还不如直接扫描整个young gen。

3. **Concurrent Preclean:** 主要是处理`mod-union table`，减轻`final remark`的负担，根据配置参数决定是否处理`reference`和`survivors`

4. **Concurrent Aborted Preclean:** 同样主要是处理`mod-union table`，根据配置参数决定是否处理`reference`和`survivors`，同时会对`eden gen`进行`samples`，将其进行合理的划分`chunk`，也是减轻`final remark`的负担，同时还可以根据`eden gen`的情况决定何时停止，以防止两次连续的STW（`Young GC`和`final remark`），以及尽可能等待一次`Young GC`

5. **Final Remark:** 重新扫描`gc root`、`young gen`、`card table`、`mod-union table`，一定是要重新扫描根集合的(`gc root`、`young gen`)防止漏标（因为是`incremental update write barrier`的做法，即把应用变化了的对象标记为灰色对象，只是单纯使用`card table`这样的方式来的话，并不能完全保证不漏标，少了`gc root`和`young gen`，即无法记录`Concurrent Mark`期间堆外根集（这里其实还包括`young gen`）的引用变化），其实也是很快的，遇到已经`mark`了的对象直接跳过就可以了

6. **Concurrent Sweeping:** 根据`bit map`中的信息，重置`free list`，这时其实`old gen`是不能分配对象的，肯定是要锁`free list`的

7. **Resizing:**

8. **Resetting:** 整理`free list`，比如将两个相邻的块进行合并

CMS是一种**增量更新(incremental update)**的算法，但是其存在很严重**内存碎片**问题，所以需要一个可以进行`mark compact`的算法（HotSpot VM中实现为串行的LISP2 算法）进行`full gc`；

CMS在大堆内存回收上性能会很差：

1. 因为`old gen`不可避免的会产生内存碎片，总会遇到`old gen`无法分配空间的时候，这时候就需要进行`compact`，而CMS使用的又是串行的`full gc`，那么堆越大，`full gc`的停顿时间就越长；
2. 每次进行`major gc`都需要遍历所有的`live objects`，堆越大，gc时间也就越长，无法做到**软实时**（即让 GC 停顿能大致控制在某个阈值以内）

**promotion failed**，**concurrent mode failure**



**G1**

`garbige first`，独自负责`Young gen`和`Old gen`的垃圾回收，垃圾回收分为：`Young GC` 、`Mixed GC`，采用**标记 - 复制 + 并行执行**，是 Regional + 增量 + 并发的组合；

堆空间被分割成一些相同大小的`region`，每一个都是连续范围的虚拟内存，每个`region`都有着分代思想中的分代角色（eden、survivor、old），**每个`region`都可以单独进行GC，基于`per-region remembered set`可以避免对所有的live objects遍历**；

G1执行一个全局的并发标记阶段来决定堆中的对象的活跃度，首先会收集那些产出大量空闲空间的区域，基于用户指定的暂停时间指标去选择收集区域的数量；

G1的执行过程：**全局并发标记(`Global concurrent marking`)**、**清除(`Evacuation`)**

