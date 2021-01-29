# Deque

##### 不要使用Stack

> https://www.ershicimi.com/p/fc16e33c7288a336d2d4c2edd90ef429

JDK中，`Stack`**继承**于`Vector`，**继承使得Stack继承了Vector所有public方法，包括add(int index, Object element)**，意思就是我们可以在栈中任意位置插入一个元素，这样肯定是不符合栈的数据结构的。

##### ArrayDeque & LinkedList

> https://blog.jrwang.me/2016/java-collections-deque-arraydeque/

`ArrayDeque`和`LinkedList`都实现了`Deque`相应的接口，`ArrayDeque`底层实现基于**循环数组**，`LinkedList`底层实现基于**链表**，推荐使用`ArrayDeque`（官方哟），因为其采用的是**循环数组**，在不涉及到扩容的情况下，其插入删除时间复杂度是`O(1)`，扩容的话就是`O(n)`，虽然链表也是`O(1)`，但是其每次添加元素，都需要重新创建一个`Node`对象，在数据量特别大的情况下，`LinkedList`的性能比`ArrayDeque`慢很多；

```java
public class ArrayDeque<E> extends AbstractCollection<E>
                           implements Deque<E>, Cloneable, Serializable
{
  // 存放数据的数组
  transient Object[] elements; // non-private to simplify nested class access
  // 指向第一个元素的数组下标，是循环的
  transient int head;
  // 指向最后一个元素的下一个数组下标，即新入队元素应该放在的位置
  // 为什么head是第一个元素，而tail是最后一个元素的下一位置？先放元素再扩容？
  transient int tail;
  // 最小初始容量
  private static final int MIN_INITIAL_CAPACITY = 8;
  
  // 指定容量的构造函数，会通过allocateElements方法将容量设置为最优的(2^n)
  public ArrayDeque(int numElements) {
    allocateElements(numElements);
  }
  
  public ArrayDeque(Collection<? extends E> c) {
    allocateElements(c.size());
    addAll(c);
  }
  
  // 修改容量为2^n，方便位运算，提升元素操作的效率，但是会浪费内存。。。
  private void allocateElements(int numElements) {
    int initialCapacity = MIN_INITIAL_CAPACITY;
    // Find the best power of two to hold elements.
    // Tests "<=" because arrays aren't kept full.
    if (numElements >= initialCapacity) {
      initialCapacity = numElements;
      // 经历5次无符号右移，再或操作，再++，一定能使其变为2^n
      // 比如：numElements = 01?? ???? ???? ???? ???? ???? ???? ????
      // 得到011? ???? ???? ???? ???? ???? ???? ????
      initialCapacity |= (initialCapacity >>>  1);
      // 得到0111 1??? ???? ???? ???? ???? ???? ????
      initialCapacity |= (initialCapacity >>>  2);
      // 得到0111 1111 1??? ???? ???? ???? ???? ????
      initialCapacity |= (initialCapacity >>>  4);
      // 得到0111 1111 1111 1111 1??? ???? ???? ????
      initialCapacity |= (initialCapacity >>>  8);
      // 得到0111 1111 1111 1111 1111 1111 1111 1111
      initialCapacity |= (initialCapacity >>> 16);
      initialCapacity++;

      if (initialCapacity < 0)   // Too many elements, must back off
        // 最大容量 2^30
        initialCapacity >>>= 1;// Good luck allocating 2 ^ 30 elements
    }
    elements = new Object[initialCapacity];
  }
  
  // 添加到头部
  // Stack.push
  public void addFirst(E e) {
    if (e == null)
      throw new NullPointerException();
    // head = (head - 1) & (elements.length - 1)，获取下一头节点位置，保证循环
    // 因为elements.length保证是2^n，那么elements.length - 1就保证了低位(n - 1)全为1
    // 如果head - 1 >= 0，那么(head - 1) & (elements.length - 1)的结果就是head - 1
    // 如果head - 1 < 0，即head = 0，-1的二进制表示是(32个1，补码：https://www.cnblogs.com/zhxmdefj/p/10902322.html)
    // 那么(head - 1) & (elements.length - 1)的结果就是(elements.length - 1)，即数组结尾
    // 这样仅仅通过位与操作就可以完成环形索引的计算，而不需要进行边界的判断，在实现上更为高效
    elements[head = (head - 1) & (elements.length - 1)] = e;
    // head == tail时，说明满了
    if (head == tail)
      doubleCapacity();
  }
  
  // 添加到尾部
  // Queue.add
  public void addLast(E e) {
    if (e == null)
      throw new NullPointerException();
    // 先填上数据，tail指向的最后一个元素的下一位置
    elements[tail] = e;
    // 变更tail索引，原来同addFirst中
    // 如果tail + 1 <= (elements.length - 1)，那么其结果就是elements.length - 1
    // 如果tail + 1 == elements.length，那么其结果就是0
    if ( (tail = (tail + 1) & (elements.length - 1)) == head)
      doubleCapacity();
  }
  
  // 得到头部数据，并删除
  // Stack.pop, Queue.poll
  public E pollFirst() {
    int h = head;
    @SuppressWarnings("unchecked")
    E result = (E) elements[h];
    // Element is null if deque empty
    if (result == null)
      return null;
    elements[h] = null;     // Must null out slot
    // head索引向后增加一位
    head = (h + 1) & (elements.length - 1);
    return result;
  }

  // 得到尾部数据，并删除
  public E pollLast() {
    // tail索引向前增加一位
    int t = (tail - 1) & (elements.length - 1);
    @SuppressWarnings("unchecked")
    E result = (E) elements[t];
    if (result == null)
      return null;
    elements[t] = null;
    tail = t;
    return result;
  }
}
```

