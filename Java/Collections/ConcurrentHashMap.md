# ConcurrentHashMap



**扩容戳**

`Integer.numberOfLeadingZeros`返回无符号整数最高位非0位前0的个数，这个n其实就是当前`table`的长度，这样保证每次新的扩容生成的扩容戳都是不同的，`| (1 << (RESIZE_STAMP_BITS - 1))`其实就是保证其**第15位为1**，设置`sizeCtl`的时候是通过`U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2)`来设置的，其中`rs << RESIZE_STAMP_SHIFT`左移16位，即刚好保证**第31位**为1，`sizeCtl`表示一个负数，

```java
/**
 * Returns the stamp bits for resizing a table of size n.
 * Must be negative when shifted left by RESIZE_STAMP_SHIFT.
 */
static final int resizeStamp(int n) {
  return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```

