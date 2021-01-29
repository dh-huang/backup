# HashMap

扰动函数，初始化容量，负载因子，扩容方法，hash冲突解决（链表，红黑树）

##### 扰动函数

`hashCode()`取值范围：`-2^31 ~ s^31 - 1`，不可能直接使用其作为数组的长度，所以要获取数组的下标，需要与真实的数组长度进行**取模**运算，而为了能直接使用`&`运算进行取模，要求数组容量是`2^n`（`length - 1`是类似`01111`这样的值，即`2^n - 1`），那么实际上有效的取值范围可能就只有低几位，`h >>> 16`取`hashCode`的高位，再异或(^)上自己的低位，混合原始`hashCode`的高低位，增大**随机性**，使数据分配更均匀；

```java
/**
 * 扰动函数，定义数组下标：(n - 1) & hash
 */
static final int hash(Object key) {
  int h;
  return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

##### 初始化容量

```java
/**
 * Returns a power of two size for the given target capacity.
 */
static final int tableSizeFor(int cap) {
  int n = cap - 1;
  // 不断的用1对低位进行占位，得到2^n - 1
  n |= n >>> 1;
  n |= n >>> 2;
  n |= n >>> 4;
  n |= n >>> 8;
  n |= n >>> 16;
  // 再加上1，得到2^n
  return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}

/**
 * Constructs an empty <tt>HashMap</tt> with the specified initial
 * capacity and load factor.
 *
 * @param  initialCapacity the initial capacity
 * @param  loadFactor      the load factor
 * @throws IllegalArgumentException if the initial capacity is negative
 *         or the load factor is nonpositive
 */
 public HashMap(int initialCapacity, float loadFactor) {
   if (initialCapacity < 0)
   	throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
   if (initialCapacity > MAXIMUM_CAPACITY)
   	initialCapacity = MAXIMUM_CAPACITY;
   if (loadFactor <= 0 || Float.isNaN(loadFactor))
   	throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
   this.loadFactor = loadFactor;
   // 使数组长度为2^n
   this.threshold = tableSizeFor(initialCapacity);
 }
```

