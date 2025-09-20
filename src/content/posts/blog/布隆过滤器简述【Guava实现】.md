---
 title: 布隆过滤器简述【Guava实现】
 published: 2025-03-05
 tags:
  - BlogPost
 lang: zh
 abbrlink:  bc6a9c8c-fb95 
---

1. 简述：
> 布隆过滤器是一种利用对象表示，通过插入自定义缓存判断对象是否存在、不存在的技术；

2. 举例，已Guava工具包中的布隆为例
```xml
       <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
        </dependency>
```
> 在Guava工具包中实现的布隆过滤器，利用`hash`处理对象得出对应的`hashcode`，将`hashcode`存入一个
`long`当中；`Guava`工具包中的布隆过滤器没有扩容，推荐创建时选择大于当前对象集2倍以上容量；

![](https://img2024.cnblogs.com/blog/3426265/202503/3426265-20250305163235202-1219754169.png)
> 当新对象通过`hash`算出新的值后，与当前的`Long`进行位与操作，如果位置已经是`1`经过与操作不变，`0`转变为`1`
如果存在`0`转变为`1`，表明这个对象是一个新的对象，没有被缓存过（即之前不存在的对象）
3. 缺点
> 1. 经过上述简介，我们知道`guava`实现的布隆过滤器存放在一个`long`下，现在假使这个缓存不会扩容的情况下，当存入的对象越来越多后，`long`的所有位都会变为`1`,此时过滤器完全失去作用，会判断所有对象存在;
>2. 假使两个对象存入`long`后刚好占据后4位，将后4位转变为`1`，这时一个新对象经过`hash`后需要转变`long`的后4位为`1`，这时同样会判为已存在，即 `1+3 = 4`，所以布隆过滤器判断已存在的对象不可信,即`不存在的一定不存在，存在的不一定存在`；
> 3. 扩容问题，当过滤器中位为`1`的位越来越多后，布隆过滤器得出的结果就越会失真，使用扩容增加`0`位可以提高准确率，但是使用扩容方案，扩容前的对象由于存在多个对象混合插入的情况（即同一个`1` 会代表多个对象），扩容后的缓存如何恢复扩容前缓存的记录。可以采取废弃旧布隆直接创建新布隆处理（需要缓存的对象是被持久化的），提高可用性的话可以考虑多个过滤器混合构成，当前一个不够用直接创建一个新的，拿前一个+后一个得出的结果作为最终结果
> 4. 删除问题，类似扩容即同一个`1`会代表多个对象,经过`hash`后无法准确的只删除当前对象的`1`，可以使`1`增加计数器处理，当删除操作发生时，如果计数器存在值，优先减去计数器的值；

4. 源码阅读
```java
    MURMUR128_MITZ_32 {
      //将对象放入容器
        public <T> boolean put(T object, Funnel<? super T> funnel, int numHashFunctions, LockFreeBitArray bits) {
            //过滤器相应的容器缓存
            long bitSize = bits.bitSize();
            //取对象相应的hash值
            long hash64 = Hashing.murmur3_128().hashObject(object, funnel).asLong();
            int hash1 = (int)hash64;
            int hash2 = (int)(hash64 >>> 32);
            boolean bitsChanged = false;
          // numHashFunctions 目的是为了减少hash冲突
            for(int i = 1; i <= numHashFunctions; ++i) {
                int combinedHash = hash1 + i * hash2;
                if (combinedHash < 0) {
                    combinedHash = ~combinedHash;
                }
              //判断是否有位变化，取余操作是为了避免越界
                bitsChanged |= bits.set((long)combinedHash % bitSize);
            }

            return bitsChanged;
        }

        public <T> boolean mightContain(T object, Funnel<? super T> funnel, int numHashFunctions, LockFreeBitArray bits) {
            long bitSize = bits.bitSize();
            long hash64 = Hashing.murmur3_128().hashObject(object, funnel).asLong();
            int hash1 = (int)hash64;
            int hash2 = (int)(hash64 >>> 32);

            for(int i = 1; i <= numHashFunctions; ++i) {
                int combinedHash = hash1 + i * hash2;
                if (combinedHash < 0) {
                    combinedHash = ~combinedHash;
                }

                if (!bits.get((long)combinedHash % bitSize)) {
                    return false;
                }
            }

            return true;
        }
    }

static final class LockFreeBitArray {
        private static final int LONG_ADDRESSABLE_BITS = 6;
        final AtomicLongArray data;
        private final LongAddable bitCount;

        LockFreeBitArray(long bits) {
          // 确保类型转换安全，保证初始化的容器位全为初始值，避免精度丢失导致创建的容器存在值
          /**
              public static int checkedCast(long value) {
                    int result = (int)value;
                    Preconditions.checkArgument((long)result == value, "Out of range: %s", value);
                    return result;
              }
          */
            this(new long[Ints.checkedCast(LongMath.divide(bits, 64L, RoundingMode.CEILING))]);
        }

       boolean set(long bitIndex) {
            if (this.get(bitIndex)) {
                return false;
            } else {
              // >>> 无符号移位操作，保证结果为正数 
              /**
                const a = 5; //  00000000000000000000000000000101
                const b = 2; //  00000000000000000000000000000010
                const c = -5; //  11111111111111111111111111111011

                console.log(a >>> b); //  00000000000000000000000000000001
                // Expected output: 1

                console.log(c >>> b); //  00111111111111111111111111111110
                // Expected output: 1073741822
              */
              // LONG_ADDRESSABLE_BITS = 6
                int longIndex = (int)(bitIndex >>> LONG_ADDRESSABLE_BITS );
              //顶掉了符号位，和 >>> 无符号操作保持一致
                long mask = 1L << (int)bitIndex;  // only cares about low 6 bits of bitIndex

                long oldValue;
                long newValue;
                do {
                    oldValue = this.data.get(longIndex);
                    newValue = oldValue | mask;
                    if (oldValue == newValue) {
                        return false;
                    }
                } while(!this.data.compareAndSet(longIndex, oldValue, newValue));

                this.bitCount.increment();
                return true;
            }
        }
    
     boolean get(long bitIndex) {
            return (this.data.get((int)(bitIndex >>> 6)) & 1L << (int)bitIndex) != 0L;
     }

}
```
